# Krexa sandbox benchmarks

Measured performance and isolation results for the Krexa trading-agent sandbox, as reported in the paper *Separating Intelligence from Authority: A Layered Enforcement Architecture for Autonomous Trading Agents* (Krexa, draft v0.2, 2026).

Every number below comes from the raw files in [`data/`](data/). No value is estimated.

## What was measured

| Name in this repo | What it is |
|---|---|
| **VM B** (`microvm-current`) | The current Krexa sandbox runtime (Rust). A KVM microVM per call with no network device, a 0.88 MB single-binary guest, snapshot restore and a warm pool. |
| **VM A** (`microvm-first`) | Our first microVM runtime (TypeScript host, Python guest on an Alpine root filesystem). Used as the baseline. |
| **Native engine** (`js-engine/native`) | QuickJS-NG in native Rust (`jsbox`), run in a pool of seccomp-confined worker processes (`jsd`). |
| **WASM baseline** (`js-engine/wasm-baseline`) | The production engine: QuickJS compiled to WebAssembly (quickjs-emscripten 0.32.0) on Bun 1.4.4. |
| **Enforcement path** (`enforcement-path`) | Gateway → signer verdict latency and throughput. |

## Hosts

- **AWS `c8i.4xlarge`**, ap-southeast-2 (Sydney): Intel Xeon 6975P-C, 16 vCPU, 30 GiB RAM, Ubuntu 24.04 x86_64. **Nested virtualization** (`NestedVirtualization=enabled`), **not bare metal**. Date: 2026-11-23.
- **Apple M4 Pro** (14 cores, 24 GB). Engine and enforcement runs are native. MicroVM runs use nested KVM inside a Linux VM (8 vCPU, 12 GiB). Dates: 2026-11-20/21.
- Guest kernel 6.1.188. 100 runs per microVM value, 1,500–5,000 per engine value, unless stated.

## Results

**Table 3. MicroVM performance on AWS `c8i.4xlarge` (nested virtualization; typically 100 runs per value; n per row in the bench-*.json files).**

| Metric | VM A (first runtime) | VM B (current runtime) | Change |
|---|---|---|---|
| Cold boot to ready, p50 / p99 (ms) | 828 / 846 | 254 / 264 | 3.3× faster |
| Snapshot restore to ready, p50 / p99 (ms) | 71 / 77 | 32 / 41 | 2.2× faster |
| Warm VM round trip, p50 / p99 (ms) | 0.44 / 1.01 | 0.12 / 0.64 | 3.6× faster |
| Decision, fresh single-use VM from the warm pool, p50 / p99 (ms) | not built | 3.3 / 4.0 | — |
| Decision through the daemon's HTTP API, p50 / p99 (ms) | not built | 3.1 / 4.3 | — |
| Decision, VM kept for one user, p50 / p99 (ms) | not built | 0.9 / 7.7 | — |
| Decision, paused VM resumed, p50 / p99 (ms) | not built | 39 / 41 | — |
| Restore + init + first decision, no pre-warm, p50 / p99 (ms) | not built | 8.4 / 10.0 | — |
| Memory, cold-booted idle VM (PSS, MiB) | 59.3 | 48.8 | 18% less |
| Memory, restored idle VM (PSS, MiB) | 57.1 | 19.1 | 3.0× less |
| Memory, paused VM (PSS, MiB) | not measured | 3.1 | — |
| Memory per VM, 16 VMs sharing one snapshot (PSS, MiB) | not measured | 5.7 | — |
| Idle CPU, 16 / 64 idle VMs (% of one CPU, total) | not measured | 0.8 / 3.2 | — |
| Maximum concurrent VMs per host | run aborted (note 2) | 487 started, 429 answering at the end | — |
| Host daemon memory (RSS, MiB) / idle CPU | — | 4.3–5.2 / 0% | — |
| Isolation suite | 8/9 pass | 12/12 pass | — |

