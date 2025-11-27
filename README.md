# XRSight

XRSight is a hardware–software co-design framework for developing and characterizing extended reality (XR) workloads on embedded systems-on-chip (SoCs). XRSight is built on top of [Firesim](https://docs.fires.im/en/latest/index.html) (an FPGA-accelerated RTL simulator) and [ILLIXR](https://illixr.github.io/ILLIXR/latest/) (an XR software system testbed), bridging a realistic XR workload with RISC-V [Chipyard](https://chipyard.readthedocs.io/en/latest/) -based SoCs. 

The XRSight paper can be found in the IISWC 2025 proceedings [here](https://ieeexplore.ieee.org/document/11242088). 

# Notes

XRSight is under active development (XRSight 2.0) to transition to an RTOS-based runtime. Ths involves a re-design of the workload from the ground-up using [Zephyr RTOS](https://chipyard.readthedocs.io/en/latest/Software/Zephyr.html). The original runtime runs on Linux (as does the original ILLIXR), which has limitations on Chipyard SoCs, including multithreading issues with Saturn Vector Unit, multi-core Rocket issues, and more. However, as RTOS runs closer to baremetal, it allows for more sophisticated heterogeneous Chipyard-based SoCs. This repository will be updated as the new workload matures in the coming months.

The original flow is documented below to build the SoCs and workloads from scratch. Additionally, The IISWC 2025 [Artifact](https://github.com/pcg108/xrsight-iiswc-artifact) contains instructions to run the pre-built workloads and bitstreams, along with the processing scripts to generate the results in the original paper. 

# Setup

There are 3 components to building the XRSight system. The system relies on a host Ubuntu machine, and a Xilinx Alveo U250 FPGA.

## ILLIXR Host Worker

As documented in the XRSight paper, we implement a target-to-host bridge to emulate an on-device GPU that is capable of generating both the render result (for downstream processing) as well as reflect the performance impact of the render operation in the target simulation. This method can be extended to offload other black-box computation to the host. 

<img width="435" height="291" alt="image" src="https://github.com/user-attachments/assets/6e55fff9-15d1-4c48-b18d-ae5a259def49" />

In order to do so, we run a bare-bones version of the ILLIXR runtime on the host that listens for rendering requests (or other tasks) from the host, generates the result, and communicates it back over the bridge. To build it:

Install dependencies:
```
sudo apt-get install build-essential libglew-dev libglu1-mesa-dev libsqlite3-dev libx11-dev libgl-dev pkg-config libopencv-dev libeigen3-dev libc6-dev libspdlog-dev libboost-all-dev git cmake cmake-data ccache libglfw3-dev libglm-dev libjpeg-dev libusb-1.0.0-dev libuvc-dev libopenxr-dev libopenxr1-monado libpng-dev libsdl2-dev libtiff-dev udev libudev-dev libwayland-dev wayland-protocols libx11-xcb-dev libxcb-glx0-dev libxcb-randr0-dev libxkbcommon-dev
```

```
cd ILLIXR
git checkout origin/host-worker
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=<path to home dir>/illixr-deps/  -DYAML_FILE=profiles/gpu_offload.yaml -DCMAKE_BUILD_TYPE=Release
cmake --build . -j32
cmake --install .
```

Following this, be sure to add the following to your `.bashrc` and source it. 

```
export PATH="<path to home dir>/illixr-deps/bin:$PATH"
export LD_LIBRARY_PATH="<path to home dir>/illixr-deps/lib:$LD_LIBRARY_PATH"
```

To test that the host worker was installed correctly:
```
main.opt.exe --yaml=<path to installation>/ILLIXR/illixr.yaml
```

## ILLIXR Target Workload

The next step is to build the target workload.

### Setup Chipyard repository:
```
cd xrsight-chipyard
./build-setup.sh -s 4 
```

### Build workload using Firemarshal:
```
cd xrsight-chipyard/software/firemarshal
```
Modify `rootfs-size` key in `boards/firechip/base-workloads/ubuntu-base.json` to 16 GB. 

Edit `boards/default/distros/ubuntu/Makefile` to modify the first line:

`RAWURL=https://cdimage.ubuntu.com/releases/22.04/release/ubuntu-22.04.5-preinstalled-server-riscv64+unmatched.img.xz`

and Line 8:

`skip=235520` 

Then run:

`./marshal -v build ubuntu-base.json`

This step will take several minutes as the base image is downloaded, unpacked, Linux kernel is built, etc. During this process, there may begin a Linux boot in Qemu (e.g. you see a login prompt). If this occurs, you can safely exit Qemu (e.g. `ctrl a + x`).

Then, we can boot the image with the following:

`./marshal -v launch ubuntu-base.json`

The login is `root:firesim`. Run the following commands once logged in, to fix sudo access, disable some unnecessary services, and speed up the Firesim boot:

**Note that the following are all run in the interactive Qemu session from the Firemarshal launch!**

```
pkexec chown root /etc/sudo.conf 
pkexec chown root /usr/libexec/sudo/sudoers.so
pkexec chown root /etc/sudoers
pkexec chown root /etc/sudoers.d
pkexec chown root /etc/sudoers.d/90-cloud-init-users
pkexec chown root /etc/sudoers.d/README

sudo systemctl disable systemd-networkd-wait-online.service && sudo systemctl mask systemd-networkd-wait-online.service && sudo systemctl stop snapd.seeded.service
sudo systemctl stop snapd.service && sudo systemctl disable snapd.service && sudo systemctl mask snapd.service
sudo systemctl stop snapd.socket && sudo systemctl disable snapd.socket && sudo systemctl mask snapd.socket 
sudo systemctl stop snapd.seeded.service && sudo systemctl disable snapd.seeded.service && sudo systemctl mask snapd.seeded.service
```

Install dependencies:

```
sudo apt update
sudo apt-get install build-essential libglew-dev libglu1-mesa-dev libsqlite3-dev libx11-dev libgl-dev pkg-config libopencv-dev libeigen3-dev libc6-dev libspdlog-dev libboost-all-dev git cmake cmake-data ccache libglfw3-dev libglm-dev libjpeg-dev libusb-1.0.0-dev libuvc-dev libopenxr-dev libopenxr1-monado libpng-dev libsdl2-dev libtiff-dev udev libudev-dev libwayland-dev wayland-protocols libx11-xcb-dev libxcb-glx0-dev libxcb-randr0-dev libxkbcommon-dev
```

Install OpenBLAS (configured to use the Gemmini Systolic array backend for VIO BLAS operations):

```
git clone https://github.com/pcg108/OpenBLAS.git
cd OpenBLAS
git checkout 0a1096285fd9daaf21b0ae25bef375ca8e91fc81
mkdir build && cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local/
cmake --build . -j4 && cmake --install .
```

Install ILLIXR Target Worker:

```
git clone https://github.com/pcg108/ILLIXR.git
git checkout origin/target-worker
mkdir build && cd build
## TODO: full zip with eye data
cmake .. -DCMAKE_INSTALL_PREFIX=/usr/local/ -DYAML_FILE=profiles/illixr_client.yaml -DCMAKE_BUILD_TYPE=Release

# update ~/.bashrc to include:
export PATH="/usr/local/bin:$PATH"
export LD_LIBRARY_PATH="/usr/local/lib:$LD_LIBRARY_PATH"

cmake --build . -j16 && cmake --install .
poweroff -f
```

### Modify OpenSBI to enable the RISC-V Hardware Performance Monitor counters for workload performance analysis:
**NOTE: This step will break the Firemarshal launch, as Qemu is not configured to use the RISC-V HPMs. If you desire to modify/recompile the target workload, rebuild the kernel without these lines, make the modifications, and then rebuild it with these lines to ensure the counters work in Firesim.** 

Open `boards/default/firmware/opensbi/platform/generic/platform.c`:

Add the following lines in the function:
```
static int generic_final_init(bool cold_boot){

  sbi_printf("Setting mcounteren and mhpevent3\n");
  csr_write(CSR_MCOUNTEREN, 0xFFFFFFFF);                                
  csr_write(CSR_SCOUNTEREN, 0xFFFFFFFF);  
  csr_write(CSR_MCOUNTINHIBIT, 0x0);
  
  // setup for artifact evaluation
  csr_write(CSR_MHPMEVENT3, 1 << 8 | 0x2); // i cache miss
  csr_write(CSR_MHPMEVENT4, 1 << 9 | 0x2); // d cache miss
  csr_write(CSR_MHPMEVENT5, 1 << 8 | 0x1); // load interlock
  csr_write(CSR_MHPMEVENT6, 1 << 9 | 0x1);  // long latency interlock
  csr_write(CSR_MHPMEVENT7, 1 << 17 | 0x1); // int multiplication interlock
  csr_write(CSR_MHPMEVENT8, 1 << 18 | 0x1); // fp interlock
  csr_write(CSR_MHPMEVENT9, 1 << 13 | 1 << 14 | 0x1);  // branch misprediction
  csr_write(CSR_MHPMEVENT10, 1 << 15 | 1 << 16 | 0x1); // pipeline flush
  
  
  uint64_t start_hpmc3  = csr_read(CSR_MHPMCOUNTER3);
  volatile int x = 0;
  for (volatile int i = 0; i < 100; i++) {
  x += i;
  }
  uint64_t end_hpmc3  = csr_read(CSR_MHPMCOUNTER3); 
  
  sbi_printf("test read of event counter 3: %ld, %ld after adds: %d\n", start_hpmc3, end_hpmc3, x);

...
}
```

Also include the following libraries at the top of the file to enable the prints:
```
#include <sbi/sbi_console.h>
#include <sbi/riscv_encoding.h>
```

Rebuild the kernel with `./marshal -v build ubuntu-base.json`.

Install the Firemarshal workload: `./marshal -v install ubuntu-base.json`.

## Build Target SoC bitstream for Firesim

Next, we need to build the sample SoC, which is a Rocket CPU with an attached FP32 Gemmini systolic array accelerator for VIO BLAS matmul acceleration. It also includes the infrastructure to enable the target-to-host graphics bridge.

```
cd xrsight-chipyard/sims/firesim
```

Follow the Firesim Local FPGA system setup [instructions](https://docs.fires.im/en/latest/Local-FPGA-Initial-Setup.html).

Follow the Xilinx Alveo U250 system setup [instructions](https://docs.fires.im/en/latest/Getting-Started-Guides/On-Premises-FPGA-Getting-Started/Initial-Setup/Xilinx-Alveo-U250.html)
Stop once the **Running a Single-Node Simulation** section is reached.

### Build the bitstream:

Modify `deploy/config_build.yaml`, to include a `default_build_dir`. 

Set the `builds_to_run` and `agfis_to_share` to `alveo_u250_firesim_rocket_fp32_gemmini`.

Then run:
```
firesim buildbitstream -a ${CY_DIR}/sims/firesim-staging/sample_config_hwdb.yaml -r ${CY_DIR}/sims/firesim-staging/sample_config_build_recipes.yaml
```

This will take several hours, following which instructions will be printed to add the bitstream to `${CY_DIR}/sims/firesim-staging/sample_config_build_recipes.yaml`. Please do so, mirroring the other example entries in that file.

## Run the XRSight Simulation

First, we need to flash the FPGA with the bitstream and the Ubuntu base image:

```
cd xrsight-chipyard/sims/firesim
source ~/.ssh/AGENT_VARS
source sourceme-manager.sh
```

Modify `${CY_DIR}/sims/firesim/deploy/config_runtime.yaml` to set:
```
default_hw_config: alveo_u250_firesim_rocket_fp32_gemmini

and

workload_name: ubuntu-base.json
```

Then run:
```
firesim infrasetup -a ${CY_DIR}/sims/firesim-staging/sample_config_hwdb.yaml -r ${CY_DIR}/sims/firesim-staging/sample_config_build_recipes.yaml
```

Once this step completes, open another terminal, and start the ILLIXR host worker:

```
cd ILLIXR/build
main.opt.exe --yaml=<path to installation>/ILLIXR/illixr.yaml
```

In another terminal, start the Firesim simulation:
```
firesim runworkload -a ${CY_DIR}/sims/firesim-staging/sample_config_hwdb.yaml -r ${CY_DIR}/sims/firesim-staging/sample_config_build_recipes.yaml
```

In another terminal, a screen session will appear with the interactive simulation. It can be attached to with 
```
screen -r fsim0
```

The Linux boot process will begin in the Firesim simulation. Login with the same credentials as before: `root:firesim`.
Once the interactive terminal appears, run the following:
```
cd ILLIXR/build
export ILLIXR_EYE_TRACKING=0
main.opt.exe --yaml=/root/ILLIXR/illixr.yaml
```

This will launch the target XR workload with VIO utilizing the Gemmini accelerator. 

## Monitoring workload, saving results, and processing. 

Due to simulation infrastructure issues, the best way to save the results of the performance metrics is to save the screen buffer to a file and use the post-processing scripts in the original IISWC artifact.

While the simulation is running in the screen session, run:

```
screen -r fsim0
:scrollback 100000
```
Then, as the simulation progresses, run in the screen session:

```
:hardcopy -h
```

This will save the output of the screen buffer to the `default_simulation_dir` specified in `xrsight-chipyard/sims/firesim/deploy/config_runtime.yaml` as `hardcopy.0`. 

One can run this hardcopy function periodically to verify the simulation output. For reference, the artifact terminates the simulation after 190 calls to the openvins plugin (so one can search the file for `openvins:` and terminate the simulation at that point. To do so, `ctrl+c` out of the process where we executed `firesim runworkload` and run:

```
firesim kill -a ${CY_DIR}/sims/firesim-staging/sample_config_hwdb.yaml -r ${CY_DIR}/sims/firesim-staging/sample_config_build_recipes.yaml
```

The post-processing scripts in the IISWC artifact contain useful functions to extract the counter information per plugin, including `parse_data_file`, [here](https://github.com/pcg108/xrsight-iiswc-artifact/blob/main/parse.py).

All of the above steps are automated and built in the Artifact as mentioned above. Please reach out if there are issues in running these instructions and we are happy to help.
