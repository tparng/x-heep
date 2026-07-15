# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

X-HEEP (eXtensible Heterogeneous Energy-Efficient Platform) is a configurable RISC-V microcontroller
described in SystemVerilog, plus its software stack (baremetal + FreeRTOS). Hardware is described in
templated SystemVerilog that is regenerated from a Python configuration; software is C compiled with a
RISC-V GCC/Clang toolchain and CMake. The design is built/simulated/synthesized via FuseSoC.

## Environment setup

Two supported paths, see `docs/source/GettingStarted/Setup.md`:
- **Docker (recommended)**: `make -C util/docker docker-pull && make -C util/docker docker-run`. Inside
  the container, use shortcuts `init_corev` / `init_gcc` / `init_clang` to select the RISC-V toolchain
  before `make app`.
- **Manual**: install OS deps (see Setup.md), then `make conda` or let `Makefile.venv` auto-create a
  Python venv (`.venv/`) the first time `make` is run — this is the default toolchain path used by the
  top-level `Makefile` when `CONDA_DEFAULT_ENV` isn't set.

## Core workflow

The standard flow is: **generate → build → run**. Hardware and register maps are generated from Python
config (`configs/general.hjson` + `configs/general.py` by default), not hand-written.

```bash
make mcu-gen                                   # regenerate all *.sv.tpl -> *.sv, runs FuseSoC reg-gen, verible format, black format
make app PROJECT=hello_world                   # compile/link a software application (CMake, under the hood)
make verilator-build                           # build the Verilator C++ simulation model (needs mcu-gen first)
make verilator-run                             # run the last-built app on the last-built Verilator model
make verilator-run-app                         # = app + run in one step (rebuilds firmware, reuses sim model)
make verilator-run-helloworld                  # mcu-gen + verilator-build + hello_world app + run, all-in-one
```

Key `make mcu-gen` parameters (see header of `Makefile` for the full list):
`CPU=cv32e20(default)|cv32e40p|cv32e40x|cv32e40px`, `BUS=onetoM(default)|NtoM`,
`MEMORY_BANKS=`, `MEMORY_BANKS_IL=`, `X_HEEP_CFG=configs/general.hjson`, `PYTHON_X_HEEP_CFG=`.

Key `make app` parameters: `PROJECT=<folder under sw/applications>`,
`TARGET=sim(default)|systemc|pynq-z2|nexys-a7-100t|genesys2|aup-zu3|zcu102|zcu104`,
`LINKER=on_chip(default)|flash_load|flash_exec`, `COMPILER=gcc(default)|clang`,
`COMPILER_PREFIX=riscv32-corev-(default)|riscv32-unknown-`, `ARCH=rv32imc_zicsr(default)`.

Other simulators: `questasim-build`/`questasim-run*`, `vcs-build`, `vcs-ams-build`, `xcelium-build`
(these need licensed EDA tools, not available in the Docker image). FPGA: `vivado-fpga`,
`vivado-fpga-pgm`. ASIC: `asic`, `openroad-sky130`.

`make app-list` lists available applications (`sw/applications/*`). `make clean` removes generated
hardware files and the sw build dir; `make clean-all` is currently equivalent.

## Testing

```bash
make test                                      # mcu-gen with configs/ci.hjson, then runs test_apps.py and test_peripherals.py
make test TEST_FLAGS=--compile-only            # skip RTL simulation, just verify all apps build
python3 test/test_apps/test_apps.py --table    # run directly; see BLACKLIST/WHITELIST globals in the script for scoping
make compare-mcu-gen                           # diff mcu-gen output between current branch and main (test/test_x_heep_gen/compare_mcu_gen.py)
```

CI (`.github/workflows/ci.yml`) additionally checks: generated-file consistency (`make mcu-gen` must
produce no diff — see `util/git-diff.py`), peripheral-generation tests
(`test/test_x_heep_gen/test_peripherals.py`), vendored-IP freshness (`util/vendor.py` + `git-diff.py`),
and Python formatting (`black`, via `psf/black` action).

Formatting (also run automatically at the end of `make mcu-gen`):
```bash
make verible          # format generated SystemVerilog (util/format-verible)
make format-python     # black-format util/xheep_gen, util/periph_structs_gen, util/waiver-gen.py, util/c_gen.py, test/test_x_heep_gen, test/test_apps, configs
```

`VerifHeep` (`test/verifheep/`) is a separate signal-tracing/verification library used for functional
verification of specific IPs (see `docs/source/Testing/Testing.md` and examples in
`test/verifheep/examples/`).

## Architecture

### Hardware generation pipeline

Most top-level RTL is **not hand-written** — it's rendered from Jinja-style `.tpl` templates by
`util/xheep_gen/mcu_gen.py`, driven by a config (`configs/*.hjson` for legacy configs, or increasingly a
Python config module like `configs/general.py` built from `util/xheep_gen`'s object model:
`xheep.py`, `cpu/`, `bus_type.py`, `memory_ss/`, `peripherals/`, `pads/`, `cv_x_if.py`).

Templated files live throughout `hw/core-v-mini-mcu/*.sv.tpl` (bus, xbar, cpu/memory/peripheral
subsystems, `include/core_v_mini_mcu_pkg.sv.tpl`) and `hw/system/*.sv.tpl` (top-level `x_heep_system`,
pad ring). **Never hand-edit a generated `.sv` file whose name matches a `.sv.tpl`** — edit the
template (or the Python config) and re-run `make mcu-gen`. `make mcu-gen` also invokes FuseSoC's
register-generation step and then formats output (verible + black).

