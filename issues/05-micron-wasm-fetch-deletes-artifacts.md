# A failed fetch of the micron WASM deletes the vendored files

## Symptom

During the second (successful) frontend build, the session reported this
prebuild message:

> micron-wasm reported a fetch failure after finding the file unchanged; the
> Vite build still completed.

The session called it "harmless", but it was not. Afterwards, these are the
results:

- `meshchatx/src/frontend/public/vendor/micron-parser-go/` is **empty**, so
  `micron-parser-go.wasm` and `wasm_exec.js` are both gone.
- `git status` shows the tracked
  `vendor/micron-parser-go/integrity.json` as **deleted**.

## Cause

Look at the `catch` block in `scripts/fetch-micron-wasm.mjs`, around line 239.
If any step fails **after** `SHASUMS256` was fetched successfully, the script
deletes the WASM, `wasm_exec.js` and `integrity.json`, then exits with code 0.

In this run:

1. The WASM hash matched upstream, so it was "unchanged".
2. A later step failed. That was most likely the `wasm_exec.js` fetch from
   `raw.githubusercontent.com/golang/go/go1.26.2/...` or
   `assertWasmExecCompat`; the exact error was not captured.
3. The script deleted the good local files.

Because the script exits with 0, the Vite build goes ahead without the micron
parser.

## How to recover

```bash
git checkout -- meshchatx/src/frontend/public/vendor/micron-parser-go/integrity.json
node scripts/fetch-micron-wasm.mjs        # needs network; check that it prints "OK"
ls meshchatx/src/frontend/public/vendor/micron-parser-go/
pnpm run build-frontend                   # rebuild so meshchatx/public picks it up
```

To keep existing artifacts when the network is unreliable, set either
`MICRON_WASM_SKIP=1` or `MESHCHATX_OFFLINE_BUILD=1`. The second fails loudly
if the files are missing.

## Possible upstream improvement

Do not delete artifacts that were verified as unchanged when only a later step
failed. Or exit with a non-zero code, so the build does not quietly ship
without the micron parser.
