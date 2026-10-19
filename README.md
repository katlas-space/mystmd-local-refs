# mystmd-local-refs — patched mystmd with `file://` reference URL support

A small patch set against upstream [jupyter-book/mystmd](https://github.com/jupyter-book/mystmd) that teaches `project.references` to accept `file://` URLs. Each release attaches a prebuilt `myst.cjs` artifact so consumers don't have to build from source.

## Why this exists

mystmd's cross-project reference loader (`loadReferences.ts`) only speaks HTTP(S). For multi-project workspaces (a kalaxy of ~N MyST projects referencing each other), that means every local build **before** the site is deployed logs one "Unable to load reference" warning per sibling per world. On a 42-world kalaxy this is ~2000 warnings per build.

The patch adds a `file://` branch to the fetcher: when a `references:` URL begins with `file://`, the loader reads `myst.xref.json` from local disk instead of fetching it. Validator needs no change — mystmd's URL validator (`simple-validators.validateUrl`) uses `new URL()` which already accepts `file://`.

See [`patches/01-file-url-support.patch`](patches/01-file-url-support.patch) for the actual diff (~30 lines against `packages/myst-cli/src/process/loadReferences.ts`).

Same gap exists in Sphinx — tracked at [sphinx-doc/sphinx#12055](https://github.com/sphinx-doc/sphinx/issues/12055). This patch is the mystmd equivalent and is intended for eventual upstream contribution.

## Install (from a release)

Each release attaches a prebuilt `myst.cjs`. Download and run it with node:

```bash
# Latest release
curl -L "https://github.com/katlas-space/mystmd-local-refs/releases/latest/download/myst.cjs" -o myst-patched.cjs

# Or pin to a specific upstream version
curl -L "https://github.com/katlas-space/mystmd-local-refs/releases/download/mystmd-1.11.0+1/myst.cjs" -o myst-patched.cjs

node myst-patched.cjs --version
```

Wrap it in a shell script so your build tooling can call it like `myst`:

```bash
#!/usr/bin/env bash
# Save as `myst-local`
exec node /path/to/myst-patched.cjs "$@"
```

## Use it (your project's references:)

Point sibling-world references at a local directory via `file://`:

```yaml
# myst.yml for world "a" referencing sibling "b"
project:
  references:
    b: file:///abs/path/to/_deploy/worlds-site/b
```

The patched binary reads `_deploy/worlds-site/b/myst.xref.json` from disk — zero network traffic, works offline.

For a multi-world build, a typical pattern is to swap every world's `references:` URL from the deployed URL to a local `file://` before running the patched binary, then restore for CI. See [`katlas-worlds/justfile`](https://github.com/katlas-space/katlas-worlds) for a worked `build-site-local` recipe.

## Build from source (for development)

```bash
git clone https://github.com/katlas-space/mystmd-local-refs.git
cd mystmd-local-refs

# clone the matching upstream mystmd
git clone --depth 1 --branch v1.11.0 https://github.com/jupyter-book/mystmd.git upstream
cd upstream
for p in ../patches/*.patch; do git apply "$p"; done

# requires bun — https://bun.sh/install
bun install
bun run build

ls packages/mystmd/dist/myst.cjs   # the built binary
```

## Release cadence

Tags are of the form `mystmd-<upstream-ver>+<patch-rev>`:

| Tag | Upstream | Patch revision |
|---|---|---|
| `mystmd-1.11.0+1` | jupyter-book/mystmd@v1.11.0 | our patch rev 1 |
| `mystmd-1.12.0+1` | jupyter-book/mystmd@v1.12.0 | our patch rev 1 (rebased) |

The [`.github/workflows/release.yml`](.github/workflows/release.yml) workflow:
1. On tag push, parses the upstream version from the tag
2. Clones upstream at that tag
3. Applies every `patches/*.patch`
4. Runs `bun install && bun run build`
5. Attaches `myst.cjs` + the patches + a `VERSION` sidecar to the Release

To cut a release:

```bash
git tag mystmd-1.11.0+1
git push --tags
```

## Upstream status

Not yet contributed upstream. The patch is small and focused and addresses a real cross-ecosystem gap (same complaint as Sphinx #12055). PR against jupyter-book/mystmd is on the roadmap — see `UPSTREAM-PR-DRAFT.md` once drafted.

## License

MIT. See [`LICENSE`](LICENSE).
