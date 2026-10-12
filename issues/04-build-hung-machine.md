# The full frontend build hung the machine, which then rebooted

## What happened

Machine: about 7 GB of RAM and 4 cores, running a GNOME desktop.

Timeline:

1. The session installed TinyGo and ran the full `pnpm run build-frontend`:
   the prebuild WASM compiles, then Vite.
2. Around **20:34**, `visualiser.wasm` and `geo.wasm` were written to disk.
3. Around **20:37**, GNOME started timing out: session clients, the shell and
   apps stopped responding.
4. At **20:42:11**, the system logged "The system will reboot now!". It came
   back up at 20:43.
5. After the reboot, `meshchatx/src/frontend/public/build` was missing, so
   Vite had not finished.

No OOM-killer lines turned up in `dmesg` or the journal. The diagnosis is
memory pressure and swap thrashing from TinyGo plus Vite. That is the most
likely cause, but it is **not proven**.

A side effect: the external SanDisk SSD had not been remounted yet after the
reboot, so `~/git` looked as if it had disappeared. Nothing was lost; the
automount had not caught up. (This comes from the separate Cursor transcript
`4116b3b1-…`.)

## What worked

Build in two stages, and give Node a heap limit so that it fails cleanly
instead of thrashing:

```bash
# WASM was already built before the hang, so skip TinyGo
export VISUALISER_WASM_SKIP=1 GEO_WASM_SKIP=1
export NODE_OPTIONS="--max-old-space-size=3072"
export UV_THREADPOOL_SIZE=2 GOMAXPROCS=1
pnpm run build-frontend
```

Vite finished in about 31 s, and `meshchatx/public/` was about 32 MB.

## Recommendations for machines with little RAM

- Run the two WASM scripts by themselves first, with nothing else heavy
  running. Then run Vite with the WASM skip flags set.
- Keep `NODE_OPTIONS=--max-old-space-size=3072`, or a similar limit.
- This is not a problem on `m55p` (6 cores, 30 GB of RAM).
