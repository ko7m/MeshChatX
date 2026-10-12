# The build rewrites tracked files and leaves `.cache/` untracked

After `task install` and `pnpm run build-frontend`, `git status` shows:

```
 M electron/assets/css/electron-shell.css
 M meshchatx/src/frontend/public/assets/css/electron-shell.css
 M meshchatx/src/frontend/public/vendor/geo-wasm/integrity.json
 D meshchatx/src/frontend/public/vendor/micron-parser-go/integrity.json
 M meshchatx/src/frontend/public/vendor/visualiser-wasm/integrity.json
?? .cache/
```

| Change | Why |
|--------|-----|
| `electron-shell.css` (both copies) | `scripts/build-electron-shell-css.mjs` regenerates the minified Tailwind output. The output differs only because of the local toolchain. |
| `geo-wasm/integrity.json`, `visualiser-wasm/integrity.json` | The `wasmExecSource` field records the **absolute local path** to TinyGo's `wasm_exec.js`. Upstream it was `/run/media/user1/projects/meshchatx/.tools/tinygo/...`; locally it is `/home/jeffwh/MeshChatX/.cache/ci-toolchain/tinygo/...`. The SRI hashes did not change. |
| `micron-parser-go/integrity.json` deleted | See [05](05-micron-wasm-fetch-deletes-artifacts.md). |
| `.cache/` untracked | `bootstrap-toolchain.sh` installs into `.cache/ci-toolchain`, but `.gitignore` only covers `/cache/`, not `.cache/`. |

## Recommendation

These are build artifacts, not real changes. Do not commit them. To discard
them:

```bash
git checkout -- electron/assets/css/electron-shell.css \
  meshchatx/src/frontend/public/assets/css/electron-shell.css \
  meshchatx/src/frontend/public/vendor/geo-wasm/integrity.json \
  meshchatx/src/frontend/public/vendor/visualiser-wasm/integrity.json \
  meshchatx/src/frontend/public/vendor/micron-parser-go/integrity.json
```

To hide the toolchain directory without changing the shared `.gitignore`, add
`.cache/` to `.git/info/exclude`.
