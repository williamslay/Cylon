# Cylon Project Guide

This guide explains how this repository is organized, how the Cylon framework fits together, and how to restart the generic CXL VM we launched during this session.

Official paper page: <https://www.usenix.org/conference/fast26/presentation/yoon>

## What Cylon Is

The FAST '26 page describes Cylon as **"a fast and extensible full-system emulator for CXL-SSDs built on FEMU"**. The page says Cylon aims to bridge the gap between closed hardware prototypes and slow software simulators, reproducing sub-microsecond cache hits and tens-of-microseconds NAND misses through a hybrid execution path that reduces hypervisor trap overheads. The page also says Cylon supports configurable caching policies, provides an application-level interface for hardware-software co-design, and was validated against a real CXL-SSD prototype.

In this repo, that framework is split across three major layers:

1. **Host/QEMU/FEMU layer** in `CylonFEMU/`: device model, CXL-SSD logic, FTL, buffer cache, launch scripts.
2. **Host kernel/KVM layer** in `CylonLinux/`: custom KVM extensions for Cylon's dual-mode memory path.
3. **Guest/experiment layer** in `tools/` and `Cylon-scripts/`: guest setup helpers and workload scripts for figures.

## Paper Design and Implementation Details

The paper's core idea is to emulate a CXL-SSD as real load/store memory instead of a block device. The guest sees a standard CXL 2.0 Type-3 device and can expose it either as a DAX region or as a CPU-less NUMA node. The emulated SSD capacity is mapped into the guest physical address space, while the CXL-SSD DRAM cache is hidden behind the device model, like a hardware-managed cache.

Cylon has three design goals:

- **Fidelity**: preserve CXL.mem byte-addressable load/store semantics, not NVMe-style block I/O semantics.
- **Efficiency**: reproduce sub-microsecond cache hits and tens-of-microseconds NAND misses without forcing every access through QEMU MMIO traps.
- **Extensibility**: expose cache policy, prefetching, observability, and application hints so researchers can study hardware-software co-design.

### Hybrid Access Path

Cylon splits each guest memory access into a fast path and a slow path.

