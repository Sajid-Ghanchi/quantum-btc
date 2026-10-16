# Linux setup guide

Run a Q-BTC node and miner on Debian/Ubuntu, with or without an NVIDIA GPU.

## One-line install

```bash
curl -fsSL https://raw.githubusercontent.com/Q-Jack-core/quantum-btc/main/scripts/install.sh | bash
```

Prefer to read it first (recommended):

```bash
curl -fsSLO https://raw.githubusercontent.com/Q-Jack-core/quantum-btc/main/scripts/install.sh
less install.sh
bash install.sh
```

Pin a release instead of taking the newest tag:

```bash
bash install.sh --tag v2.6.0
```

See what the installer detects without changing anything:

```bash
bash install.sh --check
```

## What the installer does

1. Checks OS, CPU architecture, RAM, free disk, NVIDIA GPU and CUDA toolkit.
2. Installs build dependencies with `apt` and Rust with `rustup`.
3. Clones this repository and checks out a release tag (newest `vX.Y.Z` unless you pass `--tag`).
4. Runs `cargo build --release`. `build.rs` compiles the CUDA kernel if a toolkit is found and otherwise builds a CPU stub (see [GPU_MINING.md](GPU_MINING.md)).
5. Reports whether the binary was built **with GPU support**, because the build never fails when CUDA is missing; it silently becomes CPU-only.
6. Installs a launcher, `qbtc`, into `~/.local/bin`.

It does not create wallets, does not ask for or handle seed phrases, and does not modify an existing data directory.

## Options

| Option | Meaning |
|---|---|
| `--tag <vX.Y.Z>` | Release tag to build (default: newest) |
| `--dir <path>` | Source checkout (default `~/quantum-btc`) |
| `--datadir <path>` | Node data and wallet folder (default `~/.qbtc`) |
| `--port <port>` | P2P port (default `8000`; RPC uses port + 1) |
| `--install-cuda` | If an NVIDIA GPU is present but `nvcc` is missing, install the distribution package `nvidia-cuda-toolkit` first |
| `--check` | Report only |

Environment overrides: `QBTC_TAG`, `QBTC_DIR`, `QBTC_DATADIR`, `QBTC_PORT`, `QBTC_REPO`, `CUDA_PATH`.

## What it works on

| System | Result |
|---|---|
| Debian/Ubuntu, x86_64, no GPU | Builds and mines on the CPU |
| NVIDIA GPU (compute capability 6.1+, i.e. GTX 10-series and newer), NVIDIA driver installed, CUDA toolkit installed | GPU build |
| NVIDIA GPU + driver, no CUDA toolkit | CPU-only build. Add `--install-cuda`, or install the CUDA toolkit yourself and re-run |
| NVIDIA GPU older than compute capability 6.1 | Not supported by the CUDA kernel |
| AMD / Intel GPU | No GPU backend (no OpenCL/ROCm). The node builds and mines on the CPU |
| Fedora, Arch and other non-apt distributions | The installer stops with a message; follow [Manual install](#manual-install) |
| ARM64 | Untested (CPU only at best) |
| WSL2 | Untested |
| Windows / macOS | Not this script; use the Desktop GUI or the `Start_Mining` launchers in the README |

The NVIDIA **driver** is never installed by this script. `nvidia-smi` must already work.

CUDA code is built for `sm_61`, `sm_75`, `sm_86` plus PTX for `compute_86`, so newer GPUs rely on driver JIT. Details and tuning: [GPU_MINING.md](GPU_MINING.md).

## After installing

```bash
qbtc start        # node in a tmux session; detach with Ctrl+b then d
qbtc attach       # come back to it
qbtc status       # process, backend and last hashrate line
qbtc scrub        # clear the node's tmux screen and scrollback (see COMMANDS.md)
```

Then, at the node prompt, after the node has finished syncing:

```text
wallet_gen mywallet
auto_mine start mywallet
```

All commands, flags, environment variables and **seed phrase handling** are in [COMMANDS.md](COMMANDS.md).

On a GPU build, the launcher selects the CUDA backend. The first log lines should show:

```text
[INFO] Miner: CUDA backend selected
[INFO] Miner: CUDA device 0 ready: <your GPU>
[INFO] Miner: mining on 1 GPU(s): [0]
```

If you see `CPU backend selected` instead, the node is mining on the CPU.

## Manual install

```bash
sudo apt-get update
sudo apt-get install -y build-essential clang libclang-dev cmake pkg-config libssl-dev git tmux curl
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
. "$HOME/.cargo/env"

git clone https://github.com/Q-Jack-core/quantum-btc.git
cd quantum-btc
git checkout v2.6.0
export CXXFLAGS="-include cstdint"      # needed by librocksdb-sys on newer GCC
cargo build --release 2>&1 | tee build.log
grep -E "CUDA kernel compiled|No CUDA toolkit|CUDA build failed" build.log

QBTC_MINER_BACKEND=cuda ./target/release/quantum-btc --datadir ~/.qbtc
```

## Troubleshooting

**`No CUDA toolkit found ... Building without GPU support`**: `nvcc` was not found. Install the CUDA toolkit (or use `--install-cuda`), or set `CUDA_PATH`, then rebuild.

**`CUDA build failed; falling back to the CPU stub`**: the lines after it contain `nvcc`'s own error. A frequent cause is a `gcc` newer than your CUDA toolkit supports. Install an older `g++` that your toolkit supports and point `nvcc` at it, for example `NVCC_PREPEND_FLAGS='-ccbin g++-12' bash install.sh`.

**`Miner: no CUDA device could be initialized`**: the binary has GPU support but the driver sees no usable device. Check `nvidia-smi`.

**`error while loading shared libraries: libcudart.so`**: the CUDA runtime library is not on the loader path. The `qbtc` launcher sets it for toolkits under `/usr/local/cuda*`; if you start the binary by hand, add the toolkit's `lib64` directory to `LD_LIBRARY_PATH`.

**Build runs out of memory**: the release profile uses fat LTO. Add swap or build on a machine with more RAM.

**A rocksdb compile error mentioning `uint8_t` / `cstdint`**: build with `export CXXFLAGS="-include cstdint"` (the installer does this).

## Uninstall

```bash
rm -f ~/.local/bin/qbtc
rm -rf ~/quantum-btc          # source and build output
# Your data directory (default ~/.qbtc) holds your wallets. Delete it only if
# every wallet's seed phrase is safely backed up.
```

## Tested on

| Environment | GPU | Result |
|---|---|---|
| RunPod `runpod/pytorch:2.4.0-py3.11-cuda12.4.1-devel-ubuntu22.04` (manual steps above) | RTX 3090 | GPU build, mining, ~4.8 GH/s |
| _to be filled in with `install.sh` runs_ | | |
