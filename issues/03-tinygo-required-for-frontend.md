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

## Alternative: copy the WASM from a machine that already built it

`visualiser.wasm`, `geo.wasm` and their `wasm_exec.js` files are gitignored,
but the `integrity.json` files committed to the repo hold their SRI hashes.
If another machine has already built the same commit with TinyGo 0.42.0, copy
the files across and skip TinyGo completely. This is what got m93p working on
2026-10-12:

```bash
V=meshchatx/src/frontend/public/vendor
for d in visualiser-wasm geo-wasm; do
  scp "m55p:git/MeshChatX/$V/$d/*.wasm" "m55p:git/MeshChatX/$V/$d/wasm_exec.js" "$V/$d/"
done
VISUALISER_WASM_SKIP=1 GEO_WASM_SKIP=1 pnpm run build-frontend
```

Don't write this as `"$d/{*.wasm,wasm_exec.js}"`. Modern OpenSSH `scp` uses
the SFTP protocol, which expands `*` but not `{a,b}` braces. The brace form
fails with "No such file or directory".

If the files are missing and the skip flags are set, the build still
succeeds. The network visualiser and the map then use their slower JS
fallbacks.

## Possible upstream improvement

Add `TINYGO_VERSION=0.42.0` to `scripts/ci/versions.env`, and have
`bootstrap-toolchain.sh install` fetch TinyGo into `$toolchain/tinygo` the
way it already fetches Go.
