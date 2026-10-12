# Building MeshChatX from source: the sequence that worked

Run all commands from the repo root. This is what the Cursor session ran on
2026-10-10, cleaned up, with the fixes from the later issues already applied.

## 1. Bootstrap the pinned toolchain (no sudo)

```bash
bash scripts/ci/bootstrap-toolchain.sh install
export PATH="$PWD/.cache/ci-toolchain/bin:$PATH"
```

This installs uv, Python 3.14 (through uv), Node 24, pnpm and Go. It then
builds `task` with `go install`. Everything goes under `.cache/ci-toolchain/`.
The versions are pinned in `scripts/ci/versions.env`. The run took about 5
minutes, most of it spent downloading Go modules for `task`. See
[02](02-toolchain-prerequisites.md).

## 2. Install TinyGo 0.42.0 (needed by `build-frontend`)

The bootstrap script does not install TinyGo; see [03](03-tinygo-required-for-frontend.md).

```bash
TG_VER=0.42.0
TG_ROOT="$PWD/.cache/ci-toolchain/tinygo"
mkdir -p "$TG_ROOT"
curl -fsSL -o /tmp/tinygo.tgz \
  "https://github.com/tinygo-org/tinygo/releases/download/v${TG_VER}/tinygo${TG_VER}.linux-amd64.tar.gz"
tar -C "$TG_ROOT" -xzf /tmp/tinygo.tgz
export PATH="$TG_ROOT/tinygo/bin:$PATH"
export TINYGO="$TG_ROOT/tinygo/bin/tinygo"
tinygo version
```

(`sudo sh scripts/ci/setup-tinygo.sh 0.42.0` does the same thing, but installs to `/usr/local/tinygo`.)

## 3. Install the frontend and backend dependencies

```bash
export COREPACK_ENABLE_DOWNLOAD_PROMPT=0
pnpm config set verify-store-integrity true
task install
```

`task install` runs `pnpm install`, then `uv sync --group dev`, then three
post-install scripts:

- `patch_lxst_pyogg_ogg_ctypes.py`
- `patch_lxst_codec2_optional.py`
- `build-lxst-filterlib.py`, which needs a C compiler

## 4. Build the frontend

On a machine with little RAM, build the WASM modules first in a separate step,
then run Vite with a capped Node heap. See [04](04-build-hung-machine.md).

```bash
# a) WASM modules only (TinyGo), nothing else running
node scripts/build-visualiser-wasm.mjs
node scripts/build-geo-wasm.mjs

# b) Vite build, reusing the WASM from step (a)
export VISUALISER_WASM_SKIP=1 GEO_WASM_SKIP=1
export NODE_OPTIONS="--max-old-space-size=3072"
pnpm run build-frontend
```

The output goes to `meshchatx/public/` (about 32 MB). Vite took about 31 s.

## 5. Run the app (headless)

```bash
uv run python -m meshchatx.meshchat --headless --host 127.0.0.1
```

Open <https://127.0.0.1:8000>. The certificate is self-signed, so the browser
shows a warning. On first run, the app creates:

- a Reticulum identity in `storage/identity`
- the TLS certificate in `storage/ssl/`

The app turned on the Landlock and seccomp sandboxes automatically. The
`task run` command does the same thing, but runs `install` first.

## After the build

See [06](06-build-dirties-tracked-files.md) for the files the build modifies,
and [05](05-micron-wasm-fetch-deletes-artifacts.md) to check that the micron
WASM is still present.
