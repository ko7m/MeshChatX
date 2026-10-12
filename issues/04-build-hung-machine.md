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

## It happened again on 2026-10-12 (fresh clone on m93p)

The setup ran again on the same 7 GB machine, from a fresh clone of the
`ko7m/MeshChatX` fork. All of steps 1 to 4 were pasted into one terminal. This
time the machine did not hang. GNOME showed this message instead:

> Virtual Terminal Stopped. Device memory is nearly full. Virtual terminal
> processes were using a lot of memory and were forced to stop.

That is **systemd-oomd**. When memory pressure stays high, it kills the
terminal's entire cgroup, so the shell and the build die together. Steps 1
and 2 (toolchain and `task install`) had already finished, so the kill
happened during the frontend build.

### What worked: run each heavy step in its own capped scope

```bash
export PATH="$PWD/.cache/ci-toolchain/bin:$PATH"
cap() { systemd-run --user --scope -q -p MemoryHigh=3G -p MemoryMax=4G -p MemorySwapMax=1G "$@"; }

cap bash scripts/ci/bootstrap-toolchain.sh install
COREPACK_ENABLE_DOWNLOAD_PROMPT=0 cap task install
VISUALISER_WASM_SKIP=1 GEO_WASM_SKIP=1 NODE_OPTIONS=--max-old-space-size=2560 \
  cap pnpm run build-frontend
cap uv run python -m meshchatx.meshchat --headless --host 127.0.0.1
```

`systemd-run --scope` puts the command in its own cgroup with a memory limit.
If that group runs out of memory, only the command is killed and the terminal
survives. With the browser closed, `build-frontend` took about 28 s and stayed
within the limit. TinyGo was skipped completely; see
[03](03-tinygo-required-for-frontend.md) for how to copy the WASM from another
machine.

## Recommendations for machines with little RAM

- Run each heavy step under `cap` (above), one step at a time, rather than
  pasting the whole sequence into one terminal.
- Avoid TinyGo on the small machine. Copy `visualiser.wasm` and `geo.wasm`
  from a machine that already built them, or let the JS fallback run.
- If you do build the WASM locally, run the two WASM scripts by themselves
  first, with nothing else heavy running. Then run Vite with the WASM skip
  flags set.
- Keep `NODE_OPTIONS=--max-old-space-size=2560`, or a similar limit, and close
  the browser while you build.
- This is not a problem on `m55p` (6 cores, 30 GB of RAM).
