# XRSight-RTOS

XRSight-RTOS brings an ILLIXR-based extended-reality workload to Zephyr RTOS and heterogeneous RISC-V SoCs. It connects sensor traces, pose estimation, rendering schedules, and eye inference to the processor, memory system, and accelerators executing it in cycle-accurate simulation. This is a major revision to [XRSight](https://github.com/ucb-bar/xrsight/tree/xrsight-1.0), which combines [ILLIXR](https://illixr.github.io/ILLIXR/), [Chipyard](https://chipyard.readthedocs.io/), and [FireSim](https://docs.fires.im/) for XR hardware/software co-design. See the [original IISWC 2025 paper](https://ieeexplore.ieee.org/document/11242088) for the project background.

The runtime boots directly as a Zephyr ELF. Tested systems include single-, dual-, and quad-core Rocket, Saturn vector units for OpenBLAS operations, FP32 Gemmini for selected OpenBLAS operations, and INT8 Gemmini for RITnet eye inference. 

*Other implementation notes:*
The current graphics stages model asynchronous GPU latency and publish **dummy image descriptors**. Timewarp computes a CPU rotational correction, but neither stage runs shaders or produces real rendered pixels. We plan to extend this to GPU memory behavior modeling or the original XRSight blackbox GPU model. Additionally, RITNet returns an image-space foreground centroid, not a calibrated gaze vector and timewarp records that result without changing its pixels. No desktop renderer or host GPU worker is required for this flow.

![XRSight-RTOS data flow and available backends](docs/diagrams/eye-tracking-flow.png)

## Contents

1. [Updates from XRSight 1.0](#updates-from-xrsight-10)
2. [Included Plugins](#included-plugins)
3. [Chipyard Setup](#chipyard-setup)
4. [Building the ELF](#building-the-elf)
5. [Running on FireSim](#running-on-firesim)
6. [Outputs and Analysis](#outputs-and-analysis)
7. [Contributors](#contributors)

## Updates from XRSight 1.0

### Embedded RTOS with verified multicore execution

[Zephyr](https://www.zephyrproject.org/) is a configurable real-time operating system for embedded devices. It provides preemptible threads, priorities, timers, mutexes, message queues, drivers, and symmetric multiprocessing (SMP) without requiring a Linux userspace. The application and selected kernel components link into one firmware image. [Chipyard's Zephyr guide](https://chipyard.readthedocs.io/en/latest/software/zephyr/) explains its kernel configuration, device trees, and HTIF integration.

Moving the ILLIXR-based workload from Ubuntu to Zephyr gives us a smaller OS workload and an embedded execution environment closer to a production AR/VR device. It avoids simulating a full Linux boot and userspace services. This makes simulation substantially more practical.

Single-, dual-, and quad-core execution has been validated with Zephyr threads. Generally, plugins that are based on the ILLIXR threadloop instantiate their own threads, while those that offer services (e.g. pose_prediction) are executed on the calling thread. 

### Independent work on a shared target clock

The earlier XRSight deployment used an IMU-sample-aligned global simulation tick to coordinate work around that deployment's Linux/Chipyard concurrency limitations. This was a workaround in that flow that addressed limitations in Linux-Chipyard integration. 

Here, Zephyr schedules independent workers against a shared target clock:

- IMU replay publishes every included sample in timestamp order to two independent queues. 
- Camera replay selects stereo images due by the current target clock, skipping superseded replay opportunities are skipped.
- OpenVINS consumes its camera and IMU queues and publishes a latest VIO baseline. 
- The IMU Integrator (GTSAM) independently consumes IMUs and propagates that baseline.
- Pose prediction is an on-demand service executed in the requesting render/timewarp thread. It reads a coherent integration snapshot and predicts a fast pose for the requested display timestamp.
- Eye inference independently consumes the latest eye image and publishes its latest completed result. 
- Render starts 1 ms after an absolute 120 Hz vsync boundary and targets the following boundary. 
- Timewarp independently starts 2 ms before its upcoming boundary: 1 ms GPU delay plus 1 ms margin. It can warp the same completed frame more than once.

In standalone mode, the render delay is 6.944445 ms and the timewarp delay is 1 ms, and expired opportunities are skipped without catch-up bursts. The main consumer models presentation at each vsync using timestamped completion history. Note that this is modeled presentation, without any real display. See [GPU scheduling](docs/gpu-pipeline.md), [IMU transport](docs/imu-value-transport.md), and [asynchronous eye tracking](docs/eye-tracking.rst).

### Clock settings

| Clock or quantity | Current setting | Meaning |
|---|---|---|
| Generated target CPU/bus clocks | 500 MHz | Chipyard configurations, including `WithPeripheryBusFrequency(500.0)`; these configurations keep the relevant target clocks at the same rate. |
| Generated CLINT timebase | 500 kHz | `WithTimebase(BigInt(500000))`; one `mtime` increment per 1,000 target CPU cycles, or 2 µs under the generated 500 MHz interpretation. |
| Firmware timer declaration | 1 MHz | `CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC=1000000` in `config/rocket_1ghz.conf`; Zephyr interprets an `mtime` increment as 1 µs. |
| Modeled CPU frequency | 1 GHz | `ILLIXR_CORE_HZ=1000000000`; the same 1,000:1 CPU/timer ratio is interpreted at twice the generated frequency. |
| Zephyr tick rate | 10 kHz | `CONFIG_SYS_CLOCK_TICKS_PER_SEC=10000`: 100 µs timeout units. Tickless operation does not require an interrupt on every tick. |
| Physical U250 clock request | 30 MHz | FPGA implementation constraint, with `NORETIMING`. It is distinct from simulated CPU frequency and achieved simulation throughput. |
| Host elapsed time | Measured wall time | Programming, loading, simulation, export, and analysis take real host time. This is not the application clock. |

From the above settings we can derive how the modeled CPU frequency is 1 GHz: with a peripheral bus frequency of 500 MHz and CLINT timer timebase as 500 kHz, we know that mtime/CLINT increments once per 1000 peripheral-bus cycles. The Zephyr config CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC specifies that 1 million hardware timer (mtime) increments corresponds to 1 second, yielding 1,000,000 x 1000 = 1 billion cycles/second.

The factor-two interpretation models a 1 GHz operating point on an unchanged bitstream built at 500 MHz target clock; it does not demonstrate physical 1 GHz silicon timing closure. Note that changing the modeled CPU frequency thus requires changing both the periphery bus frequency as well as the timebase. Changing only the CPU label without the timer interpretation would be inconsistent. Spike retains its separate 10 MHz timer configuration and provides functional evidence, not cycle-accurate hardware performance. See [clock experiments](docs/clock-experiments.md) and [baseline settings](docs/current-baseline.md).

## Included plugins

The complete profile contains these nine components, based on the original ILLIXR runtime. The **counterpart** links describe the related desktop component with additional documentation on the intention and algorithm, but are not necessarily the same implementation.

| Plugin | Role | Documentation |
|---|---|---|
| `offline_imu` | Replays embedded angular-velocity/acceleration samples at their dataset timestamps into independent VIO and integration queues. Every selected IMU must reach both consumers. | [Upstream](https://illixr.github.io/ILLIXR/latest/illixr_plugins/#offline_imu); [RTOS transport](docs/imu-value-transport.md) |
| `offline_cam` | Decodes embedded stereo PNG pairs and publishes due grayscale images to VIO. Expired opportunities and full-queue drops are counted separately. | [Upstream](https://illixr.github.io/ILLIXR/latest/illixr_plugins/#offline_cam) |
| `openvins` | Consumes camera/IMU input, estimates visual-inertial pose, and publishes the latest pose and integration baseline. It is the principal configurable BLAS workload. | [Upstream `open_vins`](https://illixr.github.io/ILLIXR/latest/illixr_plugins/#open_vins); [backend integration](docs/gemmini-openblas.md) |
| `imu_integrator` | Propagates the latest VIO baseline through retained IMUs, publishing current pose and coherent state for prediction. | [Related upstream integrator](https://illixr.github.io/ILLIXR/latest/illixr_plugins/#rk4_integrator); [RTOS transport](docs/imu-value-transport.md) |
| `pose_prediction` | Provides caller-thread RK4 prediction for a requested timestamp, including horizon and stale-state checks. It is an on-demand service, not another periodic publishing thread. | [Upstream service API](https://illixr.github.io/ILLIXR/latest/api/classILLIXR_1_1data__format_1_1pose__prediction/); [RTOS prediction](docs/gpu-pipeline.md) |
| `eye_tracking` | Runs asynchronous RITNet on hart-0 INT8 Gemmini, reading the latest eye image and retaining each completed eye-position result for nonblocking consumers. | [RTOS implementation](docs/eye-tracking.rst); no corresponding official upstream plugin page found |
| `offline_eye` | Republishes one embedded 240×160 eye sample at 120 Hz with timestamp/sequence metadata; pending inference notifications coalesce. | [RTOS implementation](docs/eye-tracking.rst); RTOS-specific image source |
| `render_loop` | Requests a pose for the next display boundary and publishes an immutable dummy stereo-frame descriptor after its simulated GPU delay. | [Upstream scheduling counterpart: `gldemo`](https://illixr.github.io/ILLIXR/latest/plugin_README/README_gldemo/); [RTOS model](docs/gpu-pipeline.md) |
| `timewarp` | Independently snapshots the latest completed frame, obtains a fresh pose and latest eye result, computes rotational correction, and models GPU completion. Frames and eye results can be reused. | [Upstream scheduling counterpart: `timewarp_gl`](https://illixr.github.io/ILLIXR/latest/plugin_README/README_timewarp_gl/); [RTOS model](docs/gpu-pipeline.md) |


## Chipyard setup

### Host prerequisites and workspace

This flow is tested on a Linux x86-64 host. 

```bash
git clone https://github.com/pcg108/illixr-rtos.git xrsight-rtos
cd xrsight-rtos
export XRSIGHT_ROOT="$PWD"
export XRSIGHT_WORK="$HOME/xrsight-work"
export CHIPYARD_DIR="$XRSIGHT_WORK/chipyard"
export EUROC_MAV0="$XRSIGHT_WORK/data/V1_02_medium/mav0"
mkdir -p "$XRSIGHT_WORK"
python3 -m venv "$XRSIGHT_WORK/venv"
export XRSIGHT_PYTHON="$XRSIGHT_WORK/venv/bin/python"
"$XRSIGHT_PYTHON" -m pip install --upgrade pip
```

### Pinned checkout and compiler

Complete this step before building `openblas_rvv` or `openblas_gemmini_fp32` archives. The tested RVV C compiler is GCC 13.2.0 supplied by Chipyard; the archive builder checks that exact version because its RVV compatibility workaround is compiler-specific. The Zephyr SDK continues to compile the application/kernel and supply Newlib, C++ libraries, and the ABI-compatible headers used by the RVV library. No FireSim manager, Vivado, or FPGA provisioning is needed merely to compile the RVV archive.

Eigen and scalar OpenBLAS firmware builds can skip this step. A separately provisioned compatible GCC 13.2.0 can be selected through `XRSIGHT_RVV_CC`, but the documented and tested compiler source is the pinned Chipyard environment below. Running on FireSim requires this checkout **regardless of the firmware backend**.

The hardware manifest is [config/firesim/manifest.json](config/firesim/manifest.json). The published component repositories retain upstream history and licenses. 

| Repository | Required revision |
|---|---|
| `pcg108/chipyard`, branch `fix/rocket-saturn-shuttle-rtl` | `621472a27a3ec8ce53af19af3b5e03fb96309d0c` |
| `pcg108/rocket-chip` | `2c0e4784c46f1a67ef39699c7d67aef65e3ea8c0` |
| `pcg108/saturn-vectors` | `9e04c8c6c70a4b4db5989f9a16a89679183a5d1d` |
| `pcg108/shuttle` | `e789aa207148ade9eaed79f12841102597f5a0a5` |
| `pcg108/firesim`, tested `trafficgen-xdma` base | `fa08b6cae659f88d00efb1a2d8c71be85aed97f8` |
| Existing Gemmini project | `69a1c0383283d6ac95c99d2e53136fef4b0bb67e` |
| Namespaced stock Gemmini source for these arrays | `8c3f9923a44a2fe2c7930587be297d6d4f8c09ca` |

**Before building Chipyard, please follow the** [Conda installation instructions](https://github.com/conda-forge/miniforge/) 

To build Chipyard using the pinned dependencies:

```bash
git clone --branch fix/rocket-saturn-shuttle-rtl \
  https://github.com/pcg108/chipyard.git "$CHIPYARD_DIR"
git -C "$CHIPYARD_DIR" checkout --detach 621472a27a3ec8ce53af19af3b5e03fb96309d0c
cd "$CHIPYARD_DIR"
# The pinned .gitmodules uses SSH 
# Local URL overrides allow fetching the components over HTTPS without an SSH key.
git config submodule.generators/rocket-chip.url https://github.com/pcg108/rocket-chip.git
git config submodule.generators/saturn.url https://github.com/pcg108/saturn-vectors.git
git config submodule.generators/shuttle.url https://github.com/pcg108/shuttle.git
# Requires the normal Chipyard host/Conda prerequisites.
# Skip optional indexing, precompilation, FireSim, FireMarshal, and cleanup here.
GIT_CONFIG_COUNT=1 \
GIT_CONFIG_KEY_0=url.https://github.com/pcg108/.insteadOf \
GIT_CONFIG_VALUE_0=git@github.com:pcg108/ \
./build-setup.sh -s 4 -s 5 -s 6 -s 7 -s 8 -s 9 -s 11
cd "$XRSIGHT_ROOT"
```

## Building the ELF

### Zephyr dependencies

Reproduction uses verified pinned dependencies rather than current branches:

| Dependency | Tested revision/version |
|---|---|
| `ucb-bar/zephyr` | `cd45a528d3bf81f9c7aa2a63f9fe512eee0881b0` (4.1.99) |
| Zephyr SDK / application compiler | SDK 0.17.0, GCC 12.2, matching Newlib and libstdc++ |
| Eigen | `68f4e58cfacc686583d16cff90361f0b43bc2c1b` (3.4.1) |
| OpenCV | `4223495e6cd67011f86b8ecd9be1fa105018f3b1` (4.5.4), with the included Zephyr thread-support patch |
| `zekailin00/OpenBLAS` | `6d12fcab91ebf4de5eb23c1218727d2551b8635e` |
| RVV C compiler | Chipyard `riscv64-unknown-elf-gcc` 13.2.0; exact version checked |

Fetch the pinned software sources and checksummed SDK:

```bash
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/setup_xrsight.py" --work "$XRSIGHT_WORK"
"$XRSIGHT_PYTHON" -m pip install -r "$XRSIGHT_WORK/deps/zephyr/scripts/requirements-base.txt"

export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
export ZEPHYR_SDK_INSTALL_DIR="$XRSIGHT_WORK/deps/zephyr-sdk-0.17.0"
export ZEPHYR_BASE="$XRSIGHT_WORK/deps/zephyr"
unset CROSS_COMPILE
```

The helper calls the existing pinned bootstrap with an explicit workspace, verifies dependency revisions and patches, and writes `$XRSIGHT_WORK/software-dependencies.json`. 

### Dataset and Profiles

Download **EuRoC V1_02_medium, ASL format**, from the [ETH EuRoC dataset page](https://projects.asl.ethz.ch/datasets/euroc-mav/) and its [Research Collection download](https://doi.org/10.3929/ethz-b-000690084). Extract it under `$XRSIGHT_WORK/data/V1_02_medium` so `$EUROC_MAV0` contains `cam0/data.csv`, `cam1/data.csv`, `imu0/data.csv`, and camera `data/` directories. The native accuracy report also uses the sequence's ground-truth data. Preserve the downloaded archive and its checksum. If the download is a collection archive, first extract its nested `V1_02_medium.zip`; the compiler needs the ASL directory layout, not a ROS bag.

For a direct command-line download, the following uses the same third-party [GlowBond mirror](https://huggingface.co/datasets/GlowBond/EuRoC_MAV_Dataset) used for our existing dataset, pinned to a specific revision. It downloads the Vicon room 1 collection and extracts only the required sequence archive. The checksum below matches our tested local archive; it is not an independently published ETH checksum.

```bash
mkdir -p "$XRSIGHT_WORK/data/V1_02_medium"
curl --fail --location --retry 3 \
  'https://huggingface.co/datasets/GlowBond/EuRoC_MAV_Dataset/resolve/29d08aeb1c8d9dc5c30497cc538d9cc61f872bb7/vicon_room1.zip' \
  --output "$XRSIGHT_WORK/data/vicon_room1.zip"
unzip -p "$XRSIGHT_WORK/data/vicon_room1.zip" \
  'vicon_room1/V1_02_medium/V1_02_medium.zip' \
  > "$XRSIGHT_WORK/data/V1_02_medium.zip"
printf '%s  %s\n' \
  '0cf5d44baf7aac5d6d705dfb6c5abdd097139bc95aa252dc0c00c66d479ebfc2' \
  "$XRSIGHT_WORK/data/V1_02_medium.zip" | sha256sum --check -
unzip "$XRSIGHT_WORK/data/V1_02_medium.zip" -d "$XRSIGHT_WORK/data/V1_02_medium"
test -f "$EUROC_MAV0/cam0/data.csv"
test -f "$EUROC_MAV0/cam1/data.csv"
test -f "$EUROC_MAV0/imu0/data.csv"
```

The CMake build invokes `embed_euroc_data.py` to embed the first **50 stereo pairs and 501 IMUs** directly into the ELF, extending IMUs through the final camera timestamp plus 50 ms. Thus, the target needs no mounted dataset, filesystem, or network. 

RITNet weights and one prequantized 240×160 eye sample are already bundled under `third_party/ritnet`; `offline_eye` republishes this sample instead of embedding an image sequence.

| `YAML_FILE` profile | Included pipeline |
|---|---|
| `profiles/imu.yaml` | IMU + stereo replay, OpenVINS, IMU integration |
| `profiles/gpu_pipeline.yaml` | Above, plus pose prediction, render loop, timewarp |
| `profiles/eye_tracking.yaml` | Above, plus offline eye image and asynchronous RITNet |

Profiles select plugins at build time. Use a new build directory when changing hardware, backend, or profile.

### Backend and Hardware Compatibility

XRSight is designed to allow flexibility for mapping BLAS operations to different backends in a heterogeneous SoC. This is controlled by `ILLIXR_LINALG_BACKEND`, a **global CMake choice**. 

We do this by configuring the Eigen library to use OpenBLAS as a BLAS backend (`EIGEN_USE_BLAS` is enabled consistently across application/plugin translation units that share Eigen definitions). Only eligible Eigen operations call BLAS (e.g. matrix-matrix and matrix-vector multiplication), so selecting a backend does not move every single operation to an accelerator. As per the original design, OpenVINS is the principal BLAS workload. Existing Eigen decomposition algorithms and FP64 estimator storage remain in place.

`ILLIXR_LINALG_BACKEND` options:

| Setting | Execution | Required hardware |
|---|---|---|
| `eigen` (default) | CPU Eigen using the scalar application ISA | Rocket with scalar FP64 |
| `openblas_scalar` | Static, single-threaded `RISCV64_GENERIC` OpenBLAS | Rocket with scalar FP64 |
| `openblas_rvv` | `RISCV64_ZVL256B` RVV OpenBLAS | Saturn REFV256D128, VLEN 256, FP64, vector context support |
| `openblas_gemmini_fp32` | FP32 Gemmini for real GEMM/GEMV; accepted RVV OpenBLAS for other supported operations | Saturn plus FP32 Gemmini on hart 0 |

Note that the Gemmini backend is **mixed precision**: DGEMM/DGEMV receive doubles, convert/pack into FP32, compute in FP32, and widen/write the result into the caller's layout. This does **not** preserve FP64 multiplication/accumulation. The native acceptance bound remains 1 mm / 0.001 rad.

These are the provided Chipyard configurations in XRSight-RTOS:

| Chipyard hardware family | Core counts provided | Linear algebra choices | Eye profile |
|---|---|---|---|
| Rocket | 1, 2, 4 | Eigen, scalar OpenBLAS | No |
| Rocket + Saturn | 1, 4 | Eigen, scalar or RVV OpenBLAS | No |
| Rocket + Saturn + FP32 Gemmini | 1, 4 | All four | No |
| Rocket + Saturn + INT8 Gemmini | 1, 4 | Eigen, scalar or RVV OpenBLAS | Yes |
| Rocket + Saturn + FP32 + INT8 Gemmini | 1, 4 | All four | Yes |

The majority of our experimentation was done on single and quad core Rocket with Saturn and the 2 Gemmini configurations. Different configurations can be built using standard Chipyard build procedures. Generally, the guidance for building new SoCs is that Saturn attached to every Rocket core is required to use the RVV backends. Multicore workers execute concurrently and accelerator requests are routed to the hart that owns each array. Here, the Gemmini configurations are assumed to be attached to hart 0.

FP32 Gemmini uses custom3, a 4×4 array, 32 KiB scratchpad and 8 KiB accumulators. INT8 Gemmini uses custom2, a 16×16 array, 256 KiB scratchpad and 64 KiB accumulators. Both attach only to hart 0. Each has its own priority-5 worker thread (lowest Zephyr scheduler priority), and only that worker issues its array's instructions. FP32 calls use a globally serialized, aligned 32 MiB BLAS arena. RITNet has separate static activations and publishes results asynchronously. 

There is no selectable production CPU/RVV eye-inference backend; its CPU implementation is a validation reference.

Render/timewarp always use CPU scheduling/math plus the GPU latency model. Their CMake delay settings are not real GPU backend selectors.

### Example A: single-core Rocket, Eigen

To build a simple, single-core compatible ELF that does not use the OpenBLAS backend:

```bash
export ZEPHYR_BASE="$XRSIGHT_WORK/deps/zephyr"
export XRSIGHT_BUILD="$XRSIGHT_WORK/build/rocket-single-eigen"
cmake -S "$XRSIGHT_ROOT" -B "$XRSIGHT_BUILD" -G Ninja \
  -DBOARD=chipyard_riscv64 -DCMAKE_BUILD_TYPE=Release \
  -DPYTHON_EXECUTABLE="$XRSIGHT_PYTHON" -DPython3_EXECUTABLE="$XRSIGHT_PYTHON" \
  -DZEPHYR_MODULES= \
  "-DEXTRA_CONF_FILE=$XRSIGHT_ROOT/config/rocket_single.conf;$XRSIGHT_ROOT/config/rocket_1ghz.conf" \
  -DDTC_OVERLAY_FILE="$XRSIGHT_ROOT/config/rocket_single.overlay" \
  -DOPENCV_SRC_DIR="$XRSIGHT_WORK/deps/opencv" \
  -DYAML_FILE="$XRSIGHT_ROOT/profiles/gpu_pipeline.yaml" \
  -DILLIXR_DATASET_DIR="$EUROC_MAV0" -DILLIXR_DATASET_FRAMES=50 \
  -DILLIXR_CORE_HZ=1000000000 -DILLIXR_PIN_PLUGINS=OFF \
  -DILLIXR_LINALG_BACKEND=eigen
cmake --build "$XRSIGHT_BUILD" --parallel 4
```

For dual or quad plain Rocket, replace **both** `rocket_single.conf` and `rocket_single.overlay` with `rocket_dual` or `rocket_quad`. Core count and interrupt topology are compiled into the ELF, so do not use a single-core ELF for multicore validation.

### Example B: single-core Rocket, Eigen-OpenBLAS (Zephyr SDK only)

In order to build the same example as above but using scalar OpenBLAS as the backend, first build the OpenBLAS library:

```bash
export XRSIGHT_SYSROOT="$ZEPHYR_SDK_INSTALL_DIR/riscv64-zephyr-elf/riscv64-zephyr-elf"
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/build_openblas.py" \
  --source "$XRSIGHT_WORK/openblas-source" \
  --build "$XRSIGHT_WORK/blas/openblas_scalar" \
  --cc "$ZEPHYR_SDK_INSTALL_DIR/riscv64-zephyr-elf/bin/riscv64-zephyr-elf-gcc" \
  --sysroot "$XRSIGHT_SYSROOT" \
  --backend openblas_scalar \
  --jobs 4
```

To use it, use 
```
-DILLIXR_LINALG_BACKEND=openblas_scalar \
-DILLIXR_OPENBLAS_ARCHIVE="$XRSIGHT_WORK/blas/openblas_scalar/lib/libopenblas-zephyr.a"
``` 
in the CPU CMake example above (Example A). Chipyard's compiler and the vector Zephyr preparation are not needed for this scalar build.

### Example C: Vector setup 

Vector builds use an isolated Zephyr copy with an included patch for SDK multilib-selection. First, re-run `setup_xrsight.py` to use this patched version:

```bash
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/setup_xrsight.py" --work "$XRSIGHT_WORK" --vector

# The Chipyard compiler is available after the prerequisite step
export XRSIGHT_RVV_CC="$CHIPYARD_DIR/.conda-env/riscv-tools/bin/riscv64-unknown-elf-gcc"
export XRSIGHT_SYSROOT="$ZEPHYR_SDK_INSTALL_DIR/riscv64-zephyr-elf/riscv64-zephyr-elf"
"$XRSIGHT_RVV_CC" --version
```


For RVV, invoke `build_openblas.py` with `--backend openblas_rvv --cc "$XRSIGHT_RVV_CC"` and the same Zephyr sysroot, using a separate archive directory. 

```
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/build_openblas.py" \
  --source "$XRSIGHT_WORK/openblas-source"   \
  --build "$XRSIGHT_WORK/blas/openblas_rvv"  \
  --cc "$XRSIGHT_RVV_CC"    \
  --sysroot "$XRSIGHT_SYSROOT" \
  --backend openblas_rvv \
  --jobs 4
```

Select the vector Zephyr, `config/vector.conf`, and a matching `rocket_single_rvv.overlay` or `rocket_quad_rvv.overlay` when configuring the application.

```
export ZEPHYR_BASE="$XRSIGHT_WORK/vector-deps/zephyr"
export XRSIGHT_BUILD="$XRSIGHT_WORK/build/rocket-single-rvv" # or rocket-quad-rvv
cmake -S "$XRSIGHT_ROOT" -B "$XRSIGHT_BUILD" -G Ninja \
  -DBOARD=chipyard_riscv64 -DCMAKE_BUILD_TYPE=Release \
  -DPYTHON_EXECUTABLE="$XRSIGHT_PYTHON" -DPython3_EXECUTABLE="$XRSIGHT_PYTHON" \
  -DZEPHYR_MODULES= \
  "-DEXTRA_CONF_FILE=$XRSIGHT_ROOT/config/vector.conf;$XRSIGHT_ROOT/config/rocket_1ghz.conf" \
  -DDTC_OVERLAY_FILE="$XRSIGHT_ROOT/config/rocket_single_rvv.overlay" \
  -DOPENCV_SRC_DIR="$XRSIGHT_WORK/deps/opencv" \
  -DYAML_FILE="$XRSIGHT_ROOT/profiles/gpu_pipeline.yaml" \
  -DILLIXR_DATASET_DIR="$EUROC_MAV0" -DILLIXR_DATASET_FRAMES=50 \
  -DILLIXR_CORE_HZ=1000000000 -DILLIXR_PIN_PLUGINS=OFF \
  -DILLIXR_LINALG_BACKEND=openblas_rvv \
  -DILLIXR_OPENBLAS_ARCHIVE="$XRSIGHT_WORK/blas/openblas_rvv/lib/libopenblas-zephyr.a" 
cmake --build "$XRSIGHT_BUILD" --parallel 4

```

### Example D: quad-core Saturn + both Gemmini arrays, eye tracking, RVV packing

For this example, we need to jump ahead to [Running on FireSim](#running-on-firesim) to elaborate the corresponding hardware configuration with Gemmini. The elaboration generates the Gemmini header files that we use here. Follow the instrucions through [Run elaboration without synthesis first](#run-elaboration-without-synthesis-first) where we set `XRSIGHT_FP32_HEADER` to its generated `gemmini_params_illixr.h`. Use the `FireSimILLIXRSingleRocketDualGemminiSaturnConfig` or `FireSimILLIXRQuadRocketDualGemminiSaturnConfig` configurations to get the requisite Gemmini headers.

Additionally, verify that the generated `gemmini_params_illixr_int8.h` matches the bundled RITNet parameters in [third_party/ritnet/port/include/gemmini_params.h](third_party/ritnet/port/include/gemmini_params.h). Do not substitute a header from a different accelerator geometry.

Build the OpenBLAS library mapping certain operations to FP32 Gemmini:

```bash
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/build_openblas.py" \
  --source "$XRSIGHT_WORK/openblas-source" \
  --build "$XRSIGHT_WORK/blas/openblas_gemmini_fp32" \
  --cc "$XRSIGHT_RVV_CC" --sysroot "$XRSIGHT_SYSROOT" \
  --backend openblas_gemmini_fp32 --gemmini-params "$XRSIGHT_FP32_HEADER" --jobs 4
```
Build the ELF:
```
export ZEPHYR_BASE="$XRSIGHT_WORK/vector-deps/zephyr"
export XRSIGHT_BUILD="$XRSIGHT_WORK/build/quad-eye-rvvpack"
cmake -S "$XRSIGHT_ROOT" -B "$XRSIGHT_BUILD" -G Ninja \
  -DBOARD=chipyard_riscv64 -DCMAKE_BUILD_TYPE=Release \
  -DPYTHON_EXECUTABLE="$XRSIGHT_PYTHON" -DPython3_EXECUTABLE="$XRSIGHT_PYTHON" \
  -DZEPHYR_MODULES= \
  "-DEXTRA_CONF_FILE=$XRSIGHT_ROOT/config/rocket_quad.conf;$XRSIGHT_ROOT/config/rocket_1ghz.conf;$XRSIGHT_ROOT/config/vector.conf" \
  -DDTC_OVERLAY_FILE="$XRSIGHT_ROOT/config/rocket_quad_rvv.overlay" \
  -DOPENCV_SRC_DIR="$XRSIGHT_WORK/deps/opencv" \
  -DYAML_FILE="$XRSIGHT_ROOT/profiles/eye_tracking.yaml" \
  -DILLIXR_DATASET_DIR="$EUROC_MAV0" -DILLIXR_DATASET_FRAMES=50 \
  -DILLIXR_CORE_HZ=1000000000 -DILLIXR_PIN_PLUGINS=OFF \
  -DILLIXR_LINALG_BACKEND=openblas_gemmini_fp32 \
  -DILLIXR_OPENBLAS_ARCHIVE="$XRSIGHT_WORK/blas/openblas_gemmini_fp32/lib/libopenblas-zephyr.a" \
  -DILLIXR_GEMMINI_PACKING=rvv -DILLIXR_GEMMINI_PACKING_TRAVERSAL=rows \
  -DILLIXR_PACKING_SATURN_COMPAT=ON \
  -DILLIXR_RITNET_DIAGNOSTICS=OFF \
  -DILLIXR_HPM_PROFILE=ON
cmake --build "$XRSIGHT_BUILD" --parallel 4
```
The `ILLIXR_GEMMINI_PACKING` setting controls which backend is used to perform FP64->FP32 packing and conversion for use with the FP32 Gemmini. `scalar` remains the default, while `rvv` uses the Saturn Vector Unit to accelerate the packing. Explicit RVV packing fuses conversion with packing and supports signed strides/tails without rebuilding temporary arrays element by element in scalar C++. `rows` is the traversal tested in the complete FPGA pipeline; `contiguous` has standalone equivalence/benchmark evidence. 

The tested Saturn images require explicit `ILLIXR_PACKING_SATURN_COMPAT=ON` for the tested RVV conversion path. Scalar packing still requires Saturn for the backend's other BLAS operations.

For a single-core equivalent, change the quad configuration/overlay to their single-core versions and use the single-core dual-array image. For FP32-only hardware, select `profiles/gpu_pipeline.yaml`. For INT8-only hardware, keep the eye profile and select Eigen, scalar OpenBLAS, or RVV OpenBLAS instead of FP32 Gemmini.

## Running on FireSim

### Source checkout and host setup

Follow the [Chipyard Setup](#chipyard-setup) instructions from earlier- do not clone it again! If you built only Eigen or scalar OpenBLAS and skipped that step, it needs to be completed now.

Note that we will to override the Firesim checkout in that Chipyard version to get the one used by the accepted FPGA images.

```bash
cd "$CHIPYARD_DIR"
git submodule update --init sims/firesim
git -C sims/firesim fetch https://github.com/pcg108/firesim.git fa08b6cae659f88d00efb1a2d8c71be85aed97f8
git -C sims/firesim checkout --detach fa08b6cae659f88d00efb1a2d8c71be85aed97f8
./scripts/firesim-setup.sh
git -C generators/gemmini fetch https://github.com/ucb-bar/gemmini.git 8c3f9923a44a2fe2c7930587be297d6d4f8c09ca

# Additional patches provide streaming FIRRTL emission, correct unsigned 64-bit handling of the 100-billion-cycle limit, and U250 build-worker/report handling.
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/setup_firesim.py" install \
  --chipyard "$CHIPYARD_DIR"
```


Complete the [FireSim local FPGA host setup](https://docs.fires.im/en/latest/local-fpga-initial-setup/) and [U250 setup](https://docs.fires.im/en/latest/getting-started-guides/on-premises-fpga-getting-started/initial-setup/xilinx-alveo-u250/) for XDMA, device discovery, SSH to localhost, and FPGA permissions. This flow is tested with Vivado 2022.1. Note that board provisioning, driver installation, or other `sudo` steps in the Firesim documentation may require your host administrator- the project helpers do not perform those system changes.

```bash
cd "$CHIPYARD_DIR/sims/firesim"
source sourceme-manager.sh
export XRSIGHT_DEPLOY="$CHIPYARD_DIR/sims/firesim/deploy"

# e.g. export XRSIGHT_FPGA_DB=/opt/firesim-db.json
export XRSIGHT_FPGA_DB=/absolute/path/to/your/discovered-fpga-db.json
```
Use the FPGA database generated for **your board**, selecting one U250. Do not copy another machine's PCI address or serial number. The following flow assumes the FPGA has already been provisioned for FireSim.

On a fresh checkout, initialize the manager once before editing its configuration
files. From the sourced FireSim environment:

```bash
cd "$XRSIGHT_DEPLOY"
firesim managerinit --platform xilinx_alveo_u250
```

This creates `config_build.yaml`, `config_build_recipes.yaml`, `config_hwdb.yaml`,
and `config_runtime.yaml` in `deploy/`. Do this before customizing those files:
`managerinit` backs up existing versions into `sample-backup-configs/` and replaces
them with examples. The project-specific build files generated below are separate.
If you already built an image without running `managerinit`, you do not need to
rerun initialization or rebuild. Create `config_hwdb.yaml` and paste the generated
HWDB entry into it. Also supply `config_build_recipes.yaml` as described under
[Program and run](#4-program-and-run): this FireSim revision loads recipes even
when metasimulation is disabled.

### Hardware configurations

Running `python "$XRSIGHT_ROOT/scripts/setup_firesim.py" configure --help` lists aliases for the following Hardware Configurations. Choose one and retain that choice through elaboration, firmware, bitstream packaging, and runtime setup.

| Hardware | Single-core class | Multicore class |
|---|---|---|
| Rocket | `FireSimILLIXRSingleRocketConfig` | `FireSimILLIXRDualRocketConfig`, `FireSimILLIXRQuadRocketConfig` |
| Rocket + Saturn | `FireSimILLIXRSingleRocketSaturnConfig` | `FireSimILLIXRQuadRocketSaturnConfig` |
| + FP32 Gemmini | `FireSimILLIXRSingleRocketGemminiSaturnConfig` | `FireSimILLIXRQuadRocketGemminiSaturnConfig` |
| + INT8 Gemmini | `FireSimILLIXRSingleRocketInt8GemminiSaturnConfig` | `FireSimILLIXRQuadRocketInt8GemminiSaturnConfig` |
| + both Gemmini arrays | `FireSimILLIXRSingleRocketDualGemminiSaturnConfig` | `FireSimILLIXRQuadRocketDualGemminiSaturnConfig` |

The corresponding aliases are `illixr_u250_rocket_{single,dual,quad}`, and `illixr_u250_rocket_{saturn,gemmini_saturn,int8_gemmini_saturn,dual_gemmini_saturn}_{single,quad}`.

All listed Rocket targets retain 256 MiB RAM, HTIF/TSI loading and exit, the 500 MHz/500 kHz generated ratio, default FireSim bridges without TracerV, and the tested FASED latency-bandwidth model. Saturn configurations use REFV256D128 on every hart.

### Elaborate and build the selected image

Prepare build-only files. The canonical example is quad-core dual-Gemmini with Saturn:

```bash
export XRSIGHT_HW=illixr_u250_rocket_dual_gemmini_saturn_quad
export XRSIGHT_HW_WORK="$XRSIGHT_WORK/hardware/$XRSIGHT_HW"
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/setup_firesim.py" configure \
  --chipyard "$CHIPYARD_DIR" --work "$XRSIGHT_HW_WORK" \
  --config "$XRSIGHT_HW" --build-only
```

The renderer writes complete manager files with absolute paths, plus `commands.json` identifying the matching elaboration command, generated RTL, staging directory, and accelerator headers. Templates live in [config/firesim](config/firesim). They use JSON syntax, which is valid YAML; FireSim accepts these `.yaml` files directly. 

The build recipe has this structure (the helper replaces `/ABS/...` with your paths):

```yaml
illixr_u250_rocket_dual_gemmini_saturn_quad:
  PLATFORM: xilinx_alveo_u250
  TARGET_PROJECT: firesim
  TARGET_PROJECT_MAKEFRAG: /ABS/chipyard/generators/firechip/chip/src/main/makefrag/firesim
  DESIGN: FireSim
  TARGET_CONFIG: FireSimILLIXRQuadRocketDualGemminiSaturnConfig
  PLATFORM_CONFIG: BaseXilinxAlveoU250Config
  deploy_quintuplet: null
  platform_config_args:
    fpga_frequency: 30
    build_strategy: NORETIMING
  post_build_hook: null
  metasim_customruntimeconfig: null
  bit_builder_recipe: /ABS/chipyard/sims/firesim/deploy/bit-builder-recipes/xilinx_alveo_u250.yaml
```

`config_build.yaml` selects that alias in `builds_to_run`, an externally provisioned localhost build farm, and a private build directory. These build recipe files will be used when [building the FPGA images](#build-the-fpga-image-from-the-prepared-recipes). 

#### Run elaboration without synthesis first:

```bash
cd "$CHIPYARD_DIR/sims/firesim"
set -o pipefail
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/firesim_resource_guard.py" \
  --scratch "$XRSIGHT_WORK" --run-dir "$XRSIGHT_HW_WORK/elaboration-guard" \
  --latch "$XRSIGHT_HW_WORK/elaboration-stopped.json" --jobs 4 -- \
  make -C "$CHIPYARD_DIR/sims/firesim/sim" \
    PLATFORM=xilinx_alveo_u250 TARGET_PROJECT=firesim \
    TARGET_PROJECT_MAKEFRAG="$CHIPYARD_DIR/generators/firechip/chip/src/main/makefrag/firesim" \
    DESIGN=FireSim TARGET_CONFIG=FireSimILLIXRQuadRocketDualGemminiSaturnConfig \
    PLATFORM_CONFIG=BaseXilinxAlveoU250Config verilog \
  2>&1 | tee "$XRSIGHT_HW_WORK/elaboration.log"
```

For another alias, use its exact `TARGET_CONFIG` from the table or the command array in `commands.json`. Set the generated paths from that file; for this example:

```bash
export XRSIGHT_STAGING="$CHIPYARD_DIR/sims/firesim-staging/generated-src/firechip.chip.FireSim.FireSimILLIXRQuadRocketDualGemminiSaturnConfig"
export XRSIGHT_RTL="$CHIPYARD_DIR/sims/firesim/sim/generated-src/xilinx_alveo_u250/xilinx_alveo_u250-firesim-FireSim-FireSimILLIXRQuadRocketDualGemminiSaturnConfig-BaseXilinxAlveoU250Config/FireSim-generated.sv"
export XRSIGHT_FP32_HEADER="$CHIPYARD_DIR/gemmini_params_illixr.h"
```
#### Sanity check elaborated RTL:

```
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/check_firesim_platform.py" \
  --chipyard "$CHIPYARD_DIR" --config "$XRSIGHT_HW" \
  --staging-dir "$XRSIGHT_STAGING" --rtl "$XRSIGHT_RTL" \
  --elaboration-log "$XRSIGHT_HW_WORK/elaboration.log" --elaboration-only \
  --output "$XRSIGHT_HW_WORK/platform.json"
```

This checks generated harts, ISA/vector geometry, memory/interrupt maps, clock bridges, boot ROM, accelerator instances and headers, FASED limits, and HTIF host settings. It is **not** routed-timing or FPGA-run acceptance. Now build the matching preflight and workload ELFs using the earlier CMake examples.

Package each firmware build for the runner and analyzers:

```bash
export XRSIGHT_FIRMWARE="$XRSIGHT_WORK/artifacts/quad-eye-rvvpack"
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/package_firmware.py" \
  --build "$XRSIGHT_BUILD" --deps "$XRSIGHT_WORK/vector-deps" \
  --platform "$XRSIGHT_HW_WORK/platform.json" --output "$XRSIGHT_FIRMWARE"
```

Use `--deps "$XRSIGHT_WORK/deps"` for scalar Zephyr builds. The helper records compiled settings, firmware/source/dependency hashes, dataset identity, memory/ELF reports, backend identity, and the BLAS symbol audit. It refuses an existing destination. Package the separate preflight build in its own artifact directory too.

#### Build the FPGA image from the prepared recipes:

It is recommended to launch the following in a `tmux` session, after sourcing `sourceme-manager.sh`. 

**Ensure that `vivado` is installed and accessible from the CLI before running the following**

If one needs to re-run the following, delete the generated build directory and re-run the `setup_firesim.py` `--build-only` step from above.

```bash
cd "$XRSIGHT_DEPLOY"

"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/firesim_resource_guard.py" \
  --scratch "$XRSIGHT_WORK" --run-dir "$XRSIGHT_HW_WORK/build-guard" \
  --latch "$XRSIGHT_HW_WORK/build-stopped.json" --jobs 4 -- \
  python3 "$XRSIGHT_ROOT/scripts/firesim_manager.py" --chipyard "$CHIPYARD_DIR" -- \
    buildbitstream -b "$XRSIGHT_HW_WORK/config_build.yaml" \
    -r "$XRSIGHT_HW_WORK/config_build_recipes.yaml" \
    -a "$XRSIGHT_HW_WORK/config_hwdb_build.yaml"
```

This wrapper invokes the normal `firesim buildbitstream` manager while forwarding resource-ownership metadata to localhost build workers. 

In our experience, it is best to use at most four workers. The supplied guard enforces a 48 GiB available-memory reserve, 64 GiB per-process RSS ceiling, and stop on a new kernel OOM kill. These settings are based on our own experience building Firesim bitstreams. 

The bitstream build will take several hours. Afterwards, the console will display something like:

```
Your bitstream has been created!
Add

illixr_u250_rocket_dual_gemmini_saturn_quad:
    bitstream_tar: file:///home/illixrtest/xrsight-work/chipyard/sims/firesim/deploy/results-build/2026-10-07--00-19-16-illixr_u250_rocket_dual_gemmini_saturn_quad/cl_xilinx_alveo_u250-firesim-FireSim-FireSimILLIXRQuadRocketDualGemminiSaturnConfig-BaseXilinxAlveoU250Config/firesim.tar.gz
    deploy_quintuplet_override: null
    custom_runtime_config: null

to your config_hwdb.yaml to use this hardware configuration.

```

### Run the built image

Now, follow the normal FireSim workflow: **copy the build's HWDB entry, select the hardware
and ELF in the runtime files, run `infrasetup`, then run `runworkload`.** 

The examples below use `/home/illixrtest/xrsight-work` and a quad-core Rocket+Saturn dual-Gemmini
image. Replace that home directory with yours. Run FireSim commands from
`chipyard/sims/firesim/deploy` after sourcing `sourceme-manager.sh` as described
above. YAML paths must be literal paths: `$HOME`, `$XRSIGHT_WORK`, and `~` are not
expanded inside YAML.

#### 1. Copy the completed build into the HWDB

At the end of `firesim buildbitstream`, FireSim prints an entry to add to
`config_hwdb.yaml`. It also saves it under `deploy/built-hwdb-entries/`.
**Copy that entire entry into `deploy/config_hwdb.yaml` (create the file if it
does not exist).** No template is required: the generated entry is valid HWDB YAML
on its own. Keep its
`bitstream_tar` path and `deploy_quintuplet_override` exactly as generated.
For this example its name is `illixr_u250_rocket_dual_gemmini_saturn_quad`.


Before programming, confirm that the Vivado implementation reports show completed
routing and passing final setup/hold and bus-skew timing. The optional
[recorded-run guide](docs/firesim-recorded-runs.md) automates these checks and
records artifact hashes.

#### 2. Point a workload at your ELF

Create `deploy/workloads/xrsight.json` with:

```json
{
  "benchmark_name": "xrsight",
  "common_bootbinary": "zephyr.elf",
  "common_rootfs": null,
  "common_outputs": [],
  "common_simulation_outputs": ["uartlog", "memory_stats0.csv"],
  "workloads": [{"name": "xrsight"}]
}
```

Put your matching firmware at `deploy/workloads/xrsight/zephyr.elf`. For example,
from the `deploy` directory, after packaging the quad-core workload:

```bash
mkdir -p workloads/xrsight
# the ELF can also just be copied directly from the build directory, e.g. $XRSIGHT_BUILD/zephyr/zephyr.elf
cp ~/xrsight-work/artifacts/quad-eye-rvvpack/zephyr.elf workloads/xrsight/zephyr.elf
```

For a new image, run the platform-preflight ELF first (built in a separate build
directory with `-DILLIXR_PLATFORM_CHECK_ONLY=ON`). Use the same workload definition
but copy the preflight ELF to that filename. After the hart/atomic/timer and enabled
accelerator checks pass and the program exits normally, copy the full-workload
ELF and repeat `infrasetup` and `runworkload`. The core count and accelerators in
the ELF must match the image. `common_rootfs: null` is intentional: the dataset is
embedded in the ELF, and this run has no Linux filesystem image.

#### 3. Fill in the runtime YAML

Create `deploy/config_runtime_xrsight.yaml` using the following complete example.
The machine-specific fields are **`default_simulation_dir`** (a writable run
folder) and **`default_fpga_db`** (your U250 discovery file). Set
**`default_hw_config`** to the exact entry name you pasted into `config_hwdb.yaml`.
**`workload_name`** names the JSON file from step 2.

```yaml
run_farm:
  base_recipe: run-farm-recipes/externally_provisioned.yaml
  recipe_arg_overrides:
    run_farm_tag: xrsight-rtos
    default_platform: XilinxAlveoU250InstanceDeployManager
    default_simulation_dir: /home/illixrtest/xrsight-work/runs/quad-eye-1
    default_fpga_db: /opt/firesim-db.json
    run_farm_host_specs:
      - one_u250:
          num_fpgas: 1
          num_metasims: 0
          use_for_switch_only: 0
    run_farm_hosts_to_use:
      - localhost: one_u250

metasimulation:
  metasimulation_enabled: false
  metasimulation_host_simulator: verilator
  metasimulation_only_plusargs: ""
  metasimulation_only_vcs_plusargs: ""

target_config:
  topology: no_net_config
  no_net_num_nodes: 1
  link_latency: 6405
  switching_latency: 10
  net_bandwidth: 200
  profile_interval: 1000000
  default_hw_config: illixr_u250_rocket_dual_gemmini_saturn_quad
  plusarg_passthrough: >-
    +max-cycles=100000000000
    +mm_readLatency_0=30 +mm_writeLatency_0=30
    +mm_readMaxReqs_0=10 +mm_writeMaxReqs_0=10
    +mm_useHardwareDefaultRuntimeSettings_0
    +fesvr-step-size=10000 +idle-counts=1 +fesvr-wait-ticks=8

tracing:
  enable: false
  output_format: 0
  selector: 1
  start: 0
  end: -1

autocounter:
  read_rate: 0

workload:
  workload_name: xrsight.json
  terminate_on_completion: false
  suffix_tag: quad-eye-1

host_debug:
  zero_out_dram: true
  disable_synth_asserts: false

synth_print:
  start: 0
  end: -1
  cycle_prefix: true
```

Keep the listed memory plusargs: the request limit is **10**, not 16, because the
compiled field is only four bits wide. These are the same FASED settings used in
our accepted runs. For each new run, change the run folder and `suffix_tag` so
previous output is preserved. For preflight, use names such as `quad-preflight-1`.

#### 4. Program and run

If you skipped `managerinit`, first create the missing default recipe file from
the bundled example. From `deploy`, run:

```bash
cp -n sample-backup-configs/sample_config_build_recipes.yaml config_build_recipes.yaml
```
From `deploy`, run:

```bash
firesim infrasetup -c config_runtime_xrsight.yaml -a config_hwdb.yaml
```

This prepares the matching driver, programs the FPGA, and stages the workload.
After it succeeds:

```bash
timeout --foreground --signal=INT --kill-after=60s 24h \
  firesim runworkload -c config_runtime_xrsight.yaml -a config_hwdb.yaml
```

The command is ordinary `firesim runworkload` with a 24-hour host watchdog; the
runtime YAML also sets the 100-billion-target-cycle limit. A timeout or
interruption is incomplete, never a passing run. If interrupted, check the board
and simulator state before starting another run; do not assume the FPGA is idle.

Monitor it in another terminal:

```bash
screen -ls
screen -r fsim0
# Detach with Ctrl-a, d. Or read the UART directly:
tail -f /home/illixrtest/xrsight-work/runs/quad-eye-1/sim_slot_0/uartlog
```

FireSim copies completed workload outputs into `deploy/results-workload/`.
The firmware exports buffered trace batches at shutdown, so a quiet UART during
processing does not by itself mean the run is stuck. See [Outputs and analysis](#outputs-and-analysis)
to decode the trace. For automated hardware checks and complete acceptance
reports, use the separate [recorded-run workflow](docs/firesim-recorded-runs.md).

## Outputs and analysis

### Files produced by a run

| Location | Contents |
|---|---|
| `<default_simulation_dir>/sim_slot_0/uartlog` | Live firmware output and FireSim exit/cycle records |
| `<default_simulation_dir>/sim_slot_0/memory_stats0.csv` | Periodic FASED statistics and available AXI error indicators |
| `$XRSIGHT_DEPLOY/results-workload/<timestamp>-<workload>/` | Manager-collected simulation outputs |
| Recorded-run helper output (optional) | `manager-runworkload.log`, `execution.json`, return status, watchdog/interruption state, and host elapsed time |
| `$XRSIGHT_FIRMWARE/` | ELF, compiled config/DTS, dataset manifest, BLAS audit, build provenance, and hashes |
| Collector output directory | Preserved raw UART, decoded `console.log`, `trace-transfer.json`, native outputs, `analysis.json`, `run.json`, and evidence manifests |

The firmware buffers records in RAM and transfers binary batches through HTIF. The host checks batch framing/checksums and formats JSON after processing. Trace export is a separate phase; its time must not be reported as application execution time. [Batched-trace documentation](docs/batched-traces.md) describes the protocol and earlier transfer measurements.

### Build the host reference tools

Native validation uses the same local estimator implementation, dataset/calibration, and actual delivered camera/IMU sequence. It requires a **host** OpenCV 4.5.4 installation and Eigen ≥3.4; do not point native CMake at the RISC-V library. If those versions are not already installed, build the pinned sources into a private prefix:

```bash
export XRSIGHT_WORK="$HOME/xrsight-work"
export XRSIGHT_HOST_PREFIX="$XRSIGHT_WORK/host-deps"

# use the host cmake
/usr/bin/cmake --version

cmake -S "$XRSIGHT_WORK/deps/modules/lib/eigen" \
  -B "$XRSIGHT_WORK/build/host-eigen" \
  -DCMAKE_INSTALL_PREFIX="$XRSIGHT_HOST_PREFIX" \
  -DBUILD_TESTING=OFF

cmake --build "$XRSIGHT_WORK/build/host-eigen" \
  --target install --parallel 4

cmake -S "$XRSIGHT_WORK/deps/opencv" \
  -B "$XRSIGHT_WORK/build/host-opencv" \
  -DWITH_PNG=ON -DBUILD_PNG=ON -DBUILD_ZLIB=ON \
  -DWITH_VA=OFF -DWITH_VA_INTEL=OFF \
  -DHAVE_VA=OFF -DHAVE_VA_INTEL=OFF

cmake --build "$XRSIGHT_WORK/build/host-opencv" \
  --target install --parallel 4

export XRSIGHT_NATIVE="$XRSIGHT_WORK/build/native"
cmake -S "$XRSIGHT_ROOT/tests/native" -B "$XRSIGHT_NATIVE" -G Ninja \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="$XRSIGHT_HOST_PREFIX" \
  -DPython3_EXECUTABLE="$XRSIGHT_PYTHON"

cmake --build "$XRSIGHT_NATIVE" --target estimator_replay prediction_reference --parallel 4
```

These are ordinary host builds, with no Zephyr toolchain file. The native harness disables Eigen parallel/vector reductions and FP contraction for its reference comparisons. See [native test documentation](tests/native/README.md).

### Inspect results or run full acceptance checks

To inspect and validate the logs from the firesim run, use:

```bash
mkdir -p "$XRSIGHT_WORK/inspection"

cp /home/illixrtest/xrsight-work/runs/quad-eye-1/sim_slot_0/uartlog "$XRSIGHT_WORK/inspection/console.log"

# trace_batches.py modifies its input and retains the original as `console.batched.log`
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/trace_batches.py" "$XRSIGHT_WORK/inspection/console.log"

"$XRSIGHT_NATIVE/estimator_replay" --dataset "$EUROC_MAV0" \
  --trace "$XRSIGHT_WORK/inspection/console.log" --output "$XRSIGHT_WORK/inspection/native.log"

# analyze_spike.py` is shared across platforms despite its name
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/analyze_spike.py" "$XRSIGHT_WORK/inspection/console.log" \
  --native "$XRSIGHT_WORK/inspection/native.log" --dataset "$EUROC_MAV0" \
  --dataset-manifest "$XRSIGHT_FIRMWARE/dataset_manifest.json" \
  --harts 4 --placement unpinned --require-initialized --require-async \
  --require-platform --require-gpu --expected-timer-hz 1000000 \
  --expected-core-hz 1000000000 --output "$XRSIGHT_WORK/inspection/analysis.json"
```

### Interpreting results

A passing full workload requires normal HTIF exit, orderly shutdown, complete trace export, initialized finite VIO output, normalized quaternions, ordered timestamps, all **501 IMUs at both consumers**, and accounting for all **50 camera pairs** as processed/skipped/dropped. Native replay must remain within **1 mm position and 0.001 rad orientation** for that run's delivered input sequence. Enabled accelerator/vector self-tests and prediction/transform checks must pass. Eye-enabled runs additionally validate inference outputs, routing, and asynchronous publication/consumption.

Per-plugin `work_counts` count specific processing events, not every function call or thread wakeup:

| Plugin | One `work_counts` increment represents |
|---|---|
| `offline_imu` | One IMU sample delivered |
| `offline_cam` | One decoded stereo pair offered to the VIO queue, including attempts dropped because the queue is full |
| `openvins` | One IMU sample **or** one stereo pair processed |
| `imu_integrator` | One IMU sample processed |
| `render_loop` | One render job submitted |
| `timewarp` | One warp job submitted; empty or missed display opportunities do not count |

`publication_counts` separately count output publications. For example, OpenVINS processing 501 IMUs and 17 stereo pairs records 518 work events, but could publish only 15 poses. Both arrays are indexed by hart: `[100, 200, 150, 68]` records events observed on harts 0–3. These observations identify the hart at each instrumented point; a thread can migrate during an operation.

Pose prediction has separate per-caller counters for prediction requests/results. Eye tracking separately records image publications, completed inferences, and timewarp reads. Work counters measure work volume and observed placement, not CPU utilization or execution cost; use the HPM cycle/instruction counters for performance measurements.

BLAS records distinguish caller harts from the hart-0 accelerator worker, operation counts/dimensions, mutex and queue waits, packing/unpacking, execution intervals, and scratch high-water use. These elapsed intervals can include preemption- they are not exclusive accelerator busy cycles. RVV kernel/packing counters and disassembly audits establish actual dispatch rather than relying only on ELF ISA flags.

Graphics accounting permits frame reuse: completed renders are distinct selected plus never-selected frames. Completed warps are first plus repeated frame uses. Display slots are new output, repeated output, or no output. 

Application time covers target processing. Trace-export time covers deferred output. FireSim target cycles include the executed target interval reported by the simulator. Host runtime measures real simulation wall time and must be labeled with whether programming/setup/analysis are included. Spike's bounded instruction-step counter has different semantics and cannot supply a hardware speedup claim.

**Native agreement is runtime equivalence, not physical trajectory accuracy.** Ground-truth trajectory drift is reported separately. Live paced runs can deliver different camera sequences when backend performance changes, so compare each run against its own native replay and use controlled equal-work fixtures for kernel speedups.


### Hardware performance profiling

Rocket builds can enable thirteen HPM counters per hart alongside cycles and
retired instructions. Firmware profiling is opt-in with
`-DILLIXR_HPM_PROFILE=ON` and requires a matching HPM-enabled bitstream. See
[events, thread-aware attribution, and validation status](docs/hardware-performance-counters.md).

The hardware counters belong to each hart. Software divides their increments
into per-plugin totals using Zephyr's thread-switch tracing callbacks:

1. Before a worker starts, `threadloop::start()` registers its Zephyr thread ID
   and plugin name with HPM.
2. When Zephyr switches threads, its tracing callbacks invoke
   [`hpm.cpp`](src/hpm.cpp). The profiler reads the current hart's counters and
   charges the difference from the previous snapshot to the outgoing context.
3. On switch-in, the profiler looks up the incoming thread's registered owner
   and establishes the baseline for its next execution interval. Switch gaps
   and measured profiler work have separate accounting categories.

For example, when `imu_integrator` blocks waiting for an IMU sample, its
scheduled interval ends. If OpenVINS runs next, those counter increments belong
to OpenVINS. When the integrator resumes, a new interval contributes to its
existing totals. Each hart maintains its own snapshots, so migration never
requires subtracting counters read on different harts. These totals cover the
worker's scheduled execution, including queue handling and publication, rather
than only its numerical function or individual iterations.

ISR callbacks account for interrupt-handler bodies separately. Explicit HPM
scopes temporarily identify pose prediction within the render/timewarp caller's
thread, and the FP32 Gemmini worker inherits its requester's identity while
recording packing, accelerator, and unpacking phases. Accounting starts before
replay release and stops after orderly workload completion, before trace export.
Counters are accumulated in memory and exported afterward; no per-switch
console logging is needed. Sequential counter reads and hook boundaries mean
these are scheduled-context measurements, with sampling skew and only a lower
bound on profiler overhead, rather than exact causal attribution of every event.

Summarize preserved run directories or a matrix with one command:

```sh
"$XRSIGHT_PYTHON" "$XRSIGHT_ROOT/scripts/summarize_performance.py" \
  "$HOME/xrsight-work/runs/quad-eye-1/sim_slot_0/uartlog" \
  --output "$HOME/xrsight-work/reports/quad-eye-1"
```

The report combines VIO, queues, BLAS/accelerators, eye tracking, display deadlines,
placement, memory traffic, runtimes, and available per-plugin HPM measurements.
Legacy measurements remain unavailable and incomplete runs remain incomplete.


## Citing XRSight

```
@inproceedings{ganesh2025xrsight,
  title={XRSight: An End-to-End Hardware-Software Co-Design Platform for XR SoC Evaluation},
  author={Ganesh, Prashanth and Lin, Zekai and Shao, Yakun Sophia},
  booktitle={2025 IEEE International Symposium on Workload Characterization (IISWC)},
  pages={452--463},
  year={2025},
  organization={IEEE}
}
```

## Contributors

Prashanth Ganesh - prashanthcganesh108@berkeley.edu

Zekai Lin - zekailin00@berkeley.edu

Isaac Tsang - isaacmiltontsang@berkeley.edu

Sophia Shao - ysshao@berkeley.edu
