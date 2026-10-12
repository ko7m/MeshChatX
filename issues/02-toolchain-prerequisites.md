# Toolchain and system prerequisites

## What was missing

The session's first environment check found that **`uv`, `pnpm` and `task`**
were missing. `node` was on PATH, but older than required: this machine has
Node 22.22.1, and the project requires Node 24 or later.

## Pinned versions

The versions come from `scripts/ci/versions.env`, which matches
`.github/workflows/ci.yml`.

| Tool | Version | Installed by | Location |
|------|---------|--------------|----------|
| uv | 0.11.15 | `bootstrap-toolchain.sh` (GitHub release tarball) | `.cache/ci-toolchain/bin/uv` |
| Python | 3.14 (got 3.14.5) | `uv python install 3.14` | `~/.local/share/uv/python/` |
| Node | 24.14.0 | `bootstrap-toolchain.sh` (nodejs.org tarball) | `.cache/ci-toolchain/node/` |
| pnpm | 12.7.0 (`packageManager` in `package.json`) | `npm install -g` into the toolchain node prefix | `.cache/ci-toolchain/node/bin/pnpm` |
| Go | 1.27.1 (only used to build `task`) | `bootstrap-toolchain.sh` (go.dev tarball) | `.cache/ci-toolchain/go/` |
| Task | 3.49.1 | `go install github.com/go-task/task/v3/cmd/task@v3.49.1` | `.cache/ci-toolchain/bin/task` |
| TinyGo | 0.42.0 | **manual**; see [03](03-tinygo-required-for-frontend.md) | `.cache/ci-toolchain/tinygo/tinygo/` |

`bootstrap-toolchain.sh verify` checks for uv, Node, pnpm and Task. It does
**not** check for TinyGo. The script skips any tool whose version on PATH is
already high enough.

## System packages assumed to be present

The session did not have to install any of these, but the build depends on
them:

- `curl` or `wget`, `tar` and `xz`, used by the bootstrap downloads
- `git`
- a C compiler (`gcc` / `build-essential`) for
  `scripts/build-lxst-filterlib.py`. Without one, the script exits with code 2.
- optional: `espeak-ng`, for voicemail TTS; see [08](08-espeak-missing.md)

## Network access during the build

The build needs network access to GitHub releases, nodejs.org, go.dev, the
Go module proxy, the npm registry, PyPI and `raw.githubusercontent.com`. For
an air-gapped machine, use `task bundle:offline` on a connected machine, then
`task install:offline` on the target, with `MESHCHATX_OFFLINE_BUILD=1` set.