After `mcu-gen`, FuseSoC (`.core` files) is used for every downstream flow: building simulation models,
FPGA bitstreams, and ASIC synthesis. The root `core-v-mini-mcu.core` is the main core description,
declaring dependencies on vendored IP cores (`x-heep:ip:*`, `pulp-platform.org::*`, `lowrisc:ip:*`,
`openhwgroup.org:ip:*`) and listing the generated RTL files. `.core` files also exist throughout `hw/`
for individual IPs.

### Vendored dependencies

Third-party IP is not vendored via git submodules but via OpenTitan's `util/vendor.py` tool, driven by
`<org>_<repo>.vendor.hjson` description files (mainly in `hw/vendor/`, some in `sw/vendor/` and
`util/`) with matching `.lock.hjson` files pinning the fetched revision. Patches applied on top of
vendored sources live in `hw/vendor/patches/`. Regenerate with
`util/vendor.py --update hw/vendor/<org>_<repo>.vendor.hjson`, or `make vendor-update` for all of them.
CI enforces that vendored trees match their `.vendor.hjson`/lock files.

### Directory map

- `hw/core-v-mini-mcu/` — the CPU/bus/memory/peripheral subsystems that make up the `core_v_mini_mcu`
  FuseSoC core (mostly template-generated).
- `hw/system/` — the top-level `x_heep_system` wrapping `core_v_mini_mcu` plus pads.
- `hw/ip/` — X-HEEP-authored IP blocks (DMA subsystem, boot ROM, power manager, fast interrupt
  controller, OBI fifo/spimemio, PDM2PCM, SoC control, serial link wrapper).
- `hw/ip_examples/` — reference/example accelerators and peripherals demonstrating extension patterns
  (simple_accelerator, im2col_spc, fpu_ss_wrapper, i2s_microphone, etc.).
- `hw/vendor/` — vendored third-party IP (PULP-Platform, OpenHW Group cores CV32E20/40P/40X/40PX,
  lowRISC/OpenTitan blocks) — see "Vendored dependencies" above.
- `hw/fpga/`, `hw/asic/` — board wrappers/constraints and ASIC implementation files (incl. sky130).
- `sw/applications/` — example/test applications, one per directory (`main.c` + sources); this is the
  primary place new firmware examples go. `make app-list` enumerates them.
- `sw/device/` — HAL/SDK: `lib/{base,crt,drivers,runtime,sdk}`, `bsp/`, `target/<board>/` (per-target
  startup/linker glue for sim, systemc, and each supported FPGA board), CMake build system
  (`sw/cmake`, `sw/CMakeLists.txt`).
- `sw/linker/` — linker scripts, selected via `LINKER=on_chip|flash_load|flash_exec`.
- `sw/freertos/` — FreeRTOS integration.
- `configs/` — MCU configuration: `general.hjson`/`general.py` (default), `ci.hjson`/`ci.py` (used by
  CI, adds more peripherals for coverage), `minimal.hjson`, others per feature (xif, interleaved
  memory, etc.), plus `pad_cfg.py` for pad configuration.
- `util/xheep_gen/` — the Python MCU generator (`mcu_gen.py` + config object model).
- `util/periph_structs_gen/` — generates peripheral C structs/headers from the config.
- `util/vendor.py`, `util/check-vendor.py` — vendoring tool and CI check.
- `util/docker/` — Docker image build + `docker-pull`/`docker-run` targets, `env.sh` toolchain shortcuts.
- `util/profile/` — RV profiling (`make profile`), produces a flamegraph.
- `util/area-plot/` — post-synthesis area report visualization (`make area-plot`).
- `test/test_apps/` — `test_apps.py` compiles/simulates every application (used by `make test`).
- `test/test_x_heep_gen/` — tests for the mcu-gen config/peripheral pipeline, plus `compare_mcu_gen.py`.
- `test/verifheep/` — signal-tracing verification library and examples for functional IP verification.
- `tb/` — testbench SystemVerilog and OpenOCD configs for on-chip debug.
- `ides/` — IDE integration files.

### Extending X-HEEP with custom hardware/software

X-HEEP is typically consumed as a vendored dependency inside a larger project (see
`docs/source/Extending/eXtendingHW.md` / `eXtendingSW.md`), rather than modified in place, when adding:
- A **CV-X-IF coprocessor**: vendor X-HEEP into your own top-level repo, copy
  `hw/system/x_heep_system.sv` as your new top module, wire the CV-X-IF interface to your coprocessor
  (see `tb/testharness.sv` for reference), build with `make mcu-gen CPU=cv32e40px`.
- A **bus-attached accelerator**: similar pattern; see `hw/ip_examples/` for worked examples.
- New software living outside X-HEEP's own tree can still use its CMake/Makefile flow via the `SOURCE=`
  variable pointing at an external `sw/` directory that mirrors X-HEEP's `applications/device/linker`
  layout.

When working *inside this repo itself* (not as a vendored dependency), new peripherals are added
through the Python config object model in `util/xheep_gen/peripherals/`, not by hand-writing RTL glue.
