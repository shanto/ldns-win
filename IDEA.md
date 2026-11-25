# ldns-win — why this exists

## Purpose

Build ldns for Windows (x64 + arm64) with MSYS2 and publish the zips as releases here,
so the Scoop bucket only consumes artifacts instead of carrying a cross-compile recipe
for one package. Two patches are required; neither is upstreamable as-is, see below.

## Layout

- `patches/` — applied in filename order on top of the upstream tag.
  - `ldns-windows-winsock.patch` — configure.ac: detect `windows.h`, define
    `USE_WINSOCK`, link `iphlpapi`.
  - `ldns-windows-resolver.patch` — drill: build a resolver from the Windows adapter
    DNS servers (`GetAdaptersAddresses`) and resolve the trust-anchor path relative to
    the running exe. Windows has no `/etc/resolv.conf`, so drill without a nameserver
    argument otherwise fails.
- `.github/workflows/build.yml` — resolve → build matrix → publish.
- `Taskfile.yml` — local helpers (`verify-patches`, run watching).

## Decisions

- **The version comes from upstream, not from a file or a consumer.** NLnetLabs
  publishes tags but no GitHub *releases*, so `/releases/latest` 404s and the
  workflow lists tags and keeps the newest stable `X.Y.Z`. The bucket used to be
  the version source; that made a bucket bump race the artifacts, so the bucket's
  `checkver` now reads this repo's releases instead.
- **LF end to end, pinned at the checkout rather than repaired afterwards.** This
  repo and the patches are LF (`.gitattributes: * text=auto eol=lf`), and the Windows
  checkout of NLnetLabs/ldns is forced to LF with `GIT_CONFIG_COUNT` /
  `GIT_CONFIG_KEY_0=core.autocrlf=false` / `core.eol=lf` on the checkout step. Git
  reads those from the environment, so no step writes a config file and no build step
  rewrites source files; `git apply` just sees an LF tree. (An earlier
  `git config --global` step failed because the MSYS2 shell has no `git` on PATH, and
  a later `sed` that stripped CR from two files was dead code — both are gone.)
  The bucket repos stay CRLF, following Scoop's own convention.
- **`mkdir -p .libs` before `make` is load-bearing.** The Makefile's
  `setup-builddir` rule runs `ln -s .libs lib`; MSYS2's `ln` emulation copies the
  target rather than symlinking, so a missing `.libs` kills the build with
  `ln: failed to create symbolic link 'lib'`. Do not remove it.
- **The dead `install-sh` copy is gone.** The old workflow had
  `test ! -f install-sh -a -f ../../install-sh` with `../../install-sh` outside the
  workspace, so it never fired. `autoreconf -fi` installs `install-sh` itself.
- **Release is refused if the tag already exists.** The bucket's autoupdate URL is
  keyed on the upstream version (`ldns-$version`), so one tag per version is the only
  shape that works; a rebuild would have to overwrite published assets. Fixing a patch
  means a new upstream version, or deleting the release by hand first.
- **`BUILD-NOTICE.txt` goes inside the archive.** Upstream publishes no Windows
  binaries and does not support these; the archive says so, with version, commit, run
  URL and the patch list.
- **This workflow never writes outside this repo.** Publishing ends at the release;
  there is no cross-repo token, no bucket manifest edit and no post-upload read-back
  step. The bucket's own nightly autoupdate is the only thing that moves
  `bucket/ldns.json`, which keeps the direction of trust one-way: releases here,
  polling from there.

## Bucket wiring (shanto/Scoop)

`bucket/ldns.json` points at this repo:

```
https://github.com/shanto/ldns-win/releases/download/ldns-$version/ldns-windows-<arch>.zip
hash: $url.sha256
checkver: github -> shanto/ldns-win releases/latest, version derived from the tag
```

Both directions now read what is already published: the bucket cannot bump to a
version whose artifacts do not exist, and this repo cannot publish a version the
bucket will never see.

## Not done yet

- No release line for a patch-only fix: a fix to `patches/` cannot ship under an
  existing tag (the bucket's autoupdate URL is keyed on the version). It waits for
  the next upstream tag, or a manual delete-and-rebuild of the release.
- The bucket picks a new release up on its next nightly autoupdate (04:23 UTC) or a
  manual `Autoupdate` dispatch — at worst a day behind, never a version ahead.
- Nothing in CI reads the published assets back; the release is verified by hand
  (`gh release view`, download + hash) when it matters.
- `task verify-patches` is untested here (no `task` in the agent container).
