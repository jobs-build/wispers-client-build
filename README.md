# wispers-client-build

A real-world [JOBS](https://github.com/draganm/jobs) build of
[s-te-ch/wispers-client](https://github.com/s-te-ch/wispers-client) — the Wispers
Connect client library (`libwispers_connect.so`) and its two CLIs (`wcadm`,
`wconnect`) — hermetic, offline (`net=none`), reproducible, on amd64 or arm64.

## Why this is interesting

wispers-client is not a toy: it is a Cargo workspace whose build needs a whole
native toolchain besides rustc —

| upstream needs | filled hermetically by |
|---|---|
| C/C++ compiler (vendored BoringSSL via the `boring` crate, `ring` asm) | Alpine v3.22 `gcc`/`g++` apks staged into the sandbox root |
| `cmake` + `make` (BoringSSL, libjuice) | Alpine apks |
| `libclang` (bindgen for libjuice + boring bindings) | Alpine `clang20-libclang` + `clang20-headers`, `LIBCLANG_PATH` |
| `protoc` (tonic-build gRPC codegen) | Alpine `protoc` apk, `PROTOC` |
| OpenSSL headers + libs (`wcadm`'s reqwest → native-tls) | Alpine `openssl-dev`; `libssl`/`libcrypto` shipped in `lib/` next to the binaries |
| the `libjuice` git submodule (github tarballs exclude submodules) | a separate pinned `github` import copied into place |
| Rust ≥ 1.85 (edition 2024) | the musl-hosted Rust 1.96 dist tarball |
| 326 crates.io dependencies | the `cargo` build plugin: one vendored import per crate in `Cargo.lock` |

Everything is content-addressed and pinned (apk sha256s, crate checksums, the
upstream + submodule commit SHAs), so the whole build runs with the network
disabled and caches/joins across runners.

## Layout

- `BUILD.jobs` — the submittable recipe: pins the upstream commit
  (`WISPERS_REF`) and drives `wispers.BUILD.jobs` against that tree as a
  sub-build (`bld(source = imp(github …), buildJobs = source.read(…))`), then
  re-exports the result. Because the sub-build's *source* is the real upstream
  repo, the cargo plugin reads the real upstream `Cargo.lock` — no copy lives
  here, and bumping `WISPERS_REF` (plus `LIBJUICE_REF` if the submodule moved)
  is a one-line change.
- `wispers.BUILD.jobs` — the actual build recipe (see its header comment for the
  full gap-by-gap story).

## Output

```
bin/wcadm                 CLI for managing Wispers Connect domains
bin/wconnect              connectivity test / sidecar utility
lib/libwispers_connect.so the client library (cdylib)
lib/libssl.so.3, libcrypto.so.3, …   runtime shared libs ($ORIGIN/../lib RUNPATH)
include/wispers_connect.h the C FFI header
JOBS.entrypoint           runs `wconnect --help`
```

The binaries are dynamic musl ELFs; their interpreter and RUNPATH point at the
`hostmusl` loader's content-addressed store path (a `runtime_dep`), so
`jobs run` / `jobs image` resolve everything from the materialized closure.

## Building

From a JOBS checkout with the local dev environment up:

```bash
jb run --source .            # build + run the entrypoint (wconnect --help)
```

or submit to an engine with this repo as the `github`-fetched source.
