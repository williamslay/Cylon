# AGENTS.md

## Repo Shape
- Artifact repo, not normal app repo: `CylonFEMU/` is QEMU 8.0.50 + FEMU CXL-SSD work, `CylonLinux/` is custom Linux/KVM support, `Cylon-scripts/` reproduces paper figures, `tools/` holds guest helpers.
- No root build/test/lint runner exists. Validate inside touched component only; many scripts need sudo, KVM, a prepared VM image, CXL tools, and real NUMA/memmap setup.
- `CylonLinux/` vendors a full kernel tree. Avoid broad searches/formatting there unless task is kernel-specific.

## High-Risk Commands
- Do not run `CylonLinux/compile.sh` casually: it does `sudo make -j 16 && sudo make modules_install -j 16 && sudo make install && sudo reboot now`.
- Host memory setup edits GRUB with `memmap=<size>!<start>` and reboots; choose `<start>` from target NUMA physical ranges (`docs/backend-memory-setup.md`).
- Guest device setup mutates CXL/DAX state: `tools/setup_dev.sh [system_ram|devdax]` runs `cxl create-region`, switches `/dev/dax0.0`, and warms pages via `./femu-cxl-test/cxl_warmup`.
- `CylonFEMU/femu-scripts/run-cxlssd.sh` disables NUMA balancing/THP with sudo, expects `~/images/ubuntu22.qcow2`, writes `qmp-sock` and `log`, and currently uses `backend_dev=/dev/cmahog`, `hpa_base=0xae80000000`.

## FEMU Build/Run
- Build from a build dir beside `CylonFEMU/femu-scripts/`, e.g. `cd CylonFEMU && mkdir -p build-femu && cd build-femu`; then copy scripts with `cp ../femu-scripts/femu-copy-scripts.sh . && ./femu-copy-scripts.sh .`.
- Install FEMU deps only on Debian/Ubuntu: `sudo ./pkgdep.sh`. Compile with `./femu-compile.sh`; it runs `make clean`, configures `x86_64-softmmu` with KVM/slirp and AVX flags, then `make -j $NRCPUS` where `NRCPUS` is counted from `/proc/cpuinfo`.
- Launch CXL-SSD VM from build dir with `./run-cxlssd.sh <ssd_size_mb>`; script only initializes NAND geometry for `98304` and `49152` MB.
- Run `./pin.sh` only after VM launch produced `qmp-sock` and `log`; CPU IDs are hardcoded (`2..9` vCPUs, poller CPUs from `29` down) and must match host NUMA placement.

## Code Entrypoints
- CXL-SSD device logic starts in `CylonFEMU/hw/femu/cxlssd/cxlssd.c`; it initializes KVM dual-mode memory, SSD state, CXL request/response rings, and FEMU mode 6.
- KVM/SPTE glue is split between `CylonFEMU/hw/femu/kvm_ext.c` and custom kernel changes summarized in `CylonLinux/change` (`KVM_MEMSLOT_DUAL_MODE`, `KVM_GET_LINEAR_SPT`, `KVM_SET_SPTE_FLAG`).
- Buffer/FTL policy code lives in `CylonFEMU/hw/femu/bbssd/`: `buffer.c`, `ftl.c`, `fifo.c`, `lifo.c`, `s3fifo.c`, `clock.c`. `hw/femu/meson.build` links these plus `cxlssd/cxlssd.c`.
- Guest helpers: build warmup with `cd tools && gcc -o cxl_warmup cxl_warmup.c -fopenmp`; `tools/pin_binary/build.sh [debug]` builds `pin_binary.so` with `-ldl -lnuma` for `LD_PRELOAD` placement on NUMA node 1.

## Experiments
- Mode matters: Intel MLC, Redis YCSB, and GAPBS need CXL exposed as system RAM (`tools/setup_dev.sh system_ram`, daxctl mode `system-ram`); MIO needs `devdax`.
- Redis scripts require `ENABLE_PRELOAD=1`; examples use `RUN_TAG=... ./run-figure*.sh` from `Cylon-scripts/eval-redis/`.
- MIO figure 9 and Redis figure 11 require VM reboot/restart between eviction policies.
- Prefer `*-test.sh` scripts for quick checks (`eval-mlc/figure4-test.sh`, `eval-gapbs/figure8-test.sh`); full GAPBS figure 8 takes multiple hours.

## Style
- In `CylonFEMU/`, follow `.editorconfig`: C/H use 4-space indents, shell uses 4 spaces, Makefiles use tabs width 8.
- In `CylonLinux/`, preserve kernel style and avoid repo-wide reformatting; changes usually need focused kernel build/test guidance from the touched subsystem.
