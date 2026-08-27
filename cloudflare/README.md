# cloudflare

Optional. The mirror uploads to R2 and serves from R2 whether or not anything here ever
runs; nothing in the pipeline depends on the zone. This directory holds the four rulesets
the mirror wants on its zone, and the tasks that read and write them.

Leave `CF_API_TOKEN` or `CF_ZONE_ID` unset and every task here is a no-op. Delete the
directory and `task sync` still runs.

## The rulesets

| File | Phase | What it does |
|---|---|---|
| `bypass-rules.json` | `http_request_cache_settings` | `CACHE=off`: every request reads the bucket |
| `cache-rules.json` | `http_request_cache_settings` | `CACHE=on`: one day at the edge, 3xx and up never stored |
| `transform-rules.json` | `http_request_transform` | `/` serves CTAN's own `index.html` |
| `redirect-rules.json` | `http_request_dynamic_redirect` | R2 has no directory listings, so `/foo/` goes to ctan.org |
| `config-rules.json` | `http_config_settings` | Email Obfuscation and Rocket Loader off, so HTML is served byte for byte |

`config-rules.json` is not cosmetic. Email Address Obfuscation rewrites every `text/html`
response Cloudflare serves: it injects a script, encodes mailto addresses, and drops
`content-length`. A 4,006 byte file arrives as 4,216. About 7,300 files in CTAN are HTML,
and `smoke` compares what it reads back against the upstream listing, so a run that samples
one of them fails.

Each file carries `@HOST@` instead of a hostname and `"sha256:STAMP"` instead of a hash.
`set` fills in `HOST` from `Taskfile.yml`, hashes the result, and sends that hash as the
ruleset's description. That description is how a later run recognises its own work.

## The token

Cloudflare dashboard, My Profile -> API Tokens -> Create Token -> Create Custom Token.

| Permission | Access | Phase it covers |
|---|---|---|
| Cache Rules | Edit | `http_request_cache_settings` |
| Transform Rules | Edit | `http_request_transform` |
| Single Redirect | Edit | `http_request_dynamic_redirect` |
| Config Rules | Edit | `http_config_settings` |
| Cache Purge | Purge | only with `CACHE=on`; skip it otherwise |

Zone Resources: Include -> Specific zone -> your zone. No account resources. Leave client
IP filtering empty, because GitHub's runners do not have stable addresses. `Edit` covers
reading, so the same token drives `get`.

`CF_ZONE_ID` is on the zone's Overview page, under API. It is not secret; the pipeline
passes it like one anyway.

Cloudflare's own pages disagree about whether these need account-level permissions as well:
the Cache Rules page also asks for _Account_ > _Rulesets_ > _Edit_ and _Account_ > _Filter
Lists_ > _Edit_, while Single Redirect and Configuration Rules name only the zone
permission. Those account entries are for account-scoped rulesets and lists, which this
repo never touches. Start with the five rows above; a 403 names what it wanted.

## By hand

```sh
export CF_API_TOKEN=... CF_ZONE_ID=...

task cloudflare:get                              # what the zone holds now, and who wrote it
task cloudflare:get PHASE=http_config_settings   # one phase
task cloudflare:set                              # put the rulesets on the zone
task cloudflare:set PHASE=http_config_settings
```

`get` reads and prints; it never writes. `set` writes a phase only when that phase is empty
or already carries one of our stamps:

```
http_request_cache_settings: empty; bypass-rules.json goes on as is
http_request_transform: 1 rule(s) NOT written by this repo; transform-rules.json is held back until they are gone
http_config_settings: 1 rule(s) already from config-rules.json
```

A PUT replaces every rule in a phase, so a phase holding rules this repo did not write is
left alone: `set` warns, skips that file, and carries on. To take such a phase over, fold
the rules worth keeping into that file — a ruleset's `rules` array takes as many entries as
the plan allows — or clear the phase in the dashboard, then run `set` again.

## In the automation

Set the repository secret `CF_ENABLE_AUTOMATION` to anything. From then on `sync` runs
`cloudflare:set` at the start of every hour, which converges the zone and leaves it alone
when it already agrees. Unset, `rules` reports itself up to date and no request is made.

The token and the zone id must be set too; without them the tasks here skip regardless.

A run never deletes a ruleset, never touches a phase outside the five files above, and never
fails the mirror over the zone: the only outcomes are applied, already current, or a warning
that a phase was skipped.

## Caching

`CACHE=on` swaps `cache-rules.json` in for `bypass-rules.json` and makes the pipeline purge
every changed key after each batch. It is off because it buys nothing yet: R2 gives 10M
Class B reads a month free and the mirror uses about 6,500, so roughly 27 full TeX Live
installs a day fit inside the free tier. Turn it on when Class B passes 5M a month twice in
a row, and add Cache Purge to the token first.

## Checking a change offline

`CF` points at the API; override it with a directory of canned replies and no request
leaves the machine.

```sh
d=$(mktemp -d)
for p in http_request_cache_settings http_request_transform \
         http_request_dynamic_redirect http_config_settings; do
  mkdir -p "$d/rulesets/phases/$p"
  echo '{"result":{"description":"mine","rules":[{"expression":"x"}]}}' \
    > "$d/rulesets/phases/$p/entrypoint"
done
CF_API_TOKEN=x CF_ZONE_ID=x task cloudflare:set CF="file://$d"
```

That tree stands for a zone whose phases someone else wrote, so the run must print four
warnings, apply nothing, and exit 0. Put our own stamp in a `description` instead and the
same command must report `already` and make no call. `task cloudflare:lint` checks every
file parses, holds a rule, carries its `sha256:STAMP` line, and names no host of its own.
