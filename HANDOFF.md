# Handoff

Last updated 2026-08-25.

## State

Scaffolded, not yet deployed. Nothing has been pushed to R2; no Cloudflare resources exist.

Done and verified:
- Sizes measured against `rsync.dante.ctan.org`: full CTAN 68.98 GB / 351,866 files; tlnet
  6.78 GB / 16,971 real files (17,393 after symlink dereference). Free tier fits tlnet only.
- `Taskfile.yml` renders (`task --dry sync`), `task size` returns 17,393 files / 6.78 GB,
  and `task verify` passed against a real `tlpkg/` from `mirrors.mit.edu` (sha512 OK,
  `VALIDSIG` with primary key `C78B82D8C79512F79CC0D7C80D5E5D9106BAB6BC`).
- `TLPDB.pm` in the current install-tl confirms tlmgr fetches unversioned container names over
  HTTP, which is why symlinks are dereferenced and versioned originals dropped.

Not verified:
- An actual `rclone` push to R2. Flags follow Cloudflare's rclone guide but have not run.
- The workflow end to end on a runner (rsync throughput from MIT, total job time).
- `tlmgr` installing from the bucket through a custom domain.

## Decisions

- tlnet only, all of it. Full CTAN cannot be free (~$0.89/month on R2); a subset gains
  nothing over the full 6.8 GB.
- Stateless daily job: re-download 6.8 GB each run rather than cache. Upgrade path if it
  ever matters: `rclone mount` the bucket and rsync into it, or a phantom tree built from
  `rclone lsjson` so rsync transfers only the delta.
- Source mirror is `mirrors.mit.edu` (rsync, near the Azure US runners). We are not an
  official CTAN mirror, so the "sync from dante hourly" rule does not apply.
- No directory index pages; tlmgr does not need them.
- No edge caching to start. `.xz` is uncached by default, Class B budget is ample.

## Next Steps

1. Cloudflare: enable R2, create bucket `tlnet`, scoped API token, custom domain.
   Steps are in `README.md`.
2. Add repository secrets `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`.
3. `workflow_dispatch` the sync; watch the first run (expect ~6.8 GB upload, ~17k Class A).
4. `tlmgr option repository https://tlnet.<zone>/` and `tlmgr update --self --all` from a
   machine to prove it end to end.
5. Decide on the 60-day cron rule: keep committing, add a keepalive action, or go private.

## Open Questions

- Which zone hosts the custom domain.
- Whether to keep docs (3.7 GB of the 6.8 GB). Dropping them buys headroom but the free tier
  already fits; revisit only if `task size` approaches 10 GB.
- Cache-everything rule plus purge on publish, once traffic justifies it.
