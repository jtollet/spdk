# SPDK VCL Socket Backend

This document describes the SPDK `vcl` socket backend, how it differs from the
standard `posix` and `io_uring` backends, how to build and run it, and what
performance characteristics were observed on the current validation platform.

The goal of this work is to let SPDK TCP transports use the VPP Communications
Library (VCL) as a socket backend, so that NVMe/TCP traffic can run on top of
VPP's user-space host stack instead of the Linux kernel TCP stack.

## What This Adds To SPDK

This evolution adds a new socket backend:

- `posix`: Linux socket API
- `uring`: Linux socket API + `io_uring`
- `vcl`: VPP Communications Library (`libvppcom`)

The VCL backend is exposed through the standard SPDK socket abstraction. It is
not a new transport type. Existing SPDK TCP users keep using the TCP transport,
but the transport can now be backed by VCL sockets.

This work also includes target-side changes in SPDK NVMe/TCP to better match
the VCL threading model:

- per-poll-group listeners for VCL-backed NVMe/TCP targets
- local accept on the owning poll group
- no post-accept cross-thread socket handoff for VCL target sockets

That target-side change is important because raw `vppcom` works best when a
session stays owned by one thread for its whole lifetime.

## Why VCL

Typical motivations are:

- accelerating NVMe/TCP with a user-space TCP/IP stack
- benchmarking SPDK over VPP host stack
- building high-performance storage gateways
- exploring an `RDMA <-> NVMe/TCP` proxy architecture

On the current validation setup, the VCL backend showed:

- a strong advantage over `posix` and `io_uring` on small-block (`4K`) traffic
- competitive large-block (`64K`) read throughput and a strong `64K` write
  advantage over the classic Linux socket backends
- near line-rate with jumbo frames once the descriptor depth and queueing model
  were tuned correctly

## Current Status

The backend is functional and has been validated with:

- SPDK NVMe/TCP target over VCL
- SPDK user-space initiator tools over VCL
- `VCL <-> VCL` traffic
- Linux kernel NVMe/TCP client interoperability against an SPDK/VCL target

Important implementation points:

- the VCL backend remains Linux-only
- target-side VCL uses per-poll-group listeners
- `c2h_success` optimization is disabled for VCL-backed NVMe/TCP listeners
  because it exposed an unstable path during validation

## Known Limitations

At the time of writing:

- some Linux `<->` VCL jumbo-frame interoperability cases still need more work
  under aggressive offload combinations
- VCL backend socket option support is intentionally conservative; some socket
  options return `-ENOTSUP`
- the initiator-side async connect path is functional, but still less polished
  than the mature `posix` backend

## Repository Changes Of Interest

The main areas changed by this work are:

- [`module/sock/vcl/vcl.c`](vcl.c)
- [`lib/nvmf/tcp.c`](../../../lib/nvmf/tcp.c)

Recent key commits on the branch include:

- `5f794f5c3` `nvmf/tcp: use per-poll-group listeners for vcl`
- `6bde91e6e` `nvmf/tcp: disable c2h success optimization for vcl`
- `a995feb60` `sock/vcl: tighten error handling and flush checks`
- `84baf8f98` `sock/vcl: simplify connect completion paths`

## Build Requirements

- Linux
- a VPP source tree with `libvppcom` built
- SPDK configured with `--with-vcl=<path-to-vpp-tree>`

Expected VPP layout:

- headers under `<VPP_TREE>/src`
- `libvppcom` under:
  - `<VPP_TREE>/build-root/build-vpp-native/vpp/lib/x86_64-linux-gnu`

## Build SPDK With VCL

Example:

```bash
./configure --with-vcl=/path/to/vpp --with-uring
make -j
```

This enables the `sock_vcl` module and links SPDK against `libvppcom`.

## Runtime Requirements

Each SPDK process using VCL needs:

- a reachable `libvppcom`
- a valid `VCL_CONFIG`
- a VPP instance with host stack enabled

Example:

```bash
export VCL_CONFIG=/path/to/vcl.conf
```

Typical VCL configuration:

```conf
vcl {
  app-scope-local
  app-scope-global
  app-socket-api /run/vpp/app_ns_sockets/default
  use-mq-eventfd
  event-queue-size 16384
  rx-fifo-size 16777216
  tx-fifo-size 16777216
}
```

Typical VPP startup elements:

