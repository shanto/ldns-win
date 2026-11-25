# ldns-win — why this exists

## Purpose

Build ldns for Windows (x64 + arm64) with MSYS2 and publish the zips as releases here,
so the Scoop bucket only consumes artifacts instead of carrying a cross-compile recipe
for one package. Two patches are required; neither is upstreamable as-is, see below.

## Layout

- `VERSION` — the ldns tag to build. Single source of truth; the workflow reads it.
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

- **Version comes from `VERSION`, not from a consumer.** In the bucket the workflow
  grepped `bucket/ldns.json`; that tied the build to a checkout of another repo. The
  manifest is now downstream of the release, not upstream of it.
- **The apply step normalizes a CRLF ldns checkout** instead of the build trying
  to configure git. The patches are LF; a Windows runner that checks ldns out as
  CRLF makes every hunk miss, so the step strips CR from `configure.ac` and
  `drill/drill.c` before applying. (A separate `git config --global core.autocrlf`
  step failed on the MSYS2 shell — git is not guaranteed there — and was removed.)
- **Patches are LF and this repo is LF** (`.gitattributes: eol=lf`), unlike the bucket
  repos (CRLF). Inline `run:` shell scripts and patch files must not carry CR.
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
- **The bucket bump is optional and gated.** It needs `PAT_SCOOP_RW` (a cross-repo
  write; `GITHUB_TOKEN` cannot), and it only touches a manifest whose download URLs
  already point at this repo's releases — otherwise it logs and leaves it alone.

## Bucket wiring (shanto/Scoop)

`bucket/ldns.json` must point at this repo:

```
https://github.com/shanto/ldns-win/releases/download/ldns-$version/ldns-windows-<arch>.zip
hash: $url.sha256
```

`checkver` still reads `https://nlnetlabs.nl/projects/ldns/about/` for the version;
this repo publishes no pointer file, so that page stays the upstream version signal.

## Not done yet

- Repo not created on GitHub and nothing pushed (local commit only).
- `bucket/ldns.json` still downloads from `shanto/Scoop`'s `ldns-1.9.2` release. Repoint
  it after the first release here, not before — the assets do not exist here yet.
- The old `.github/patches/` and `.github/workflows/build-ldns.yml` are still in
  shanto/Scoop, so a bucket push can still trigger a build there; delete them when the
  manifest is repointed.
- No scheduled upstream check: a new ldns release needs a `VERSION` bump.
