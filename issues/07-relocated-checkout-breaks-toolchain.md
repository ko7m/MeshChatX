# After the rsync, `.cache/` and `.venv/` point at the old path

The original build was done in `/home/jeffwh/MeshChatX` on the old machine.
On `m55p`, the tree is at `/home/jeffwh/git/MeshChatX`, and some of the
toolchain still points at the old location.

## What breaks

| Item | Problem |
|------|---------|
| `.cache/ci-toolchain/bin/{go,node,npm,pnpm}` | Absolute symlinks to `/home/jeffwh/MeshChatX/.cache/...`, a path that does not exist here |
| `.venv/bin/python` | Symlink to `~/.local/share/uv/python/cpython-3.14-linux-x86_64-gnu/...`. That Python lived on the old machine; this machine only has cpython 3.12 |
| `.venv/bin/*` console scripts | Their shebangs point at `/home/jeffwh/MeshChatX/.venv/bin/python` |
| `uv`, `task` and the `tinygo` binaries | These are real files, not symlinks, so they still work |

These tools on `m55p`'s own PATH do **not** meet the pinned versions:

- `go` 1.26.0 (1.27.1 wanted)
- `node` 22.22.1 (24 or later required)
- `uv` 0.12.23 (this one is fine)

## Fix: rebuild the local toolchain and venv here

```bash
cd ~/git/MeshChatX
rm -rf .cache/ci-toolchain/bin/{go,node,npm,pnpm} .venv
bash scripts/ci/bootstrap-toolchain.sh install     # restores node/pnpm/go links and installs Python 3.14
export PATH="$PWD/.cache/ci-toolchain/bin:$PWD/.cache/ci-toolchain/tinygo/tinygo/bin:$PATH"
export TINYGO="$PWD/.cache/ci-toolchain/tinygo/tinygo/bin/tinygo"
task install                                       # recreates .venv, reruns the LXST patches and the filterlib build
```

The bootstrap script skips any tool that is already new enough. With the
broken links removed, it reinstalls Node and pnpm, and Go if needed.

For a clean start instead, delete all of `.cache/ci-toolchain` and follow
[01](01-build-from-source.md) from step 1.

These notes were written from the current state of the tree. The rebuild on
`m55p` has not been run yet.