```conf
socksvr { socket-name /run/vpp/api.sock }
session { enable use-app-socket-api event-queue-length 100000 }
tcp { tso no-tx-pacing max-rx-fifo 128m }
```

## Selecting The Backend

After starting an SPDK application with RPC:

```bash
scripts/rpc.py sock_set_default_impl -i vcl
```

This applies to:

- SPDK NVMe/TCP target
- SPDK user-space initiator tools
- any SPDK TCP user built on `spdk_sock`

## Example Target Setup

```bash
scripts/rpc.py sock_set_default_impl -i vcl
scripts/rpc.py framework_start_init
scripts/rpc.py framework_wait_init
scripts/rpc.py bdev_null_create Null0 67108864 4096
scripts/rpc.py nvmf_create_transport -t TCP
scripts/rpc.py nvmf_create_subsystem nqn.2026-03.io.spdk:null0 -a -s SPDK00000000000001
scripts/rpc.py nvmf_subsystem_add_ns nqn.2026-03.io.spdk:null0 Null0
scripts/rpc.py nvmf_subsystem_add_listener nqn.2026-03.io.spdk:null0 -t tcp -a 10.42.0.1 -s 4420
```

## Example User-Space Initiator

```bash
build/bin/spdk_nvme_identify \
  -S vcl \
  -r 'trtype:TCP adrfam:IPv4 traddr:10.42.0.1 trsvcid:4420 subnqn:nqn.2026-03.io.spdk:null0'
```

## Example Linux Kernel Initiator

```bash
sudo modprobe nvme_tcp
sudo modprobe nvme_fabrics
sudo nvme connect -t tcp -a 10.42.0.1 -s 4420 -n nqn.2026-03.io.spdk:null0 -i 1
sudo nvme list
```

## Recommended Validation Flow

For first validation, use this order:

1. `VCL <-> VCL`, `MTU 1500`
2. `VCL <-> VCL`, `MTU 9000`
3. Linux kernel NVMe/TCP initiator -> SPDK/VCL target

This narrows problems more quickly:

- backend correctness first
- then jumbo performance
- then Linux interoperability

## Practical Tuning Notes

The most important knobs found during validation were:

- thread ownership model
- DPDK RX/TX descriptor depth
- number of VPP queues
- `MTU 1500` vs `MTU 9000`
- VCL FIFO sizes
- CPU and NUMA placement

### Queue Mapping

Best results were obtained when:

- `NUM_QUEUES = VPP_WORKERS`

This avoids under-provisioning NIC queueing when VPP worker count increases.

### Jumbo Frames

For jumbo traffic, ring depth, VPP buffer size and RDMA receive mode mattered a
lot. Early validation needed deeper rings to avoid instability. With the fixed
RDMA striding-RQ buffer accounting, the current best jumbo point on this setup
uses:

- `MTU 9000`
- VPP `default data-size 10240`
- RDMA `striding-rq` enabled
- hardware checksum offload enabled
- TSO disabled for the best jumbo VCL point
- SPDK target mask `0x3` (two reactors)
- SPDK initiator `P=2`
- target RX/TX descriptors `1024`
- initiator RX/TX descriptors `2048`

### NUMA Awareness

Keep SPDK, VPP workers, and NIC queues aligned as much as possible:

- pin SPDK reactors
- pin VPP workers
- keep both close to the NIC NUMA node

## Performance Summary

The numbers below are the best points observed on the current lab setup. They
are not universal, but they show the relative positioning of the backends under
the same environment.

This is an architectural comparison, not a fixed-core-budget microbenchmark.
The current classic SPDK and VCL target runs both use a two-reactor SPDK target,
but the networking CPU is spent in different places: Linux uses kernel TCP,
softirq, NAPI and driver paths, while VCL uses VPP host-stack processing.

### Current `MTU 9000`: Classic SPDK vs SPDK + VCL

Current best validated runs use a 45-second window.

| Workload | Best classic SPDK | SPDK + VCL | VCL lead |
|---|---:|---:|---:|
| `4K randread` | `3027.66 MiB/s` (`io_uring`) | `4638.42 MiB/s` | `+53.2%` |
| `4K randwrite` | `2770.09 MiB/s` (`io_uring`) | `4972.24 MiB/s` | `+79.5%` |
| `4K randrw 70/30` | `3190.93 MiB/s` (`io_uring`) | `4605.58 MiB/s` | `+44.3%` |
| `64K randread` | `4734.80 MiB/s` (`posix`) | `5241.31 MiB/s` | `+10.7%` |
| `64K randwrite` | `3464.24 MiB/s` (`posix`) | `5065.81 MiB/s` | `+46.2%` |

