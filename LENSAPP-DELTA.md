# lensapp/libkrunfw delta

This is a fork of [containers/libkrunfw](https://github.com/containers/libkrunfw).
It exists to enable a small set of kernel `CONFIG_*` flags that upstream's
default config leaves out, in particular netfilter and a handful of its match
extensions. The fork is intentionally tiny so rebasing on upstream stays cheap.

## What's changed vs containers/libkrunfw

Two files only:

- `config-libkrunfw_aarch64` — netfilter block added, marked `# --- BEGIN lensapp delta ---`
- `config-libkrunfw_x86_64` — same block

That's it. No C-level patches, no kernel-source diffs, no `patches/` additions, no Makefile changes.
The kernel version tracks upstream verbatim.

## Why we did NOT pick up the other forks' work

We reviewed [`superradcompany/libkrunfw`](https://github.com/superradcompany/libkrunfw)
and [`smol-machines/libkrunfw`](https://github.com/smol-machines/libkrunfw) before forking.

| Their additions | Adopted? | Why / why not |
|---|---|---|
| superrad: full netfilter block (~70 lines) | partially | We took the subset we exercise; skipped extras (bridge/VLAN/IPSET/MASQUERADE) we don't need today — every line we add is a line we keep merging on every rebase. |
| superrad: extra netfilter match extensions superrad omits | added | Required for our build. |
| superrad: EROFS xattr + partition features | no | Not used yet. |
| smol: disable sound/SCSI/XFS/BTRFS/DRM | no | Size optimization we don't need; trimming costs rebase pain when upstream edits those areas. |
| smol: TSI socket fixes (custom patches) | no | We haven't hit those issues. |
| smol: vsock patch reverts | no | Same — only if we observe the underlying issue. |

Principle: **add the smallest possible config delta, no C patches, no subsystem trimming**.

## Sync recipe (minimal-effort upstream tracking)

When `containers/libkrunfw` cuts a new release (~every 1-3 months per their
[release page](https://github.com/containers/libkrunfw/releases)):

```bash
git fetch upstream
git rebase upstream/main          # rebase our delta commit onto new base
make                              # smoke-test the build
git push --force-with-lease       # update our main
```

Expected merge conflicts: usually zero. The `# --- BEGIN/END lensapp delta ---`
markers sit in a stable section of the config file (between
`CONFIG_NETWORK_PHY_TIMESTAMPING` and `CONFIG_BPFILTER`), and upstream rarely
edits those neighboring lines. If a conflict does happen, the markers make it
obvious which lines are ours.

If a kernel-source change makes a `CONFIG_*` we depend on go away or rename:
run `make oldconfig` after the rebase, address the prompt for any deprecated
symbols, commit the result.

## Bumping the delta

When additional kernel features are needed:

1. Add the necessary `CONFIG_*` lines inside the `# --- BEGIN/END lensapp delta ---` block in **both** arch configs.
2. Tag a new release. CI builds the .so / .dylib artifacts and uploads them to the GitHub release.

## Out of scope

We deliberately do not maintain:

- A CI-side mirror of upstream's release process. Our CI builds only when *we* tag a release.
- Backports of upstream patches. Those flow naturally on the next rebase.
- Any C-level kernel patches.
- Kernel size optimization (trimming subsystems).

If any of those become necessary, document the addition here in the same minimal-delta spirit.
