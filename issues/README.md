# Build notes: MeshChatX from source

These notes come from the Cursor session `2fd5807b-10b3-49cb-940f-2e3f55badfe1`
(2026-10-10). That session built MeshChatX in `~/MeshChatX` on the old machine
(4 cores, about 7 GB of RAM). The tree was later rsync'd to
`~/git/MeshChatX` on `m55p`.

The transcript records the commands and the assistant's messages, but not the
tool output. Where these notes say what happened, the source is the
assistant's own report, the saved terminal logs (`terminals/910718.txt` and
`180918.txt`), or the files the build left in the tree.

| File | Topic |
|------|-------|
| [01-build-from-source.md](01-build-from-source.md) | The full sequence that worked, start to finish |
| [02-toolchain-prerequisites.md](02-toolchain-prerequisites.md) | Toolchain and system packages: what is needed and where it comes from |
| [03-tinygo-required-for-frontend.md](03-tinygo-required-for-frontend.md) | `build-frontend` needs TinyGo, which the bootstrap script does not install |
| [04-build-hung-machine.md](04-build-hung-machine.md) | The frontend build hung a 7 GB machine, which then rebooted |
| [05-micron-wasm-fetch-deletes-artifacts.md](05-micron-wasm-fetch-deletes-artifacts.md) | A failed fetch of the micron WASM deleted the vendored files |
| [06-build-dirties-tracked-files.md](06-build-dirties-tracked-files.md) | The build rewrites tracked files and leaves `.cache/` untracked |
| [07-relocated-checkout-breaks-toolchain.md](07-relocated-checkout-breaks-toolchain.md) | After the rsync, `.cache/` and `.venv/` point at the old path |
| [08-espeak-missing.md](08-espeak-missing.md) | Voicemail TTS reports "eSpeak not found" at runtime |