*Notes to Table 3.*
1. VM B's ramp stopped when VM 488 did not become ready within 10 s. 429 of the 487 VMs still answered a ping at the end. This is a stop criterion, not a clean capacity limit. At that point the 487 idle VMs used 108% of one CPU in total and 2.65 MiB of host memory each.
2. VM A's maximum-concurrency run ended when one VM process exited during the ramp, so no value is reported.
3. VM A has no warm pool and no decision path, so only its raw round trip is reported.
4. On the Apple M4 Pro nested host, the same VM B code measured 1,653 ms cold boot, 149 ms restore, 9.1 ms fresh-VM decision and a maximum of 62 VMs (`data/microvm-current/apple-m4-nested/`). The AWS values are 2.8× to 7.9× better. Nested-virtualization overhead therefore depends strongly on the host, and bare-metal values are expected to be lower again. A bare-metal run is pending a vCPU quota increase.

**Table 3b. Strategy engine, time per decision (ms, p50 / p99; Apple M4 Pro, native).** Each call uses a fresh JavaScript context, as in production.

| Workload | Old: WASM, in-thread | Old: WASM, worker pool | New: native engine | New: process pool |
|---|---|---|---|---|
| Trivial strategy (n = 5,000) | 0.17 / 0.21 | 0.20 / 0.26 | 0.09 / 0.22 | 0.14 / 0.19 |
| DIP strategy, 120 bars (n = 5,000) | 0.43 / 0.50 | 0.50 / 0.72 | 0.20 / 0.28 | 0.34 / 0.40 |
| Replay, 4,261 decisions over 4,380 real bars | 0.43 / 0.61 | 0.50 / 0.61 | 0.20 / 0.23 | 0.34 / 0.50 |
| 500 bars × 6 series (n = 1,000 to 5,000) | 6.24 / 7.19 | 7.32 / 8.04 | 3.88 / 4.56 | 6.08 / 6.93 |
| Submit to first decision, warm (n = 1,500) | — | 0.70 / 1.02 | — | 0.50 / 0.86 |

**Table 3c. Strategy engine throughput and cost (DIP strategy; 0 errors in all runs).**

| Setup | Decisions per second | Idle CPU | Memory per worker |
|---|---|---|---|
| Old: WASM pool, 2 workers (best, at concurrency 64) | 3,867 | 0.33% of one core | ≈12.2 MiB RSS per worker thread |
| New: process pool, 2 workers, through the Bun client | 7,836 (2.0×) | below 0.03% (below measurement resolution) | 3.2–3.9 MiB RSS idle; up to 9.7 MiB after 20,000 calls |
| New: process pool, 14 workers, through the Bun client | 16,851 | — | — |
| New: process pool, 14 workers, native load generator | 21,633 | — | — |

**Correctness.** The native engine gives the same decision as the production WebAssembly engine in every differential comparison: 444 cases × 5 seeds (2,220 comparisons, raw output in this repo) and 624 cases × 3 seeds (1,872 comparisons, raw output not retained). 14 cases per run differ only in error-message wording. See [`data/js-engine/differential/`](data/js-engine/differential/).

**Table 4. Order-path latency (ms; Apple M4 Pro host; p50 / p99).**

| Path segment | Bun | Node |
|---|---|---|
| Signer `decide()` in process (approve; forged, replayed and over-budget requests cost the same, because the signature check runs first) | 3.53 / 4.13 | 5.17 / 5.59 |
| Intent in → verdict out: gateway → signer over loopback HTTP, signer in its own process | **16.46 / 17.41** | 20.31 / 21.58 |
| — of which ed25519 sign and verify (tweetnacl, pure JavaScript) | ≈14.4 | — |
| Bare loopback HTTP floor (keep-alive / new connection) | 0.054 / 0.122 | — |
| Envelope decrypt with a local KMS stand-in (AES-GCM + zeroize) | 0.0013 / 0.0047 | — |
| Signer throughput, one process | ≈110 requests/s | ≈76 requests/s |

**Table 4b. Venue network round trip from the same host (read-only requests; not an order acknowledgement).**

| Venue endpoint | Warm p50 (ms) | Cold p50, new TCP + TLS (ms) |
|---|---|---|
| Hyperliquid `/info` | 327 | 292 |
| Base mainnet RPC | 253 | 376 |
| Monad mainnet RPC | 114 | 202 |
| Polymarket CLOB | not measured: blocked by the ISP at DNS level | — |
| Binance futures | 1 of 20 succeeded (ISP block): 227 | 202 (20 of 20) |
| Jupiter | 174 (29 of 200; 85% HTTP 429) | 195 (30 of 200) |