On a cache hit, the Extended Page Table Entry (EPTE/SPTE in this repo's naming) points directly to the host physical memory backing the CXL-SSD DRAM cache. The guest load/store completes at remote-NUMA DRAM speed and does not VM-exit to QEMU.

On a cache miss, the EPTE is left in a trapping/MMIO state. The guest access causes an EPT violation, KVM routes the miss to the QEMU/FEMU side, FEMU maps the guest physical address to an SSD offset, the FTL/NAND model supplies the data and latency, and Cylon inserts the page into the DRAM cache. After fill, Cylon updates the EPTE to direct state so later accesses bypass QEMU. On eviction, the EPTE returns to trap/MMIO state; dirty pages are written back through the SSD model first.

This is the paper's Dynamic EPT Remapping (DER): Cylon changes EPTE permissions and target PFNs to switch pages between:

- **Direct state**: readable/writable, write-back memory type, HPA points at CXL-SSD DRAM cache.
- **Trap state**: permissions cleared so access causes EPT violation and enters emulation.

In this repo, the names are slightly implementation-specific:

- `CylonFEMU/hw/femu/kvm_ext.c` defines `DIRECT_MASK` and `MMIO_MASK`.
- `make_direct_spte()` builds direct mappings from `femu->hpa_base + (lpn << PAGE_SHIFT)`.
- `make_mmio_spte()` builds trapping/MMIO mappings.
- `femu_kvm_spte_clear_mmio_flag()` switches an LPN to direct state.
- `femu_kvm_spte_set_mmio_flag()` switches an LPN back to MMIO/trap state.

### Shared EPT Memory

DER still needs fast residency updates when pages are filled, evicted, or prefetched. A naive design would call a KVM ioctl for every transition and force KVM to find the target EPTE through internal page-table walks. The paper calls that too expensive for frequent cache churn.

Cylon's answer is Shared EPT Memory: pre-allocate leaf EPTE storage and make it addressable from QEMU/FEMU by logical page number. FEMU does not freely install arbitrary page-table entries; KVM validates and constrains updates so only selected fields change. The paper describes this as constant-time EPTE lookup/update plus batched/coalesced invalidation.

Repo mapping:

- `femu_kvm_set_user_memory_region()` registers the CXL-SSD memory slot with `KVM_MEMSLOT_DUAL_MODE`.
- `init_spt()` allocates shared anonymous memory for SPTE pointers and calls `KVM_GET_LINEAR_SPT`.
- `dualslot_get_sptep(lpn)` indexes directly into the shared SPTE array.
- `CylonLinux/include/linux/kvm_ext.h` defines the Cylon KVM structs/ioctls.
- `CylonLinux/virt/kvm/kvm_main.c` and `CylonLinux/arch/x86/kvm/mmu/` implement the host-kernel side.

This is why Cylon paper mode cannot run on stock Ubuntu KVM. The QEMU/FEMU code expects CylonLinux KVM to understand the dual-mode memslot and shared SPTE machinery. The paper also describes targeted invalidation with INVEPT/INVVPID and batching/coalescing so cache fills, evictions, and prefetch bursts do not require global TLB shootdowns.

### Cache Policy and Prefetch Framework

The paper emphasizes that CXL-SSD performance is dominated by the gap between DRAM-cache hits and NAND misses. Cylon therefore makes the device-side cache policy configurable rather than hardcoded.

This repo implements policy modules in `CylonFEMU/hw/femu/bbssd/`:

| Policy | File | Use |
|---|---|---|
| FIFO | `fifo.c` | Evict oldest cached page |
| LIFO | `lifo.c` | Evict most recently inserted page |
| S3FIFO | `s3fifo.c` | Segmented FIFO-style policy for skew/locality experiments |
| CLOCK | `clock.c` | Approximate recency with clock hand |

`buffer.c` is the central cache controller. It tracks buffer entries, calls the selected replacement policy, performs Next-N prefetching, and invokes KVM/SPTE switches when pages enter or leave the cache.

`run-cxlssd.sh` exposes the main knobs:

```sh
policy=2   # 1:LIFO 2:FIFO 3:S3FIFO 4:CLOCK
prf_dg=0   # Next-N prefetch degree
```

### FEMU Backend and NAND Miss Modeling

Cylon uses QEMU for the CXL Type-3/full-system VM surface and FEMU for SSD timing. FEMU supplies FTL behavior, NAND read/program/erase latency, channel/die/plane behavior, queueing, and GC interference. The paper notes that backend data is currently stored in host DRAM for speed and simplicity; capacity is therefore limited by host memory, though the architecture could swap in another backend such as SPDK/NVMe.

Repo mapping:

- `CylonFEMU/hw/femu/cxlssd/cxlssd.c` initializes CXL-SSD mode, CXL request/response rings, the priority queue, KVM dual-mode memory, and SSD state.
- `CylonFEMU/hw/femu/bbssd/ftl.c` handles CXL requests in `FEMU_CXLSSD_MODE`, performs buffer lookups, models NAND timing, and runs GC.
- `CylonFEMU/hw/femu/backend/dram.c` provides the DRAM-backed storage backend.
- `CylonFEMU/hw/femu/meson.build` links the CXL-SSD, KVM bridge, FTL, cache policy, and backend objects into QEMU/FEMU.

### Application-Level Interface and Observability

The paper also describes an application-level interface for explicit prefetch, pin, evict, policy tuning, and fine-grained statistics. The high-level purpose is cooperative caching: applications know future access patterns that a device-managed cache cannot infer. The paper describes this control plane as a shared-memory ring queue for low-latency commands/completions, with a kernel-mediated ioctl path and thin userspace library as alternate access paths.

In this artifact, the visible hooks are mostly the CXL/FEMU queues, stats, buffer policy controls, benchmark scripts, and guest helpers. Treat the paper's application-level API as a design capability to look for when extending the artifact, not as a polished end-user CLI in the root README.

## Two Modes: Cylon Artifact vs Generic CXL

There are two different things you can run from this repo.

### Cylon Artifact / Paper Mode

Use this when reproducing Cylon paper behavior.

Required pieces:

- `CylonFEMU` built with FEMU CXL-SSD support.
- Custom host kernel from `CylonLinux`, because FEMU calls Cylon-specific KVM interfaces.
- Reserved host physical memory via `memmap=<size>!<start>`.
- Prepared guest image with CXL kernel/tools.
- Guest CXL setup via `tools/setup_dev.sh system_ram|devdax`.

Why custom host kernel is needed:

- `CylonFEMU/hw/femu/kvm_ext.c` calls `KVM_GET_LINEAR_SPT` and contains the Cylon SPTE update path.
- `CylonFEMU/hw/femu/kvm_ext.c` registers memory with `KVM_MEMSLOT_DUAL_MODE`.
- `CylonLinux/include/linux/kvm_ext.h` defines those custom ioctls.
- `CylonLinux/virt/kvm/kvm_main.c` accepts the dual-mode memslot flag and handles custom ioctls.
- `CylonLinux/arch/x86/kvm/mmu/` changes MMU/SPTE behavior for dual-mode memory.

Reserved memory alone is not enough for paper mode. `memmap` supplies backend memory, but Cylon's fast path also depends on custom KVM/SPTE behavior.

### Generic QEMU CXL Mode

Use this when you only want a normal QEMU CXL Type-3 device.

Required pieces:

- Built QEMU/FEMU binary.
- Guest image.
- Host `/dev/kvm`.

Not required:

- No `CylonLinux` host kernel.
- No `memmap` reservation.
- No `-device femu,...`.
- No Cylon FTL/cache/SPTE switching.

This is useful for learning and debugging guest CXL tooling, but it does not reproduce Cylon's CXL-SSD behavior.

## Repository Layout

### Root

- `README.md`: artifact usage overview. It points to guest setup, host backend memory setup, VM launch, benchmark modes, and experiment READMEs.
- `AGENTS.md`: compact operational notes for future OpenCode sessions.
- `docs/`: setup guides and this project guide.

Important note: root `README.md` says `./run-cxlssh.sh`, but the actual CXL-SSD launch script in this tree is `CylonFEMU/femu-scripts/run-cxlssd.sh`.

### `CylonFEMU/`

This is the main QEMU/FEMU codebase. `CylonFEMU/README.rst` says it is based on QEMU 8.0.50 and FEMU v8.0.0.

High-value subdirectories:

- `hw/femu/`: FEMU device model implementation.
- `hw/femu/cxlssd/`: Cylon CXL-SSD mode.
- `hw/femu/bbssd/`: black-box SSD FTL and cache policies reused by CXL-SSD path.
- `hw/femu/backend/`: backend memory mapping.
- `hw/femu/lib/`: helper structures such as queue and priority queue code.
- `femu-scripts/`: build and run scripts.
- `include/hw/femu/`: FEMU headers.
- QEMU upstream directories such as `accel/`, `block/`, `hw/`, `target/`, `softmmu/`, `qapi/`, `util/`: mostly QEMU infrastructure.

`CylonFEMU/hw/femu/meson.build` links the Cylon-relevant objects: `kvm_ext.c`, `cxlssd/cxlssd.c`, `bbssd/ftl.c`, `bbssd/buffer.c`, `fifo.c`, `lifo.c`, `s3fifo.c`, `clock.c`, `lib/pqueue.c`, `lib/rte_ring.c`, and `backend/dram.c`.

### `CylonLinux/`

This is a full Linux 6.4.6 kernel tree with Cylon KVM changes. Treat it as kernel source, not an app directory.

Important files/directories:

- `include/linux/kvm_ext.h`: Cylon-specific KVM ioctl definitions and structs.
- `include/linux/kvm_host.h`: includes `KVM_MEMSLOT_DUAL_MODE`.
- `virt/kvm/kvm_main.c`: accepts Cylon memslot flags and routes custom ioctls.
- `arch/x86/kvm/mmu/mmu.c`: changes MMU fault handling for dual-mode memslots.
- `arch/x86/kvm/mmu/tdp_mmu.c` and `spte.c`: shadow/TDP MMU support for Cylon's SPTE path.
- `arch/x86/kvm/emulate.c`: emulation path touched by dual-mode memory behavior.
- `compile.sh`: dangerous convenience script; it runs build, installs modules, installs kernel, then reboots.

Do not run `CylonLinux/compile.sh` casually. Safer manual sequence is:

```sh
cd /home/sylei/Cylon/CylonLinux
make -j$(nproc) bzImage modules
sudo make modules_install
sudo make install
sudo update-grub
```

Then reboot manually when ready.

### `Cylon-scripts/`

This holds figure reproduction scripts.

- `eval-mlc/`: Intel MLC, Figure 2 and Figure 4. `figure4-test.sh` is the quick check; full Figure 4 is `figure4.sh`.
- `eval-mio/`: MIO, Figures 3, 6, 9, and 10. MIO requires `devdax`. Figure 9 requires VM reboot/restart between eviction policies.
- `eval-redis/`: Redis/YCSB, Figures 7, 11, and 12. Redis uses `system-ram` and examples require `ENABLE_PRELOAD=1`. Figure 11 requires VM reboot/restart between eviction policies.
- `eval-gapbs/`: GAPBS, Figure 8. `figure8-test.sh` is quick; full `figure8.sh` takes multiple hours.

Benchmark mode map from root README:

| Benchmark | Required guest CXL mode |
|---|---|
| Intel MLC | `system-ram` |
| MIO | `devdax` |
| Redis YCSB | `system-ram` |
| GAPBS | `system-ram` |

### `tools/`

Guest-side helpers.

- `setup_dev.sh`: creates a CXL region, switches `/dev/dax0.0` to `devdax`, warms pages, then optionally switches to `system-ram`.
- `cxl_warmup.c`: touches CXL/DAX pages to trigger cold EPT faults and populate EPT/SPTE state.
- `compile.sh`: builds `cxl_warmup` with `gcc -o cxl_warmup cxl_warmup.c -fopenmp`.
- `pin_binary/`: builds `pin_binary.so` for `LD_PRELOAD` placement on NUMA node 1.

`tools/setup_dev.sh` mutates guest CXL/DAX state. Run it only inside the guest after CXL device appears.

## Cylon Framework Flow

### 1. Host reserves backend memory

The CXL-SSD backend needs physically contiguous host memory. Docs recommend reserving memory with `memmap=<size>!<start>` in GRUB. The start address must match a real target NUMA node range from `lsmem --output-all` or `/proc/iomem`.

This reserved region is used as backend storage/memory for the FEMU CXL-SSD device. The host OS should not allocate it for normal use.

### 2. QEMU/FEMU starts CXL-SSD device

`run-cxlssd.sh` launches QEMU with:

- `-machine type=q35,accel=kvm,nvdimm=on,cxl=on`
- `-device femu,...,femu_mode=6,...`
- CXL root topology: `pxb-cxl`, `cxl-rp`, `cxl-type3`
- `-M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=<ssd_size>M`

`run-cxlssd.sh` currently supports NAND geometry only for:

- `98304` MB
- `49152` MB

The same script also disables NUMA balancing and transparent huge pages:

```sh
echo 0 | sudo tee /proc/sys/kernel/numa_balancing
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled
```

### 3. CXL-SSD mode initializes FEMU structures

`cxlssd_init()` in `hw/femu/cxlssd/cxlssd.c` initializes:

- controller identity string,
- CXL request ring `n->cxl_req`,
- CXL response ring `n->cxl_resp`,
- priority queue `n->cxl_pq`,
- custom KVM user memory region via `femu_kvm_set_user_memory_region(n)`,
- SSD state via `ssd_init(n)`.

### 4. Custom KVM/SPTE path enables fast direct access

`hw/femu/kvm_ext.c` is the bridge between FEMU and CylonLinux KVM changes.

Key operations:

- `femu_kvm_set_user_memory_region()` registers a memslot with `KVM_MEMSLOT_DUAL_MODE`.
- `init_spt()` calls `KVM_GET_LINEAR_SPT` to get/manage shadow page table state.
- `femu_kvm_spte_clear_mmio_flag()` changes an LPN to direct mapping.
- `femu_kvm_spte_set_mmio_flag()` changes an LPN back to MMIO/emulated handling.

This is why Cylon paper mode needs the host CylonLinux kernel.

### 5. Buffer cache controls direct vs emulated memory

`hw/femu/bbssd/buffer.c` connects cache state to KVM mapping state:

- `direct_mr_add()` calls `femu_kvm_spte_clear_mmio_flag()`.
- `direct_mr_del()` calls `femu_kvm_spte_set_mmio_flag()`.
- `buffer_insert_entry()` inserts cache entries and handles Next-N prefetching.
- `buffer_clear()` flushes entries and removes direct mappings.

Replacement policy implementations live beside it:

- `fifo.c`
- `lifo.c`
- `s3fifo.c`
- `clock.c`

### 6. FTL thread models NAND misses

In `hw/femu/bbssd/ftl.c`, the CXL request path handles reads/writes in `FEMU_CXLSSD_MODE`:

- dequeue CXL request from `ssd->cxl_req`,
- compute logical page number,
- lookup buffer entry,
- on hit: enqueue response immediately and update hit counters,
- on miss: map/fetch/write NAND page, add NAND latency, enqueue response, insert/prefetch buffer entry,
- run GC when thresholds require it.

This is the key split described by the paper page: cache hits should avoid expensive emulation path; misses fall back to NAND/FTL timing.

### 7. Guest config exposes CXL memory

Inside the guest, `tools/setup_dev.sh` does:

```sh
cxl create-region -m -t ram -d decoder0.0 -w 1 -g 4096 mem0
daxctl reconfigure-device --mode=devdax --force dax0.0
./femu-cxl-test/cxl_warmup
```

Then it either keeps `devdax` or switches to `system-ram` depending on argument.

Use `system_ram` for unmodified memory workloads. Use `devdax` for applications that explicitly mmap `/dev/dax*`.

## Build and Run Notes

### Build FEMU/QEMU

```sh
cd /home/sylei/Cylon/CylonFEMU
mkdir -p build-femu
cd build-femu
cp ../femu-scripts/femu-copy-scripts.sh .
./femu-copy-scripts.sh .
sudo ./pkgdep.sh
./femu-compile.sh
```

Successful build creates:

```sh
/home/sylei/Cylon/CylonFEMU/build-femu/x86_64-softmmu/qemu-system-x86_64
```

### Launch Cylon CXL-SSD Paper Mode

Only after host kernel, backend memory, and guest image are prepared:

```sh
cd /home/sylei/Cylon/CylonFEMU/build-femu
./run-cxlssd.sh 49152
```

or for the 96 GB geometry:

```sh
./run-cxlssd.sh 98304
```

After launch, in a second terminal:

```sh
cd /home/sylei/Cylon/CylonFEMU/build-femu
./pin.sh
```

`pin.sh` expects `qmp-sock` and `log` from a running VM. Its CPU IDs are hardcoded and may need editing for your host NUMA layout.

### Launch Generic QEMU CXL Mode

This is the generic CXL VM launched during this session. It does not require CylonLinux host kernel.

```sh
cd /home/sylei/Cylon/CylonFEMU/build-femu

./x86_64-softmmu/qemu-system-x86_64 \
  -name generic-cxl-vm \
  -machine type=q35,accel=kvm,cxl=on \
  -enable-kvm \
  -cpu host \
  -smp 8 \
  -m 8G,maxmem=16G,slots=8 \
  -object memory-backend-ram,id=cxl-mem0,size=4G,share=on \
  -device virtio-scsi-pci,id=scsi0 \
  -device scsi-hd,drive=hd0 \
  -drive file=/home/sylei/images/ubuntu22.qcow2,if=none,format=qcow2,id=hd0 \
  -device pxb-cxl,bus_nr=12,bus=pcie.0,id=cxl.1 \
  -device cxl-rp,port=0,bus=cxl.1,id=root_port13,chassis=0,slot=2 \
  -device cxl-type3,bus=root_port13,volatile-memdev=cxl-mem0,id=cxl-pmem0 \
  -M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=4G \
  -net user,hostfwd=tcp::2222-:22 \
  -net nic,model=e1000 \
  -nographic \
  -qmp unix:./qmp-generic-cxl.sock,server,nowait 2>&1 | tee generic-cxl.log
```

Important detail: for this QEMU build, `cxl-type3,memdev=...` failed because persistent devices require an `lsa` property. `volatile-memdev=cxl-mem0` works for generic volatile CXL memory.

## Restart the Generic CXL VM

Current session name used here: `generic-cxl`.

### Attach to console

```sh
tmux attach -t generic-cxl
```

Detach without stopping VM:

```text
Ctrl-b d
```

### Stop VM

Graceful from guest, if logged in:

```sh
sudo poweroff
```

Hard stop from host:

```sh
tmux kill-session -t generic-cxl
```

Clean stale launch artifacts if needed:

```sh
cd /home/sylei/Cylon/CylonFEMU/build-femu
rm -f qmp-generic-cxl.sock generic-cxl.log
```

### Restart VM

```sh
cd /home/sylei/Cylon/CylonFEMU/build-femu
rm -f qmp-generic-cxl.sock generic-cxl.log

tmux new-session -d -s generic-cxl 'cd /home/sylei/Cylon/CylonFEMU/build-femu && ./x86_64-softmmu/qemu-system-x86_64 -name generic-cxl-vm -machine type=q35,accel=kvm,cxl=on -enable-kvm -cpu host -smp 8 -m 8G,maxmem=16G,slots=8 -object memory-backend-ram,id=cxl-mem0,size=4G,share=on -device virtio-scsi-pci,id=scsi0 -device scsi-hd,drive=hd0 -drive file=/home/sylei/images/ubuntu22.qcow2,if=none,format=qcow2,id=hd0 -device pxb-cxl,bus_nr=12,bus=pcie.0,id=cxl.1 -device cxl-rp,port=0,bus=cxl.1,id=root_port13,chassis=0,slot=2 -device cxl-type3,bus=root_port13,volatile-memdev=cxl-mem0,id=cxl-pmem0 -M cxl-fmw.0.targets.0=cxl.1,cxl-fmw.0.size=4G -net user,hostfwd=tcp::2222-:22 -net nic,model=e1000 -nographic -qmp unix:./qmp-generic-cxl.sock,server,nowait 2>&1 | tee generic-cxl.log'
```

Verify:

```sh
pgrep -af 'qemu-system-x86_64.*generic-cxl-vm'
test -S qmp-generic-cxl.sock && echo qmp-ok
ss -ltn sport = :2222
```

### Login Notes

The repo does not record the actual guest password. `docs/guest-image-setup.md` says the root password is set during image preparation with:

```sh
sudo virt-customize -a ubuntu22.qcow2 --root-password password:[PASSWORD]
```

If the password is unknown, stop the VM first, then reset it offline:

```sh
tmux kill-session -t generic-cxl
sudo virt-customize -a /home/sylei/images/ubuntu22.qcow2 --root-password password:root
```

Then restart and log in as:

```text
login: root
password: root
```

Do not run `virt-customize` while the VM is running.

## Setup and Verification Checklist

### Host preflight for Cylon paper mode

```sh
uname -r
cat /proc/cmdline
lsmem --output-all
ls -l /dev/cmahog /dev/pmem* /dev/cxl 2>/dev/null || true
test -x /home/sylei/Cylon/CylonFEMU/build-femu/x86_64-softmmu/qemu-system-x86_64
test -f /home/sylei/images/ubuntu22.qcow2
```

You want:

- Cylon-capable host kernel, not generic Ubuntu if running paper mode.
- `memmap=` present for reserved backend memory.
- Backend device/config matching `run-cxlssd.sh`.
- Built QEMU binary.
- Guest image.

### Guest CXL checks

Inside guest:

```sh
cxl list
daxctl list
numactl -H
```

For `system-ram`, `numactl -H` should show a CPU-less NUMA node after `daxctl reconfigure-device --mode=system-ram`. For `devdax`, `/dev/dax0.0` should exist and applications must explicitly use it.

## Common Pitfalls

- Root README launch command has a likely typo: use `run-cxlssd.sh`, not `run-cxlssh.sh`.
- `run-cxlssd.sh` currently hardcodes `backend_dev=/dev/cmahog` and `hpa_base=0xae80000000`; these must match your host.
- `CylonLinux/compile.sh` automatically reboots. Prefer manual build/install commands.
- `pin.sh` hardcodes CPU IDs and needs `qmp-sock` plus `log` from an already running VM.
- `tools/setup_dev.sh` changes guest CXL/DAX state and uses `--force`; run only when you are ready to mutate guest device state.
- MIO requires `devdax`; Redis/MLC/GAPBS require `system-ram`.
- Figure 9 (MIO) and Figure 11 (Redis) require VM restart/reboot between eviction policies.

## Source Index

- Official paper page: <https://www.usenix.org/conference/fast26/presentation/yoon>
- Root overview: `README.md`
- Host memory setup: `docs/backend-memory-setup.md`
- Guest image setup: `docs/guest-image-setup.md`
- FEMU overview: `CylonFEMU/README.md`, `CylonFEMU/README.rst`
- CXL-SSD entrypoint: `CylonFEMU/hw/femu/cxlssd/cxlssd.c`
- KVM bridge: `CylonFEMU/hw/femu/kvm_ext.c`
- Buffer policies and FTL: `CylonFEMU/hw/femu/bbssd/`
- Kernel KVM extensions: `CylonLinux/include/linux/kvm_ext.h`, `CylonLinux/virt/kvm/kvm_main.c`, `CylonLinux/arch/x86/kvm/mmu/`
- Experiment scripts: `Cylon-scripts/eval-*`
- Guest helpers: `tools/`
