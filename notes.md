# My TiKV learning notes

Friendly step-by-step notes for this machine.
Not official docs — personal reference while reading the code.

See also: `design.md` (architecture and reading map), `CONTRIBUTING.md` (upstream guide).

---

## What TiKV is (one sentence)

TiKV is a **database server** (`tikv-server`).
Your app / TiDB talks to it over the network. It stores keys and values on disk (RocksDB) and copies writes with Raft.

```text
You / TiDB  →  gRPC  →  tikv-server  →  RocksDB on disk
                 ↑
                PD (cluster manager: timestamps + who owns which key)
```

---

## Two times code runs

| When | Example | What happens |
|------|---------|--------------|
| **Build time** | `make build`, `cargo build` | Compiles Rust + C++ deps. Runs `cmd/build.rs`. Makes the binary. |
| **Run time** | `./target/debug/tikv-server` | Starts the server process. Serves Get / Prewrite / Commit. |

`cmd/build.rs` runs only at **build time**.
It stamps `TIKV_BUILD_TIME` into the binary and links the C++ standard library.
It does **not** run while the server is up.

---

## Build commands (this Mac)

### What worked

```bash
export CMAKE_POLICY_VERSION_MINIMUM=3.5
make build
```

Or:

```bash
CMAKE_POLICY_VERSION_MINIMUM=3.5 cargo build
```

### What failed / avoid

```bash
make build          # alone → often dies on CMake 4 + snappy/c-ares
cargo build         # alone → same, or later Homebrew protobuf vs gRPC upb clash
```

Homebrew protobuf 34 headers under `/opt/homebrew/include` can break `grpcio-sys`.
If native build keeps fighting you: use Docker (see below).

### After a successful build — check the binary

```bash
./target/debug/tikv-server --version
```

You should see something like:

- Release Version
- Git Commit Hash
- **UTC Build Time** ← came from `cmd/build.rs` via `option_env!("TIKV_BUILD_TIME")`
- Rust Version / features / profile

Also try:

```bash
./target/debug/tikv-server --help
```

That only prints CLI help. It does **not** start a full cluster by itself.

Same for the control tool:

```bash
./target/debug/tikv-ctl --version
```

---

## Reading code — order that worked for me

1. `README.md` — product picture
2. `doc/maintenance-guides/repo-overview.md` — who owns what
3. `cmd/tikv-server/src/main.rs` — process entry
4. `components/server/src/server.rs` — `run_tikv` startup
5. `src/server/service/kv.rs` — gRPC handlers (`kv_get` → `future_get`)
6. `src/storage/mod.rs` — `Storage::get`
7. `src/storage/mvcc/reader/reader.rs` — real read of a key at `start_ts`
8. Later: `txn/scheduler.rs`, `prewrite.rs`, `commit.rs`, then raftstore

My longer map lives in `design.md`.

---

## How a request works (simple)

### Search / read (Get)

```text
Client → KvGet(key, start_ts)
      → kv.rs future_get
      → Storage::get
      → MVCC reader + RocksDB snapshot
      → return value or not_found
```

Usually **no** full Raft round-trip for a point Get.

### Store / write (transaction)

```text
1. Prewrite  → lock keys + hide new values
2. Commit    → publish values, clear locks
```

Both go through Raft (majority must copy the log before apply).

**Prewrite in plain words:**
“Reserve the keys and put the new data in a draft. Don’t show it to readers yet.”

---

## What I can actually test

### A. Binary exists / version stamp

Needs a successful local build first.

```bash
./target/debug/tikv-server --version
./target/debug/tikv-ctl --version
```

### B. Small unit tests (best while learning storage)

```bash
export CMAKE_POLICY_VERSION_MINIMUM=3.5
./scripts/test test_get -- --nocapture
./scripts/test test_get_put -- --nocapture
```

These exercise MVCC Get / Get+Put without needing a whole cluster.

### C. Fast type-check (when native deps allow)

```bash
CMAKE_POLICY_VERSION_MINIMUM=3.5 cargo check --all
```

### D. Full check before a real PR (heavy)

```bash
CMAKE_POLICY_VERSION_MINIMUM=3.5 make format
# make clippy / make test / make dev when your env is healthy
```

### E. Docker path (recommended on this Mac when native gRPC breaks)

```bash
make docker_test
```

First run is slow (image + Linux compile). Later runs reuse the image.
Closest to how CI builds.

### F. Run a real server (needs PD too)

`make run` does **not** start TiKV by itself.
You need PD + `tikv-server`. Easiest playground:

```bash
tiup playground
```

Or follow README: start `pd-server`, then `tikv-server` with `--pd-endpoints=...`.

---

## `build.rs` → `tikv-server` wiring (so I don’t forget)

```text
cmd/tikv-server/Cargo.toml
  [build-dependencies] cc, time

cmd/tikv-server/build.rs
  include!("../build.rs");     ← shared script

cmd/build.rs  (compile time only)
  sets TIKV_BUILD_TIME
  links libc++ / libstdc++

cmd/tikv-server/src/main.rs  (run time)
  option_env!("TIKV_BUILD_TIME")
  used in --version and startup log
```

Same shared `cmd/build.rs` is used by `tikv-ctl` too.

---

## Contribute without a perfect Mac build

1. Read + small change (often `src/storage` tests).
2. Branch from `master` for real work (keep `learn/repo-overview` for notes).
3. Test what you can locally; use Docker if Mac deps fail.
4. Open PR — **Linux CI** is the real compile gate.
5. PR needs: title like `storage: ...`, `Issue Number: ref #123`, `git commit -s`.

Local Mac failure from Homebrew/CMake ≠ automatic PR failure.

---

## Cheat sheet

```bash
# build (Mac)
export CMAKE_POLICY_VERSION_MINIMUM=3.5
make build

# prove binary works
./target/debug/tikv-server --version

# learn storage
./scripts/test test_get -- --nocapture
./scripts/test test_get_put -- --nocapture

# when Mac native deps break
make docker_test

# playground cluster (if tiup installed)
tiup playground
```

---

## Don’t touch first

- `components/raftstore/src/store/peer.rs`
- Server startup/shutdown order in `components/server`
- `RaftKv2` / `raftstore-v2` until classic path is clear
