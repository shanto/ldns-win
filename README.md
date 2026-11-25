# ldns-win

Unofficial Windows builds of [ldns](https://github.com/NLnetLabs/ldns) (the library,
`drill`, and the example utilities), cross-compiled with MSYS2 for x64 and arm64 and
published as GitHub releases here.

This repo started as the `.github/patches` and `.github/workflows/build-ldns.yml` of
[shanto/Scoop](https://github.com/shanto/Scoop); the bucket now consumes the releases
published here and holds no build recipe of its own.

## Releasing

1. Bump `VERSION` (the ldns tag to build, e.g. `1.9.2`), commit, push to `master`.
2. The workflow builds both architectures, publishes `ldns-<version>` with
   `ldns-windows-<arch>.zip` + `.sha256`, and reads the assets back.
3. If `PAT_SCOOP_RW` is set, it also bumps `bucket/ldns.json` in the bucket repo.

A manual run takes an optional version argument; with none it builds `VERSION`.
Publishing the same version twice is refused — see `IDEA.md`.

## Local check

```
task verify-patches     # clone ldns at VERSION and apply patches/ cleanly
```

Requires `git`, `gh` and `task`.
