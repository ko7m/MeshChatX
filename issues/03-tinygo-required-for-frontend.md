# `build-frontend` needs TinyGo, but the bootstrap script does not install it

## Symptom

`pnpm run build-frontend` failed the first time it ran, right after
`task install` succeeded.

`prebuild-frontend` runs `scripts/build-visualiser-wasm.mjs` and
`scripts/build-geo-wasm.mjs`. Both scripts call `tinygo` to compile the
network-visualiser and map WASM modules. TinyGo was not installed.

## Cause

`scripts/ci/bootstrap-toolchain.sh` installs uv, Python, Node, pnpm, Go and
Task, but **not TinyGo**. CI installs TinyGo in a separate step, with
`TINYGO_VERSION: "0.42.0"`, in these workflows:

- `ci.yml`
- `frontend-build.yml`
- `build-release.yml`

The step runs `sh scripts/ci/setup-tinygo.sh <ver>`, which needs root because
it installs to `/usr/local/tinygo`.

## Fix used

The session installed TinyGo 0.42.0 from the release tarball into
`.cache/ci-toolchain/tinygo`, without sudo, and set `PATH` and `TINYGO`. The
commands are in [01](01-build-from-source.md), step 2.

The scripts find TinyGo from the `TINYGO` environment variable, or else from
PATH. To override where `wasm_exec.js` comes from, set `VISUALISER_GO_WASM_EXEC`
or `GEO_GO_WASM_EXEC`.

## Ways to skip the WASM build

- `VISUALISER_WASM_SKIP=1` and `GEO_WASM_SKIP=1` skip the TinyGo step. Use
  them once `visualiser.wasm` and `geo.wasm` already exist in
  `meshchatx/src/frontend/public/vendor/{visualiser,geo}-wasm/`.
- `MESHCHATX_REQUIRE_VISUALISER_WASM=1` and `MESHCHATX_REQUIRE_GEO_WASM=1`
  do the opposite: they make a missing TinyGo a hard error.

## Possible upstream improvement

Add `TINYGO_VERSION=0.42.0` to `scripts/ci/versions.env`, and have
`bootstrap-toolchain.sh install` fetch TinyGo into `$toolchain/tinygo` the
way it already fetches Go.
