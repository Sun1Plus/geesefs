# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

GeeseFS is a high-performance, POSIX-ish FUSE filesystem for S3-compatible object storage, written in Go. It was forked from Goofys and is maintained by Yandex Cloud. The design trades strict POSIX semantics for parallelism and asynchrony (async writes/deletes/renames, parallel readahead and multipart uploads, disk cache). It has the most complete feature set against Yandex Object Storage (symlinks, special files, xattrs without extra RTT, PATCH partial updates); other S3 backends work but with reduced metadata fidelity.

## Common commands

Build (injects git hash into `main.Version`):
```
make build
# or directly: CGO_ENABLED=0 go build -ldflags "-X main.Version=$(git rev-parse HEAD)"
```

`CGO_ENABLED=0` is exported at the top of the Makefile — keep it that way; releases are static binaries. The GitHub CI uses Go 1.25 and builds with `env CGO_ENABLED=0 go build`; `go.mod` pins `go 1.25.9`.

Integration tests (spins up s3proxy against a local backend):
```
make run-test                          # full suite
./test/run-tests.sh TestName           # single check.v1 test by regex; forwarded as `-check.f`
SAME_PROCESS_MOUNT=1 make run-test     # what CI does; mounts in-process for clean go test output
```

Tests use `gopkg.in/check.v1`. `test/run-proxy.sh` downloads/launches `s3proxy.jar` on `PROXY_PORT` (default 8080) and sets `AWS_ACCESS_KEY_ID=foo AWS_SECRET_ACCESS_KEY=bar ENDPOINT=http://localhost:$PORT`. To point tests at a real cloud instead, unset `EMULATOR` and set `CLOUD=s3|gcs|azblob|adlv1|adlv2` plus real credentials (see the big comment block at the top of `core/goofys_test.go`).

xfstests (Linux only, requires sudo and many dev packages — see `test/run-xfstests.sh`):
```
make run-xfstests
```

Protobuf regeneration (cluster RPCs):
```
make protoc
```

Docker static build:
```
make docker-binary   # produces ./geesefs inside a Dockerfile.build container
```

## Replaced dependencies — important

`go.mod` contains three `replace` directives that are load-bearing. Don't update these modules from upstream without understanding why:

- `github.com/aws/aws-sdk-go` → `./s3ext` (a vendored, lightly modified copy of the AWS SDK; the project's custom S3 extensions live here)
- `github.com/winfsp/cgofuse` → `github.com/vitalif/cgofuse` (Windows FUSE binding fork)
- `github.com/jacobsa/fuse` → `github.com/vitalif/fusego` (Linux/macOS FUSE binding fork)

If you need to add calls into the AWS SDK and can't find a symbol, check `s3ext/` — it's a full SDK copy, not a thin shim.

## Architecture

Everything substantial lives in `core/`. The package is still called `core` (was `internal`) and `Goofys` is still the central struct name — this is historical, not a different codebase.

**Entry point & mount glue.** `main.go` + `main_nowindows.go` + `main_windows.go` parse flags, daemonize on non-Windows, and hand off to `core.Mount`. Logging goes through `core/cfg` (logrus). All tunables live on `*cfg.FlagStorage`, which is passed through the entire stack — when adding behavior, prefer a new flag there over a new global.

**FUSE bindings are platform-split.** `goofys_fuse.go` (Linux/macOS via jacobsa/fuse fork) and `goofys_windows.go` (Windows via cgofuse/WinFSP fork) are two different adapters over the same `Goofys` object. Both ultimately call methods on `Goofys` in `goofys.go`. Don't assume a change to one side applies to the other.

**Inode / handle / dir model.** `goofys.go` owns the inode table and `nextInodeID` allocation. `dir.go` and `file.go` implement per-inode semantics; `handles.go` maps FUSE handle IDs to open-file state. The `Goofys.mu` lock protects FS-global state (inode map, etc.); per-inode locks must be acquired before `Goofys.mu` is released — see the comment block on the struct.

**Buffering and async I/O.** This is the performance heart of the project and is where most bugs live:
- `buffer_pool.go` — bounded memory pool, honors `--memory-limit`
- `buffer_list.go` — dirty-range tracking per inode, plus multipart part accounting. `part_size_test.go` covers the tiered part-size scheme (1000×5MB + 1000×25MB + 8000×125MB = ~1.03 TB default limit).
- `buffer_queue.go` + `buffer_reader.go` — readahead pipelines
- `fd_queue.go`, `cgroup.go` — flusher scheduling and resource accounting
Writes land in buffers, flushers drain them to S3 in the background; `fsync` forces the drain and is the only way to surface write errors synchronously.

**Backends.** `backend.go` defines the `StorageBackend` interface; implementations are `backend_s3.go`, `backend_azblob.go`, `backend_gcs3.go`, `backend_adlv1.go`, `backend_adlv2.go`. Per the README, GCS and ADL backends are inherited-but-broken; S3 and Azure Blob are the maintained paths. `v2signer.go` implements AWS SigV2 for older Ceph/S3proxy setups. `--list-type 1` or `2` is required for any non-Yandex S3.

**Cluster mode.** `cluster_*.go` + `core/pb/` implement a multi-node mode where peers coordinate inode ownership over gRPC. `cluster_inode.go` contains the inode-stealing protocol; `cluster_fs.go`/`cluster_fs_fuse.go` are the FUSE adapter. This is separate from single-node operation — most code paths don't run it. `test/cluster/` has shell-based integration tests.

**PATCH path (Yandex-only).** `--enable-patch` enables partial object updates via the PATCH verb, bypassing the usual read-modify-write/multipart-copy flow. The write path branches on this flag in multiple places in `file.go` and the S3 backend; concurrent PATCH conflicts are retried unless `--drop-patch-conflicts` is set.

## Testing notes

- The Go tests use `go-check` (`gopkg.in/check.v1`), not stdlib `testing.T`. Run a single test with `./test/run-tests.sh TestName` — this forwards to `-check.f`.
- Tests require a live S3-compatible endpoint. Without `EMULATOR=1` / s3proxy, tests hit whatever `ENDPOINT` + credentials you set and create a throwaway bucket if `BUCKET` is empty.
- CI runs `SAME_PROCESS_MOUNT=1` so failures show up in stdout; without it, the mount runs as a child and logs go elsewhere.
- xfstests are best-effort in CI (`continue-on-error: true`) due to memory limits; don't be surprised if they're flaky locally.

## Gotchas

- Version numbers live in `debian/changelog` and release tags; `main.Version` is populated at link time from git HEAD. Don't hardcode.
- Not all backends support all features — before adding a new `StorageBackend` method, check what each existing backend can actually implement. `Capabilities` in `backend.go` is the declarative feature flag.
- Cluster mode and PATCH are mutually aware of each other in a few places; changes to write flushing should be tested with both off and at least one on.
- CloudFlare R2 is documented as broken (returns 403 instead of 429 on throttling). Don't add workarounds without checking upstream status.
