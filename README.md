# ldns-win

Unofficial Windows builds of [ldns](https://github.com/NLnetLabs/ldns) (the library,
`drill`, and the example utilities), cross-compiled with MSYS2 for x64 and arm64 and
published as GitHub releases here.

This repo started as the `.github/patches` and `.github/workflows/build-ldns.yml` of
[shanto/Scoop](https://github.com/shanto/Scoop); the bucket now consumes the releases
published here and holds no build recipe of its own.

## Releasing

Nothing to bump by hand: a daily run (03:00 UTC) reads the newest stable tag from
NLnetLabs/ldns and skips when that version is already released here. A manual
dispatch takes an optional version argument.

1. The workflow builds both architectures from that tag.
2. It publishes `ldns-<version>` with `ldns-windows-<arch>.zip` + `.sha256`, and
   reads the assets back by size.
3. If `PAT_SCOOP_RW` is set it also bumps `bucket/ldns.json`; otherwise the
   bucket's own nightly autoupdate picks the release up through its `checkver`.

A published version is never rebuilt — see `IDEA.md`.

## Local check

```
task verify-patches     # clone ldns at VERSION and apply patches/ cleanly
```

Requires `git`, `gh` and `task`.