Takeaway:

- `io_uring` is the strongest classic backend for `4K` in these tests
- `posix` is the strongest classic backend for `64K` in these tests
- `vcl` is ahead of the best classic backend on every listed workload
- the largest current VCL advantage is on `4K randwrite`

### Current `MTU 1500`: SPDK + VCL

Current validated VCL runs at `MTU 1500` use a 45-second window.

| Workload | SPDK + VCL |
|---|---:|
| `4K randread` | `4150.58 MiB/s` |
| `4K randwrite` | `4414.38 MiB/s` |
| `4K randrw 70/30` | `4481.91 MiB/s` |
| `64K randread` | `4337.58 MiB/s` |
| `64K randwrite` | `4444.12 MiB/s` |

Compared with the previous `MTU 1500` VCL results, the current setup improves
`4K randread` by `+8.4%` and `4K randwrite` by `+25.3%`. `64K` remains roughly
flat: `64K randread` is `-3.0%` and `64K randwrite` is `-0.2%`.

### Previous Published `MTU 9000` Baseline

The previous public comparison used `MTU 9000`, `QD64`, a 5-second window, TSO
enabled, and 2048 RX/TX descriptors.

| Workload | `vcl` | `posix` | `io_uring` |
|---|---:|---:|---:|
| `4K randread` | `1579.95 MiB/s` | `961.59 MiB/s` | `978.93 MiB/s` |
| `4K randwrite` | `1048.88 MiB/s` | `947.62 MiB/s` | `1050.92 MiB/s` |
| `64K randread` | `3284.29 MiB/s` | `2118.84 MiB/s` | `2273.67 MiB/s` |
| `64K randwrite` | `2943.88 MiB/s` | `2009.51 MiB/s` | `2085.30 MiB/s` |

Compared with that previous VCL baseline, the current tuned VCL path reaches:

- `4K randread`: `2.94x`
- `4K randwrite`: `4.74x`
- `64K randread`: `1.60x`
- `64K randwrite`: `1.72x`

The main shift is `4K randwrite`: in the previous public results VCL was
effectively tied with `io_uring`; in the current setup, VCL is `+79.5%` ahead
of the current best classic SPDK backend on that workload.

## Benchmarking Recipes

### SPDK User-Space Initiator

Example:

```bash
SPDK_WORKERS=2 build/bin/spdk_nvme_perf \
  -S vcl \
  -q 64 \
  -o 65536 \
  -w randread \
  -P 2 \
  -r 'trtype:TCP adrfam:IPv4 traddr:10.42.0.1 trsvcid:4420 subnqn:nqn.2026-03.io.spdk:null0'
```

### Linux Kernel Initiator

Example:

```bash
sudo fio \
  --name=test \
  --filename=/dev/nvme0n1 \
  --rw=randread \
  --bs=64k \
  --iodepth=64 \
  --ioengine=io_uring \
  --direct=1 \
  --time_based=1 \
  --runtime=10 \
  --group_reporting=1
```

## What To Check During Debug

Useful VPP commands:

```bash
vppctl show errors
vppctl show session verbose 2
vppctl show interface
```

Useful Linux commands:

```bash
ss -tinm
nvme list
nvme list-subsys
```

Symptoms that were important during development:

- `Old segment`
- `Duplicate ACK`
- `rcv_ooopack`
- stalled `Tx fifo`
- admin qpair keepalive timeout

## Interpretation Of The Results

This backend is no longer just a functional experiment. It provides a credible
optimization path for NVMe/TCP:

- strong small-I/O throughput
- competitive large-I/O throughput
- good scaling when worker ownership and queueing are correct
- a clean target-side architecture for raw `vppcom`

For anyone evaluating NVMe/TCP acceleration over VPP host stack, this backend
is a practical starting point.

## Related Files

- [`module/sock/vcl/vcl.c`](vcl.c)
- [`lib/nvmf/tcp.c`](../../../lib/nvmf/tcp.c)
- [`doc/vcl_sock.md`](../../../doc/vcl_sock.md)