**Table 5a. Sandbox isolation suite, run inside real guests on AWS `c8i.4xlarge`.**

| Group | Property tested | VM A (first runtime) | VM B (current runtime) |
|---|---|---|---|
| — | Secret-bearing VM configurations are refused before launch | Pass | (covered by group c) |
| a | Network: only loopback exists; DNS, TCP and 169.254.169.254 fail | Pass | Pass |
| b | Host files: none visible; VM B has no disk (initramfs only) | Pass | Pass |
| c | No keys in guest files, environment, kernel command line or RAM (whole-RAM scan from the host, with positive controls) | Pass (files, env, cmdline) | Pass (incl. RAM) |
| d | Snapshot/restore lifecycle; no state leak after recycle | Pass | Pass |
| e | Memory bomb and CPU spin contained; cgroup limit binds; neighbour VM healthy | Pass | Pass |
| f | Escape attempts as root with all capabilities: fork bomb, `/dev/mem`, mounts, raw sockets, kernel oops | **Fail** (note) | Pass |
| g | Two VMs cannot reach each other; identity bound by the socket | Pass | Pass |
| h | Jail: own user ID, seccomp mode 2 on all threads, cgroup limits, namespaces, no network-device file descriptor | Pass | Pass |
| i | Production image: no debug probes, init runs once, real engine, deterministic output | — | Pass |
| j | Wall clock: a guest that runs too long is timed out, marked poisoned and killed by the host | — | Pass |
| k | Intent path through the guard; bounded reads from a hostile guest | — | Pass |
| l | Control API reachable only on a local Unix socket | — | Pass |
| | **Total** | **8/9** | **12/12** |

*VM A, group f:* on the Intel host, root inside VM A's guest could write to `/dev/mem` at offset 0. This is the guest's own memory, not a host escape. VM B's guest has no `/dev/mem`.

## Data layout

| Folder | Contents |
|---|---|
| `data/microvm-current/aws-c8i/` | VM B on AWS: `bench-*.json` (summaries), `bench-*.csv` (every sample), `isolation-results.json`, `host-spec.txt` |
| `data/microvm-current/apple-m4-nested/` | VM B on the Apple host (nested) |
| `data/microvm-first/aws-c8i/` | VM A on AWS: cold boot, restore, memory, ping, isolation |
| `data/microvm-first/apple-m4-nested/` | VM A on the Apple host (nested), including the density ramp |
| `data/js-engine/native/` | Native engine and process pool: per-run CSVs, `summary.jsonl`, `rust-summary.jsonl`, `jsd-load.jsonl` |
| `data/js-engine/wasm-baseline/` | Production WebAssembly engine: latency, kill time, memory, density, idle CPU |
| `data/js-engine/differential/` | Differential test output against the production engine |
| `data/enforcement-path/` | Gateway → signer latency, throughput, KMS envelope, venue round trips |

Notes on the raw files:
- `host-spec.txt` in `microvm-current/aws-c8i/` notes that a header line from the benchmark tool wrongly names the Apple host. The real host is the AWS instance above.
- The local user name in probe paths is replaced with `USER`.

## Caveats

- The JS engine and process-pool timings were measured on macOS, where the worker seccomp filter does not apply (resource limits only). On Linux the filter is on; it was tested for correctness, not timed.
- All microVM numbers are nested virtualization. Bare-metal numbers are pending and are expected to be lower.
- The AWS and Apple values for the same code differ by 2.8× to 7.9×; compare only rows from the same host.
- The maximum-concurrency figure uses a 10 s readiness stop criterion, not a clean capacity limit.
- Venue round trips were measured from a home network in India and are read-only requests, not order acknowledgements.

## Cite

> Kher, Y., Mourya, H., and Bansal, M. (2026). Separating Intelligence from Authority: A Layered Enforcement Architecture for Autonomous Trading Agents. Krexa. Benchmark data: https://github.com/krexa-xyz/benchmark

## License

Data: CC BY 4.0.
