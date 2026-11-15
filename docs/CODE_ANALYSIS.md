# Code analysis of this AI-deck examples tree

Read-only review of `johngmuender/fork_aideck-gap8-examples` at `322361f` (20 Oct 2026), which is John's fork of [bitcraze/aideck-gap8-examples](https://github.com/bitcraze/aideck-gap8-examples). No application source was modified. The only added file is this document.

The hardware John is evaluating is a Crazyflie 2.1 Brushless with an AI-deck 1.1 (GAP8 + NINA-W10 ESP32 + Himax HM01B0). The design being compared is NanoCockpit, described in Cereda, Giusti, and Palossi, *NanoCockpit: Performance-optimized Application Framework for AI-based Autonomous Nanorobotics*, arXiv:2601.07476v2. A line-by-line review of John's other tree, `fork_nanocockpit`, already exists as a companion note. This document is about **this** tree: the Bitcraze examples, plus the slice of NanoCockpit that Bitcraze vendored under `nanocockpit/`.

How to read the labels:

- **[Repo-stated]** A fact in this repository, with a path and line, or a fact Bitcraze wrote in a committed doc.
- **[Derived]** A conclusion from that code, from arithmetic on its constants, or from a check outside the tree (Docker Hub, a GitHub compare, a linker print from a build run for this note). Not a measurement on a drone.
- **[Opinion]** A judgment. Not something the repository proves.

Builds that were actually run are in [section 14](#14-what-was-built-in-this-environment). Nothing was flashed. No drone, radio, or AI-deck was attached.

Paper performance numbers (48 Hz closed loop, 72 Hz of 160×160, 55 ms RTT, 3.8 MAC/cycle, the abstract's error and mission-success sentences) are **not** constants in this repository. The companion review of `fork_nanocockpit` extracted them from the same PDF. This note did not re-OCR the paper body: the PDF's text streams are compressed, and a text extract only recovered the title page (authors, arXiv id `2601.07476v2`, DOI `10.48550/arXiv.2601.07476`). Where a paper number is mentioned below, it is labeled as coming from that companion reading, not as a new measurement.

---

## 1. Upstream version and fork diff

**[Repo-stated]** Local `HEAD` is `322361f18fe6e198fe4e2721e9694994f86561e3`, "Merge pull request #163 from bitcraze/rik/repo", author Rik, 2026-10-20. `origin` is `johngmuender/fork_aideck-gap8-examples`. The default branch is `master`. There are 401 commits. There is no `.gitmodules`.

**[Derived]** GitHub's compare of `bitcraze/aideck-gap8-examples` `master`...`johngmuender:master` returns `status: identical`, `ahead_by: 0`, `behind_by: 0`, same SHA `322361f`. John has not added commits on top of Bitcraze. Treat this checkout as a mirror of Bitcraze `master` on that date, not as a private patch series.

**[Repo-stated]** Annotated tags include `2022.05`, `2023.10-rc1`, `2023.10-rc2`, `2024.04-rc1`, `2025.02`, `2025.02-rc1`, `2026.10`, `2026.10.1`, `2026.10.2`. Tag `2026.10.2` is `f3cc5fb`, two documentation commits behind `HEAD` (`3ee8bf0` clone instructions, `ef2beb8` viewer-doc link). The code John would build from `master` is the same code as that tag plus those two doc commits.

**[Repo-stated]** The history that matters for this review landed in 2026:

- `8685706` moves NanoCockpit to a top-level folder, copied from upstream `idsia-robotics/nanocockpit` (the pulp-frontnet commit message names upstream SHA `741f12b`).
- `044f5d2` sets the NanoCockpit streamer frame rate to 60 Hz. Upstream Frontnet in the companion tree is 30 Hz; this copy keeps Frontnet at 30 and only changes the streamer. See `nanocockpit/examples/streamer/config.h` line 150 and `nanocockpit/examples/pulp-frontnet/config.h` line 159.
- `b6b2c7c` / `8d90067` ship a pipelined capture path in `examples/other/wifi-img-streamer`.
- `ba51f66` and the README state that the Autotiler can no longer be downloaded because GreenWaves Technologies went into liquidation in 2025.

**[Repo-stated]** What Bitcraze left out of the vendored NanoCockpit, written in `nanocockpit/README.md` lines 17–26 and `docs/examples/nanocockpit/index.md` lines 49–58: the closed-loop follow-me demo, the ROS 2 viewer, the Wireshark dissectors, and the ESP32 trace notebook. There is no STM32 firmware, no ESP-IDF project, and no GAP SDK Dockerfile in this tree. Those live in upstream NanoCockpit and in `bitcraze/aideck-esp-firmware` / `bitcraze/aideck-esp-firmware-v2`.

---

## 2. Repository layout

**[Repo-stated]** Top level:

| Path | Role |
| --- | --- |
| `README.md` | Says there are two firmware combinations, and recommends NanoCockpit. |
| `module.json` | Bitcraze module descriptor, `"environmentReqs": { "build": ["gap8"] }`. |
| `tools/build/` | `build`, `make`, `make-example`. The scripts that CI and the docs run inside `bitcraze/aideck`. |
| `lib/cpx/` | Original CPX for the GAP8: `inc/cpx.h`, `inc/com.h`, `src/cpx.c`, `src/com.c`. FreeRTOS queues, SPI master. |
| `examples/` | Original examples. GPL-3 file headers. FreeRTOS (`PMSIS_OS = freertos`) on the ones that talk CPX. |
| `nanocockpit/` | Vendored framework. Apache-2.0 (`nanocockpit/LICENSE`), except `lib/crc32.c` and `lib/crc32.h` which stay GPL-3. |
| `docs/` | The published documentation source. |
| `.github/workflows/CI.yml`, `release.yml` | What actually gets compiled, and which binaries a release contains. |

`examples/` splits into three groups. **[Repo-stated]**

- `examples/ai/classification` — MobileNetV2 binary classifier, NNTool + Autotiler.
- `examples/image_processing/FaceDetection` — Haar cascade, Autotiler.
- `examples/image_processing/simple_kernel_example` — demosaic or invert, cluster vs fabric controller.
- `examples/image_processing/image_manipulations` — resize, crop, threshold of one captured frame, fabric controller only.
- `examples/other/wifi-img-streamer` — the Wi-Fi camera streamer, the example CI treats as the product.
- `examples/other/hello_world_gap8` — CPX console print.
- `examples/other/com-test` — CI / lab smoke test. Its own header says so (`com-test.c` lines 26–29).
- `examples/other/stm_gap8_cpx` — bounce a counter to the STM32 over CPX.
- `examples/other/test_functionalities/test_camera` — one Himax frame out over JTAG semihosting.
- `examples/other/test_functionalities/uart_send_char` — one byte on UART1 toward the STM32.

`nanocockpit/` splits into `lib/` (framework), `examples/` (`coroutine`, `queue`, `queue_async`, `cpx`, `streamer`, `pulp-frontnet`), and `client/aideck_cpx_streamer/` (the host viewer). **[Repo-stated]** `nanocockpit/README.md` lines 1–7.

---

## 3. Two firmware combinations, and why they cannot be mixed

**[Repo-stated]** `README.md` lines 7–16 and `docs/development/switching-firmware.md` lines 7–16:

- **NanoCockpit.** ESP32 runs [aideck-esp-firmware-v2](https://github.com/bitcraze/aideck-esp-firmware-v2). GAP8 is flashed with JTAG. The image overwrites the GAP8 bootloader.
- **Original examples.** ESP32 runs [aideck-esp-firmware](https://github.com/bitcraze/aideck-esp-firmware). GAP8 keeps the bootloader and is flashed over the air with `cfloader`, target `deck-bcAI:gap8-fw`, or with JTAG.

The two GAP8 programs do not speak the same SPI CPX. The original stack is `lib/cpx` (10 MHz, 1022-byte packets, FreeRTOS, a copy into every queue). The NanoCockpit stack is `nanocockpit/lib/cpx` (30 MHz one way, 7.2 MHz if the GAP8 must also receive, up to 4092-byte DMA, a pointer at the image bytes). **[Repo-stated]** `docs/development/cpx.md` lines 14–24 says the NanoCockpit CPX has no route to the Crazyflie and no cfclient console.

**[Repo-stated]** A newer Crazyflie release package flashes the NanoCockpit ESP32 firmware and leaves the GAP8 bootloader in place (`docs/development/switching-firmware.md` lines 55–69). After that upgrade, `examples/` will not stream until the original ESP32 image is flashed back. NanoCockpit on the GAP8 still needs a JTAG write even if the ESP32 side arrived with the Crazyflie upgrade.

**[Opinion]** For a first afternoon on the bench, pick one combination and finish it. Flashing a NanoCockpit GAP8 image and then trying `cfloader` against the original examples fails in the way the switching page describes: the bootloader is gone, and the ESP32 router is the other one.

---

## 4. Build system and toolchain

### 4.1 Two Docker images, two SDKs

**[Repo-stated]** Original examples are built with the floating tag `bitcraze/aideck` (`tools/build/make-example`, `docs/examples/wifi-streamer.md` lines 71–81). NanoCockpit is built with `bitcraze/aideck:3.8.1` (`docs/development/flashing.md` lines 36–50, `.github/workflows/CI.yml` lines 39–57). The comment in CI says `make-example` is not used for NanoCockpit because its `numpy==1.22.3` pin does not install on that image's Python.

**[Derived]** Docker Hub, queried for this note on 15 Nov 2026, lists nine tags. The ones that matter:

| Tag | Hub `full_size` | Last updated | What it contains |
| --- | --- | --- | --- |
| `3.8.1` | 2.01 GB | 2021-07-15 | GAP SDK git tag `release-v3.8.1`, Python 3.6.9 |
| `4.22.0` | — | 2024-04-11 | predecessor of latest |
| `latest` and `4.22.0.1` | 3.84 GB | 2024-05-31 | GAP SDK `release-v4.22.0-15-ga2302650`, Python 3.8.10 |

There is no tag newer than May 2024. **[Opinion]** "Latest" here means "last image Bitcraze published", not "SDK released after the GreenWaves shutdown".

**[Derived]** Both images, after `source /gap_sdk/configs/ai_deck.sh`, provide `riscv32-unknown-elf-gcc (GCC) 7.1.1 20170509` at `/usr/lib/gap_riscv_toolchain/bin`. The compiler is not on `PATH` until that script is sourced. `make` outside the container fails immediately: `examples/other/wifi-img-streamer/Makefile` line 74 includes `$(RULES_DIR)/pmsis_rules.mk`, and `RULES_DIR` is empty, so Make looks for `/pmsis_rules.mk`.

**[Repo-stated]** `tools/build/make-example` lines 12–15 exports `GAPY_OPENOCD_CABLE=interface/ftdi/olimex-arm-usb-tiny-h.cfg`, runs `pip3 install numpy==1.22.3`, sources `ai_deck.sh`, and calls `make`. `docs/development/jtag-programmer.md` lines 9–17 names the Olimex ARM-USB-TINY-H plus the ARM-JTAG-20-10 adapter. J-Link is mentioned as tested. ST-Link is stated not to work.

**[Derived]** Inside `bitcraze/aideck:latest`, that numpy pin installed (it replaced numpy 1.24.3). pip also printed `imageio 2.17.0 has requirement pillow>=8.3.2, but you'll have pillow 7.2.0`. The Wi-Fi streamer still linked. The pillow mismatch is in the image's host Python, not in the GAP8 link.

### 4.2 PMSIS, FreeRTOS, and pulp-os

**[Repo-stated]** The CPX examples set `PMSIS_OS = freertos` (`examples/other/wifi-img-streamer/Makefile` lines 1–2, and the same two lines in `hello_world_gap8`, `com-test`, `stm_gap8_cpx`, classification, and FaceDetection). CI's artifact path is `BUILD/GAP8_V2/GCC_RISCV_FREERTOS/` (`.github/workflows/CI.yml` line 60, `release.yml` line 46).

**[Repo-stated]** `simple_kernel_example/Makefile` line 8 and `test_camera/Makefile` line 8 set `PMSIS_OS ?= pulp_os`. `image_manipulations/Makefile` and `uart_send_char/Makefile` set neither.

**[Repo-stated]** NanoCockpit Makefiles do not set `PMSIS_OS`. The link line from the build in section 14 defines `__PULP_OS__` and writes `BUILD/GAP8_V2/GCC_RISCV/` (no `_FREERTOS`). `release.yml` line 47 copies that path. NanoCockpit is PMSIS on pulp-os. The original streamer is PMSIS on FreeRTOS.

**[Derived]** That is the scheduling difference the rest of this note hangs on. FreeRTOS tasks in `wifi-img-streamer.c` each have a stack (`configMINIMAL_STACK_SIZE * 2` or `* 4`, lines 257–277). NanoCockpit coroutines are functions that return, with the resume point stored in a `switch` (`nanocockpit/lib/coroutine.h`). They do not get a private stack.

### 4.3 Autotiler, NNTool, and GreenWaves

**[Repo-stated]** `README.md` lines 22–27 and `docs/development/gap8.md` lines 36–58: GreenWaves went into liquidation in 2025, greenwaves-technologies.com is gone, and `make autotiler` no longer downloads the Autotiler. Classification and face detection need it. The PULP-Frontnet example does not: its network was generated by [DORY](https://github.com/pulp-platform/dory) and the C is already in the tree.

**[Derived]** Both public images contain the Autotiler *tree* (`/gap_sdk/tools/autotiler_v3`, headers, `download_tiler.sh`, `get_tiler.py`) and zero `LibTile.a` files (`find ... -name '*.a'` returned count 0). `download_tiler.sh` saves whatever URL is in `.tiler_url` as `Autotiler/LibTile.a`. `get_tiler.py` posts the user's name and email to `https://hooks.zapier.com/hooks/catch/2624512/e6qico/` and then asks the user to paste a URL from the resulting email. **[Opinion]** That is a GreenWaves-operated signup, not a GitHub release. With the company gone, a machine that does not already have `LibTile.a` cannot build classification or face detection from these images. NNTool itself is in the image (`/gap_sdk/tools/nntool`); it is the closed Autotiler library that the generators link against.

**[Derived]** Face detection's first build step is a host link of `GenTile` against that library (`examples/image_processing/FaceDetection/Makefile` lines 56–61). Running it in `bitcraze/aideck:latest` failed with:

```text
gcc: error: /gap_sdk/tools/autotiler_v3/Autotiler/LibTile.a: No such file or directory
```

The docs tell you to use an image named `aideck-with-autotiler` (`docs/examples/classification-demo.md` line 127, `docs/examples/face-detection.md` line 25). That name is not on Docker Hub. **[Opinion]** It was a local image someone built when the download still worked. If you do not have that image or a saved `LibTile.a`, those two examples are source you can read and cannot rebuild.

### 4.4 DORY, in this tree

**[Repo-stated]** DORY is not a tool you run here. `docs/examples/nanocockpit/pulp-frontnet.md` lines 11–12 says the generated code and the weights are included. The generator banner is in every layer file, for example `nanocockpit/examples/pulp-frontnet/app/networks/frontnet-160x32-bgaug/src/layer0_BNReluConvolution.c` lines 22–51 (`sdk gap_sdk`, `x_h 96`, `x_w 160`, `double_buffering 1`). `network.mk` lines 20–22 and 41–55 compile that C, set `CORE ?= 8`, `RAM_TYPE ?= HYPERRAM`, `FLASH_TYPE ?= HYPERFLASH`, and put the `.hex` weights in the read-only filesystem.

**[Repo-stated]** `app/app.mk` lines 4–5 demotes a list of warnings to non-errors (`-Wno-error=int-conversion`, `discarded-qualifiers`, and others) so this generated C can compile under the example's `-Werror`. The build in section 14 printed those warnings and still linked.

### 4.5 What CI and the release actually build

**[Repo-stated]** `.github/workflows/CI.yml`:

- `tools/build/build`, which is the Wi-Fi streamer with `SETUP_WIFI_AP=1 CAPTURE_MODE=pipelined CAMERA_RESOLUTION=qvga STREAM_ENCODING=raw SENSOR_FRAME_RATE=default` (`tools/build/build` line 7).
- `test_camera`, `uart_send_char`, `com-test`, via `make-example`.
- Every NanoCockpit example, on `bitcraze/aideck:3.8.1`, including `pulp-frontnet`.

CI does not build classification, face detection, the simple kernel, image manipulations, hello-world, or `stm_gap8_cpx`.

**[Repo-stated]** `release.yml` builds two binaries and attaches them to a draft GitHub release: `aideck_gap8_wifi_img_streamer_with_ap.bin` and `aideck_gap8_nanocockpit_streamer.bin`. There is no released PULP-Frontnet binary (`docs/examples/nanocockpit/pulp-frontnet.md` lines 16–18).

**[Derived]** The Makefile default for the Wi-Fi streamer is not the release. `Makefile` lines 6–12 default to `CAPTURE_MODE=start-stop`, `SETUP_WIFI_AP` unset, `SENSOR_FRAME_RATE=default`. The doc's "default configuration" paragraph (`docs/examples/wifi-streamer.md` lines 42–44) describes that Makefile. The file you download from a release is the pipelined, access-point build from `tools/build/build`. Building "the example" by following the first docker command in the doc (line 73, no extra variables) produces the stop-start streamer without its own access point.

---

## 5. Each example

### 5.1 `wifi-img-streamer`

The example that a release ships, and the one the pipelined-capture work landed in.

**[Repo-stated]** `start_example` (`wifi-img-streamer.c` lines 239–287) opens UART at 115200, calls `cpxInit`, enables `CPX_F_WIFI_CTRL`, and creates three FreeRTOS tasks at `tskIDLE_PRIORITY + 1`: heartbeat (LED on GPIO 2, 500 ms), `camera_task` (stack `configMINIMAL_STACK_SIZE * 4`), and `wifi_rx_task`. `main` sets the fabric controller to 250 MHz and 1.2 V (lines 290–295). The cluster is not opened. This example does not run a network.

**[Repo-stated]** `camera_task` (lines 147–219) waits 2 seconds, optionally sends the Wi-Fi setup packets, initialises the camera, and then loops: if no Wi-Fi client is connected, it calls `camera_pipeline_disconnect` and delays 10 ms; otherwise it takes a frame, optionally JPEG-encodes it, sends it with `cpx_stream_*`, then calls `camera_pipeline_end_frame`.

Build switches, all in `Makefile` lines 3–68 and `stream_config.h`:

| Variable | Effect |
| --- | --- |
| `CAMERA_RESOLUTION=qvga` | `PI_CAMERA_QVGA`, 324×244, and `CAMERA_QVGA_WINDOW_ENABLE=1` |
| `qqvga` | `PI_CAMERA_QQVGA`, 162×122 |
| `native` | format stays QVGA, size 324×324, and `SENSOR_FRAME_RATE=60` is rejected |
| `CAPTURE_MODE=start-stop` | one buffer, sensor started and stopped around each frame (`CAPTURE_MODE` 0) |
| `CAPTURE_MODE=pipelined` | three buffers (`CAPTURE_BUFFER_COUNT` 3), sensor left running (`CAPTURE_MODE` 2) |
| `SENSOR_FRAME_RATE=60` | writes Himax frame length 266 lines and max integration 264 lines |
| `STREAM_ENCODING=jpeg` | GAP JPEG encoder, 15 KB output buffer |
| `OUTPUT_PROFILING_DATA=1` | one cfclient console line per frame |
| `SETUP_WIFI_AP=1` | GAP8 tells the ESP32 to become an access point named `WiFi streaming example` |

**[Repo-stated]** The doc says to pass `clean` whenever those variables change, because the build does not track compiler flags (`docs/examples/wifi-streamer.md` lines 84–86). That matches the Makefile: the flags are `APP_CFLAGS +=`, and there is no dependency from the object files on the flag set.

The pipeline itself is section 6. The CPX send path is section 8. Two bugs in `setup_wifi` are in section 13.

### 5.2 `hello_world_gap8`

**[Repo-stated]** `hello_world_gap8.c` lines 30–44: `cpxInit`, then `cpxPrintToConsole(LOG_TO_CRTP, "Hello World\n")` once a second. FreeRTOS. No camera. The doc (`docs/examples/hello-world.md`) expects the line `CPX: GAP8: Hello World` in the cfclient console. This is the smallest proof that the original ESP32 firmware, the bootloader, and the radio path agree.

### 5.3 `com-test`

**[Repo-stated]** Header at `com-test.c` lines 26–29: the application used for testing in CI and the lab. It is the binary CI uploads as `aideck_gap8.bin` (`.github/workflows/CI.yml` lines 59–66). It is not a demo to fly.

### 5.4 `stm_gap8_cpx`

**[Repo-stated]** `stm_gap8_cpx.c` lines 34–53: enable `CPX_F_APP`, block on a packet, read `data[0]`, and send the same byte back to `CPX_T_STM32`. The matching STM32 app is `app_stm_gap8_cpx` in crazyflie-firmware, which is not in this repo (`docs/examples/stm_gap8_cpx.md` lines 11–30). This is the only original example whose job is GAP8 ↔ STM32 traffic, and it uses CPX through the ESP32, not the UART that NanoCockpit's Frontnet uses.

### 5.5 `test_camera` and `uart_send_char`

**[Repo-stated]** `test_camera` is pulp-os, async capture, QVGA, optional color (`test_camera/Makefile` lines 8–12). The doc runs it with `make clean all run io=host` over JTAG and writes a PPM (`docs/examples/test-camera.md` lines 16–18). It says non-QVGA capture "might not always work correctly".

**[Repo-stated]** `uart_send_char` writes a character on UART1 for the Crazyflie to read (`docs/examples/uart-send-char.md` lines 8–11). The doc tells you to force-compile the AI-deck driver in the Crazyflie firmware. There is no Wi-Fi and no CPX in this example.

### 5.6 `simple_kernel_example` and `image_manipulations`

**[Repo-stated]** The simple kernel reads one image and either demosaics or inverts it, on the fabric controller or on the cluster, one kernel at a time (`simple_kernel_example/Makefile` lines 18–34, `README.md` lines 1–9). The cluster path splits rows across cores (`img_proc.c` lines 91–117, `pi_core_id()`, `nPE`). Output goes back over JTAG semihosting, not Wi-Fi.

**[Repo-stated]** The only performance numbers in the whole `examples/` tree that are written down as measurements are in that README, and they are for a 50 MHz fabric controller and cluster, on an older revision of the demosaic: 2.33 s with floating-point coefficients, 303.3 ms with three integer divisions, 199.2 ms with one division, 33.3 ms once the work is split across 8 cores. The README says the example was developed with GAP SDK 3.8.1. **[Opinion]** Those numbers are a teaching trace of "GAP8 has no FPU and the cluster pays off when the work is the same on every core". They are not a camera frame rate, and they are not from the 250 MHz FreeRTOS images.

**[Repo-stated]** `image_manipulations/test.c` lines 22–29 allocates four L2 buffers (324×244 input, a resized image, a 28×28 crop, a binary image) and runs on the fabric controller. `Makefile` line 6 defines `ASYNC_CAPTURE` and does not open a cluster.

### 5.7 Classification

**[Repo-stated]** A fine-tune of MobileNetV2, alpha 0.35, 96×96×3, with extra layers so the 324×244 grayscale (or Bayer) AI-deck frame can be the input (`docs/examples/classification-demo.md` lines 19–21). The GAP8 side is `classification.c`. One L2 camera buffer, 324×244 (lines 39–40, 184). Fabric controller 200 MHz (`FREQ_FC` in the Makefile line 49, applied at `classification.c` line 148). The cluster clock line is commented out (line 149), so the cluster stays at whatever the SDK leaves it at, not the `FREQ_CL=170` the Makefile computes for the Autotiler memory budget.

**[Repo-stated]** Capture is start, one DMA, stop, then `pi_cluster_send_task_to_cl` inside the camera callback (lines 71–89). The next capture is armed only after inference and the console print. One buffer, no overlap of the sensor with the cluster.

**[Repo-stated]** Memory budget passed to the generator (`Makefile` lines 42–47): cluster master stack 6096, slave stack 1024, times 7 slaves, subtracted from 60000 to get `MODEL_L1_MEMORY`. **[Derived]** 6096 + 7×1024 = 13264, so L1 left for the tiling is 46736 bytes, under the 64 KB cluster L1 the GAP8 page describes (`docs/development/gap8.md` lines 18–20). `MODEL_L2_MEMORY=270000`. `MODEL_L3_CONST=hflash`, so weights live in HyperFlash, not L2 (`Makefile` lines 54–55). `FS_PARTITIONTABLE_OFFSET=0x40000` (line 68) keeps the bootloader slot so the image can be flashed over the air.

**[Repo-stated]** The two class scores are signed 16-bit (`classification.c` lines 192, 79–86). The printed "confidence" is the difference of the two scores. There is no timing print around `RunNetwork`.

Training and quantization are section 9. The example cannot be linked today without `LibTile.a` (section 4.3).

### 5.8 Face detection

**[Repo-stated]** A GreenWaves Haar cascade, not a CNN (`docs/examples/face-detection.md` lines 14–16). Tested, the doc says, on GAP SDK 4.22.0. `main.c` captures 324×244, start/stop, one buffer (lines 360–375). The cluster is opened and then pinned to 75 MHz (lines 335–340). The comment on line 339 says the maximum (175 MHz) causes more camera failures, cause unknown. The fabric controller is 250 MHz and 1.2 V (lines 468–470).

**[Repo-stated]** The detector writes a 64×48 image (`IMAGE_OUT_WIDTH` / `HEIGHT`, lines 48–49, allocated at line 277). Integral images of that same 64×48 are allocated as `unsigned int` (lines 278–279): 64×48×4×2 = 24576 bytes plus the 3072-byte output. The full frame is 79056 bytes. All of this is `pmsis_l2_malloc`.

**[Repo-stated]** `faceDet` forks the cluster with `pi_cl_team_fork(gap_ncore(), ...)` (`FaceDetBasicKernels.c` line 368). `gap_ncore()` is the SDK's core count. The Makefile does not pass `NUM_CORES`.

**[Repo-stated]** With `USE_STREAMER` (on by default, Makefile line 34) the task sends a CPX image whose header width and height are the full 324×244 (`main.c` lines 126–133) and whose byte count is `STREAM_W * STREAM_H` = 160×120 (lines 385–388). The pointer it sends is `ImageOut`, which is 64×48. Section 13. The doc says you must also edit `opencv-viewer.py` to match `STREAM_W` (`docs/examples/face-detection.md` lines 49–53). The current viewer reads width and height from the packet (`opencv-viewer.py` lines 87–112) and does not have a resolution constant to edit. The doc and the viewer have drifted.

**[Repo-stated]** After the optional stream, one byte, `ClusterCall.num_response`, is written on the UART with no header (`main.c` line 392).

This example also fails at `LibTile.a` (section 14).

### 5.9 NanoCockpit examples

**[Repo-stated]** `docs/examples/nanocockpit/component-examples.md`:

- `coroutine` — several tasks on `lib/coroutine.h`, runs until Ctrl-C over JTAG.
- `queue` and `queue_async` — the queue API, and a producer/consumer pair. They exit on their own.
- `cpx` — two small packets to the host every half second. `CPX_SPI_BIDIRECTIONAL` is **on** (`examples/cpx/config.h` line 43). The other examples leave it commented out.
- `streamer` — camera to Wi-Fi, no network. Default HALF format, 60 Hz (`streamer/config.h` lines 110 and 150).
- `pulp-frontnet` — the same camera path plus the DORY network. Default HALF format, 30 Hz (`pulp-frontnet/config.h` lines 122 and 159). The estimate is also written on UART1. The Crazyflie firmware in this repository does not exist, and the doc says the stock firmware does not consume that UART message (`pulp-frontnet.md` lines 45–49).

**[Repo-stated]** Viewer: `nanocockpit/client/aideck_cpx_streamer/pyproject.toml` requires Python `>=3.14`, matplotlib 3.11, numpy 2.5, opencv-python-headless 5. The doc uses `uv sync` then `uv run plt_viewer`, joining Wi-Fi `aideck-nanocockpit` and connecting to `aideck.local` (`docs/examples/nanocockpit/streamer.md` lines 45–64).

---

## 6. Camera acquisition

The sensor in all of these examples is the Himax HM01B0 on the AI-deck, 8-bit, one byte per pixel. **[Repo-stated]** `docs/development/gap8.md` lines 28–33: the GAP8's camera parallel interface (CPI) feeds the micro-DMA, so the fabric controller does not copy the pixels.

### 6.1 Formats

**[Repo-stated]** NanoCockpit's table, in both `streamer/config.h` lines 60–81 and `pulp-frontnet/config.h` lines 72–93:

| `HIMAX_FORMAT` | Name | Size | Max rate written in the comment |
| --- | --- | --- | --- |
| 0 | FULL | 324×324 | 45 fps |
| 1 | QVGA | 324×244 | 60 fps |
| 2 | HALF | 162×162 | 90 fps |
| 3 | QQVGA | 162×122 | 120 fps |

HALF and QQVGA see the full field of view and bin it. The comment's "max" is not the same table as upstream's measured `himax_supported_framerates.md`, which this repo only links (`streamer/config.h` lines 147–149, `docs/examples/nanocockpit/streamer.md` lines 33–35).

**[Repo-stated]** Both NanoCockpit apps use format 2, MCLK mode, 6 MHz, divider /1, auto-exposure off, integration 10 ms, analog gain `0x10` (the comment says 1× to 16×; `himax.c` lines 207–211 decodes bits [6:4] as a power of two, so `0x10` is 2×). Fabric controller 246 MHz, cluster 175 MHz, 1.2 V (`config.h` lines 62–64 and 121–159).

**[Repo-stated]** Crop for format 2 is 1 pixel on each side (`config.h` lines 174–177). Capture height drops the bottom margin before the DMA (`config.h` lines 277–283):

- capture 162 × 161 = 26082 bytes
- crop 160 × 160

Two buffers are 52164 bytes. They are allocated inside the streamer payload (`nanocockpit/lib/streamer.c` around the `pi_l2_malloc` at line 137), not as a separate camera malloc.

**[Repo-stated]** The original Wi-Fi streamer does not crop. QVGA DMA is `CAMERA_WIDTH * CAMERA_HEIGHT` = 324×244 = 79056 bytes (`camera_pipeline.c` lines 92–97 and 188). The extra 4 pixels on width and height are the padding the simple-kernel README also mentions (`simple_kernel_example/Makefile` lines 14–16).

### 6.2 The original streamer's pipeline, and what it overlaps

**[Repo-stated]** `stream_config.h` lines 70–74: pipelined mode has 3 buffers, start-stop has 1.

**[Repo-stated]** `camera_pipeline_init` (`camera_pipeline.c` lines 178–197) creates a FreeRTOS event group and `pmsis_l2_malloc`s each buffer. `open_camera` (lines 112–176) opens the Himax, starts the sensor, writes orientation register `0x0101` = 3, optionally programs the QVGA window and the 266-line frame, waits for the Himax frame counter (`0x0005`) to advance by 2 with a 1 second timeout, reads the orientation back, stops the sensor, and runs `PI_CAMERA_CMD_AEG_INIT`.

**[Repo-stated]** `himax_timing.c` writes the group-hold register `0x0104` around the real writes, then read-backs frame length (`0x0340`), max integration (`0x2105`), and the QVGA window bit (`0x3010`). A mismatch returns −1. The 60 Hz build is those two constants, 266 and 264 (`Makefile` lines 41–44). The file does not read the line length or the pixel clock back, and nothing prints a measured frame rate. **[Opinion]** "60" is the name of the register set, not a fps counter.

**[Repo-stated]** Pipelined `camera_pipeline_begin_frame` (lines 233–244), first time only: mark the consumer as waiting, arm DMA on buffers 1 and 2, start the sensor, wait until one buffer completes, remember that index, arm buffer 0, and leave the sensor running. Later calls just return the buffer acquired last time.

**[Repo-stated]** `capture_done` (lines 59–87): if the consumer is waiting, publish that buffer and set the event. If the consumer is not waiting, increment `dropped_frames` and re-arm the same buffer. The comment says a replacement leaves the outstanding-DMA count unchanged, and the code does that (it does not increment `captures_completed` on the drop path).

**[Repo-stated]** `camera_pipeline_end_frame` (lines 254–266) sets `consumer_waiting`, waits for the next completion, re-arms the buffer that was just processed, and returns the wait in FreeRTOS ticks.

**[Derived]** During JPEG encode and the SPI send, `consumer_waiting` is 0, because it is set only inside `end_frame` and cleared inside `capture_done`. Any frame that finishes DMA in that window is counted as a drop and the buffer is re-armed. The three buffers do not form a queue of completed frames. They keep the sensor streaming and the DMA armed so that, when the send finishes, the next frame is one that was exposed while the previous one was in flight, and a frame is never handed to the encoder halfway through a DMA. That is what the doc says (`docs/examples/wifi-streamer.md` lines 56–62): low photon-to-host latency, not maximum throughput.

**[Derived]** This is a different pipeline from NanoCockpit's. NanoCockpit runs the consumer on buffer N while buffer N+1 is being captured, and the consumer's result is kept (`camera.c` lines 137–182: the consume step does not wait for the callback; the next loop arms the other buffer). The original streamer throws away the frames that complete during the send. Overlap exists (the sensor and the DMA stay on), and the pixels that overlap the send are the ones you do not get.

**[Repo-stated]** Start-stop mode (lines 226–231) is one buffer, `PI_CAMERA_CMD_START`, wait, `PI_CAMERA_CMD_STOP`. The sensor is off for the whole transfer. That is the Makefile default and not the release binary (section 4.5).

**[Derived]** `camera_pipeline_disconnect` (lines 268–285) sets `discarding_captures` and waits until the in-flight DMAs finish before `PI_CAMERA_CMD_STOP`. The sensor is not stopped mid-frame. `wifi-img-streamer.c` calls this when the Wi-Fi client drops (lines 174–178), so a disconnect does not leave a DMA aimed at a buffer the next client will reuse.

**[Opinion]** `consumer_waiting` is a `volatile int` read in the camera callback and written in the task, with no critical section (`camera_pipeline.c` lines 45, 72–86, 258). If the callback runs in task context, as the use of `xEventGroupSetBits` rather than the `FromISR` variant suggests (contrast `com.c` lines 84–88, which does use `FromISR`), the window is a lost wakeup: a completion that observes `consumer_waiting == 0` and then loses the race with the store of 1 will drop a frame and not wake the waiter. Another buffer is in flight, so the waiter unblocks on the next one. It costs a frame. It is not a deadlock. I did not trace the PMSIS camera ISR, which is inside the SDK, not this repo.

### 6.3 NanoCockpit's camera

**[Repo-stated]** `nanocockpit/lib/camera.c` `camera_task` (lines 128–184), one coroutine, two buffers:

1. If this buffer was used `CAMERA_BUFFERS` captures ago, wait until its consumer has finished.
2. `himax_capture_async` into that buffer, `himax_start`.
3. Wait for that same DMA, `himax_stop`, read the Himax frame counter and `time_get_us`.
4. Crop in place, yielding every row (`CAMERA_CROP_YIELD` is 1, lines 191–218).
5. Start the user callback and do not wait for it.

**[Derived]** The sensor is stopped between frames. A Himax frame that starts while the CPI is stopped is not captured. The rate in `HIMAX_FRAME_RATE` and the rate at which the fabric controller re-arms DMA are different knobs. Two buffers hide the consumer's work (inference, SPI) behind the next exposure. They do not hide the crop, because the crop of frame N finishes before frame N+1 is armed.

**[Repo-stated]** `himax.c` lines 183–188 and 231–233: `compute_frame_len_lines` is declared `(format, line_len_pck, frame_rate)` and called as `(format, desired_frame_rate, line_len_pck)`. The arguments are swapped. **[Derived]** For an integer frame rate the product `frame_len_lines * line_len_pck` still equals `pixel_clock / fps`, so 30 Hz and 60 Hz still come out as those rates. `frame_len_lines` is not "the number of lines" a reader of the datasheet expects. A non-integer rate is truncated by the `uint16_t` parameter. The companion review of the upstream tree reached the same conclusion; the function text in this vendored copy is the same.

**[Repo-stated]** `himax_start` and `himax_stop` cast the SDK device private pointer and call `pi_cpi_control_start` / `pi_cpi_control_stop` directly (`nanocockpit/lib/camera/himax.c` lines 388–400). Both comments say GAP SDK 3.8+ also writes `HIMAX_MODE_SELECT` inside `PI_CAMERA_CMD_START` / `STOP`, and that this bypass is looking for a better way. **[Opinion]** That is why this tree pins image `3.8.1` and why pointing the NanoCockpit Makefiles at `bitcraze/aideck:latest` (SDK 4.22) is a port, not a retag.

### 6.4 Frame rates you can point at

Nothing in `examples/` prints a measured frames per second. The Wi-Fi streamer's profiling line (`wifi-img-streamer.c` lines 211–217) prints FreeRTOS tick deltas named `q_ms`, `enc_ms`, `tx_ms`, `frame_ms`, `wait_ms`, plus the drop counter. **[Derived]** The tick period is the SDK's `configTICK_RATE_HZ`, which is not in this repo. The numbers are in ticks. The viewer prints `meanTimePerImage` and `1/meanTimePerImage` (`opencv-viewer.py` lines 105–108), which is host receive rate, after Wi-Fi.

**[Repo-stated]** The NanoCockpit viewer computes fps from GAP8 timestamps in the stream metadata (`client/.../cpx/streamer.py` lines 401–414) and prints dropped frames from the Himax frame counter. That is a real rate, on the host, once a radio link to the deck's Wi-Fi exists. This note did not run the viewer.

**[Repo-stated]** Checked-in sensor targets: original release streamer, "default" Himax timing (not the 266-line table); NanoCockpit streamer, 60 Hz HALF; PULP-Frontnet, 30 Hz HALF. The paper's 48 Hz closed loop and 150 frame/s figures are not in this source. The companion note's reading of the PDF is the place those sentences are written down.

---

## 7. Cluster, cores, and where the bytes live

**[Repo-stated]** `docs/development/gap8.md` lines 11–24, Bitcraze's own description of the chip:

- Fabric controller: peripherals, its own L1 (16 KB), instruction cache.
- Cluster: 8 cores, shared L1 (64 KB), one instruction cache. Useful when every core runs the same code on different data.
- L2: 512 KB, shared. The only memory the core executes from.
- External HyperBus chip: L3 RAM (the page says 64 Mb) and L3 flash. The page's flash bullet is unfinished ("of L3 flash").

**[Repo-stated]** Original examples that open the cluster:

| Example | When | Clock the source sets | What the cores do |
| --- | --- | --- | --- |
| classification | after each frame, blocking | FC 200 MHz; cluster set is commented out | generated CNN, one `pi_cluster_task`, stacks 6096 / 1024 |
| face detection | after each frame, blocking | FC 250 MHz, cluster 75 MHz | Haar cascade, `pi_cl_team_fork(gap_ncore())` |
| simple kernel | once, on a file | not set in the example | row split across `nPE` cores |

The Wi-Fi streamer, hello world, com-test, stm_gap8, and uart example never call `pi_cluster_open`.

**[Repo-stated]** PULP-Frontnet asks for 8 cores (`network.mk` lines 20–28, `-DNUM_CORES=$(CORE)`). `network_run_async` (`network.c` lines 111–131) builds a cluster task with master stack 3500 and slave stack 3400 and sends it with `pi_cluster_send_task_to_cl_async`. `execute_layer_fork` (lines 378–416) does `pi_cl_team_fork(NUM_CORES, layerN, ...)` for nine layers. Core 0 allocates 36700 bytes of L1 for the tile buffer (lines 381–382) and frees it after the layer.

**[Repo-stated]** `dory_dma.c` lines 121–129: multi-core 3D DMA deadlocks on GAP8, so `SINGLE_CORE_DMA` forces the 3D copy onto core 0. The 8 cores share the compute. They do not share the DMA.

**[Repo-stated]** Weights are not in L2 during inference. `network.c` lines 52 and 76–101 copies 311088 bytes from the read-only filesystem into HyperRAM. The L2 working set is `NETWORK_L2_BUFFER_SIZE` 352000 (`network.h` line 30). The linker, on the binary built in section 14, placed 84128 bytes in L2 (16.05% of 512 KB) for code and static data. **[Derived]** The 352000-byte buffer and the two 26082-byte camera buffers are `pi_l2_malloc` at boot, out of the remaining heap, about 404 KB on top of that 84 KB. 512×1024 − 84128 − 404164 leaves on the order of 35 KB for the CPX buffer, stacks, and the UART. The companion review's conclusion holds for this copy: a third camera buffer, or a larger crop, fails the malloc. `main.c` aborts if that malloc fails.

**[Repo-stated]** After layer 0 the cluster posts `input_done` back to the fabric controller (`network.c` lines 292–306). Frontnet passes the camera buffer's done event as that argument (`main.c` lines 111–116 and 146) and waits on it before returning the buffer (`main.c` lines 129–130). **[Derived]** The CPI can refill that buffer while layers 1–8 still run. That is the overlap the paper draws. It is in this tree. It is also the overlap the original Wi-Fi streamer does not implement: that streamer has no cluster, and it drops frames instead of computing on them.

**[Repo-stated]** Classification's L2 budget for the generator is 270000 bytes, with weights in HyperFlash (`Makefile` lines 46 and 54–55). Face detection keeps the 324×244 frame, a 64×48 output, and two 64×48 integral images in L2, and does not use HyperRAM. The simple kernel's buffers are `pmsis_l2_malloc` of the whole frame, times two or three (`test.c` lines 156–169).

**[Derived]** From the Wi-Fi streamer link map (`wifi-img-streamer.map` line 4433), `__heapl2ram_size` is `0x70548` = 460104 bytes. Static L2 from `riscv32-unknown-elf-size` on that ELF: `.text` 49384, `.rodata` 1800, `.data` 2016, `.bss` 10812. The three QVGA buffers are 3 × 79056 = 237168 bytes. The SPI transmit queue is 80 packets × 1024 bytes = 81920 (`com.h` line 27, `com.c` lines 69–71). Together that is about 319 KB of the 460 KB heap, before FreeRTOS task stacks. A pipelined native 324×324 build is 3 × 104976 = 314928, plus the 80 KB queue, and is the configuration most likely to fail `pmsis_l2_malloc` at boot. JPEG adds a 15 KB encode buffer (`wifi-img-streamer.c` lines 110–115) on top.

---

## 8. CPX and the Wi-Fi path

### 8.1 Original CPX, GAP8 side

**[Repo-stated]** Routing (`lib/cpx/inc/cpx.h` lines 33–50): targets STM32, ESP32, Wi-Fi host, GAP8. Functions include system, console, CRTP, Wi-Fi control, app, test, bootloader. Version is 0. The on-wire header is a 2-byte length plus a packed routing byte (`cpx.c` lines 28–41). `MTU` is 1022 (`com.h` line 27), so the data array in `CPXPacket_t` is 1020 bytes.

**[Repo-stated]** SPI (`com.c` lines 128–148): SPI1, chip select 0, mode 0, 8-bit, **10 MHz**, big-endian. GAP8 is the master. Flow control is two GPIOs: NINA ready-to-talk on GPIO A18, GAP8 request on GPIO A3 (lines 46–58). The same pads are named in the NanoCockpit config (`streamer/config.h` lines 203–208).

**[Repo-stated]** `com_task` (lines 156–287) waits until either the transmit queue or the NINA GPIO is pending, raises the GAP8 GPIO, transfers 4 bytes (length + routing), pads the remainder up to a multiple of 4, and lowers the GPIO before the second transfer. A zero-length transfer is bumped to 4 bytes because a zero-length SPI call corrupts the next one (lines 248–255). Received packets with a non-zero length are queued. `RXQ_SIZE` is 5, `TXQ_SIZE` is 80 (lines 70–76). The comment says the transmit queue should hold at least one image. **[Derived]** 80 × 1020 bytes of payload is 81600 bytes, which covers one 79056-byte QVGA frame and does not cover one 104976-byte native frame. The queue stores full `packet_t` objects (1024 bytes each), so the native frame blocks the producer mid-image while the queue drains. That is legal; it is also the opposite of a zero-copy send.

**[Repo-stated]** `cpx_rx_task` (`cpx.c` lines 55–75) blocks in `com_read` and `xQueueSend`s to a per-function queue of length 2 (line 43). `cpxSendPacket` (lines 97–99) returns `true` and does not send. Every real send goes through `cpxSendPacketBlocking`, which copies the packet again into `txpPacked` (lines 86–95). `cpxPrintToConsole` takes a binary semaphore around that copy (lines 102–119).

**[Repo-stated]** `cpxInit` line 134: `memset(queues, CPX_F_LAST, sizeof(xQueueHandle))`. The size argument is one pointer, not the array. **[Derived]** Four bytes at `queues[0]` become `0x0F0F0F0F`. Functions are numbered from 1 (`cpx.h` lines 41–49), and the rest of the array is BSS zero, so the "no queue" branch on line 72 still runs for real functions. Index 0 is a garbage pointer. Nothing in these examples sends function 0.

**[Repo-stated]** The streamer copies every chunk (`cpx_stream.c` lines 61–75). Header is magic `0xBC`, width, height, depth 1, type, size, packed (`cpx_stream.c` lines 30–38). Route is GAP8 → Wi-Fi host, function `CPX_F_APP` (lines 42–44).

### 8.2 Original Wi-Fi, and the viewer

**[Repo-stated]** The ESP32 firmware is not in this repo. The GAP8 asks it to become an access point by two `CPX_F_WIFI_CTRL` packets: `WIFI_CTRL_SET_SSID` then `WIFI_CTRL_WIFI_CONNECT` with `data[0] = 0x01` (`wifi-img-streamer.c` lines 73–93, `wifi.h` lines 5–18). There is no `WIFI_CTRL_SET_KEY` in that function. The SSID string is `WiFi streaming example`.

**[Repo-stated]** The host viewer opens TCP `192.168.4.1:5000` (`opencv-viewer.py` lines 45–58). For each packet it reads a 4-byte header (little-endian length, routing, function) and then `length - 2` payload bytes (lines 76–102). Raw frames (`format == 0`) are reshaped to `(height, width)` and passed through `cv2.COLOR_BayerBG2BGRA` (lines 110–115). JPEG is `cv2.imdecode`. The file header (lines 27–38) still describes a JPEG start-of-frame scan; the code that runs is the CPX header parser, and it does handle raw.

**[Derived]** A mono HM01B0 frame run through a Bayer demosaic is a false-color image. The NanoCockpit viewer doc says the opposite on purpose: grayscale, no color reconstruction (`docs/examples/nanocockpit/streamer.md` lines 66–67). The original viewer always demosaics format 0.

**[Derived]** Throughput ceiling of this path, payload bits only, ignoring the GPIO handshake and Wi-Fi: one 79056-byte QVGA frame at 10 MHz is 63.2 ms of SPI, about 15 frames/s, in 78 packets of 1020 bytes plus the 11-byte header. One 160×160 frame at the same 10 MHz is 20.5 ms of SPI. The handshake around every packet (`com.c` lines 205–266) is the term that dominates, and it is not measured here. NanoCockpit's 30 MHz one-directional SPI and a 4092-byte DMA that points at the pixels (`cpx_spi.c` lines 41–47, streamer tail in `streamer.c`) is the mechanism behind the paper's higher stream rate. The 72 Hz and 55 ms figures themselves are not in this tree.

### 8.3 NanoCockpit CPX

**[Repo-stated]** `cpx_spi.c` lines 41–47: 30 MHz when `CPX_SPI_BIDIRECTIONAL` is off, 7.2 MHz when it is on. The comment attributes the cap to swapped MISO/MOSI on the AI-deck, which forces the ESP32 GPIO matrix. Frontnet and the streamer leave the define commented out (`pulp-frontnet/config.h` lines 191–196, `streamer/config.h` lines 182–187). The small `cpx` example turns it on.

**[Repo-stated]** With the define off, any bytes that come back in the full-duplex shift are discarded by forcing `receive_req->header.length = 0` (`cpx_spi.c` lines 311–316). The host viewer says so if you ask it for RTT (`streamer.py` lines 416–418): "enable CPX_SPI_BIDIRECTIONAL in GAP config.h". **[Derived]** The default Frontnet image can send the pose and the frames. It cannot receive the host's reply on SPI. The offboard-inference path in `main.c` (the `#ifndef NETWORK_ONBOARD_INFERENCE` block) is compiled out, because `NETWORK_ONBOARD_INFERENCE` is set (`config.h` line 47).

**[Repo-stated]** `cpx.c` lines 127–133, comment: "FIXME: this probably deadlocks if contended." Sends share one `send_done` event. `cpx_start` primes it with an extra `pi_task_push` (lines 110–112) because there is no mutex. **[Opinion]** The comment is the right thing to believe. The streamer and Frontnet each have one sender in the default app, which is why it has not been forced into the open.

**[Repo-stated]** There is no ESP32 router in this repo. `docs/development/cpx.md` lines 18–24 says this CPX implementation only carries GAP8 ↔ computer through the ESP32. A packet addressed to `CPX_T_STM32` is not delivered by code that lives here. Frontnet's path to the flight controller is UART1 at 115200 (`nanocockpit/lib/uart.c` lines 40–43), a different framing from CPX (`uart_protocol.h`). The closed-loop consumer of that UART is the follow-me app that this tree does not contain.

---

## 9. How the networks are trained, quantized, and deployed

### 9.1 Classification, the only network you can retrain from this repo

**[Repo-stated]** `train_classifier.py` fine-tunes MobileNetV2 and writes two files (`train_classifier.py` lines 15–20, 191–223):

1. `model/classification.tflite`, a float TFLite conversion.
2. `model/classification_q.tflite`, full integer. `converter.optimizations = [tf.lite.Optimize.DEFAULT]`, a representative dataset of 100 JPEGs from the training tree, `TFLITE_BUILTINS_INT8`, `supported_types = [tf.int8]`, input type `uint8`, output type `uint8`.

**[Repo-stated]** The GAP8 Makefile sets `MODEL_PREQUANTIZED = true` (line 22), which selects `model/nntool_script` and passes `-q` to NNTool (lines 24–29). That script (`model/nntool_script`) runs `adjust`, `fusions --scale8`, `save_state`. The other script, `nntool_script_q`, is the one that calls `aquant samples/* --scheme SQ8`, and it is used only when `MODEL_PREQUANTIZED` is false. **[Derived]** The shipped path trusts the TFLite int8 model and asks NNTool not to quantize again. The samples directory is for the other path.

**[Repo-stated]** `model_rules.mk` lines 51–74 is the chain: NNTool state file, NNTool `-g` to emit Autotiler model C, host `gcc` links that C with `$(TILER_LIB)` into `GenTile`, `GenTile` emits the GAP kernels. Step 3 is the one that needs `LibTile.a`.

**[Repo-stated]** The pretrained weights are a zip, `model/classification_tflite_files.zip`, not loose `.tflite` files. The doc says to unzip `classification.tflite` and `classification_q.tflite` into `model/` (`docs/examples/classification-demo.md` lines 117–118). `requirements.txt` pins `tensorflow==2.11.0` and `numpy==1.21.5`. The doc's Docker training line uses `tensorflow/tensorflow:2.17.0` (`classification-demo.md` lines 84–85) and says native training wants Python 3.10. **[Opinion]** The requirements file and the Docker tag are two different TensorFlow versions. Use the Docker line the doc gives, or the requirements file, and do not mix them. Neither path produces a GAP8 binary without the Autotiler.

**[Repo-stated]** The doc is frank about the model: one table, one Christmas package, weak augmentation, poor generalization (lines 14–16). Retraining on your own Wi-Fi-streamer captures is the path it recommends.

### 9.2 Face detection, a cascade, not a quantized net

**[Repo-stated]** `FaceDetGenerator.c` / `FaceDetModel.c` describe a cascade. There is no TFLite file and no quantization step. The Autotiler generates `FaceDetKernels.c`. Same missing `LibTile.a`.

### 9.3 PULP-Frontnet, already quantized, already generated

**[Repo-stated]** `network.h` lines 30–40: uint8 input of 15360 elements, 4 int32 outputs. 15360 = 160×96. Layer 0's banner says `x_h 96`, `x_w 160` (`layer0_BNReluConvolution.c` lines 50–51). The directory name is `frontnet-160x32-bgaug`.

**[Derived]** The camera crop is 160×160 (section 6.1). `network_run_async` reads `activations_size[0]`, which is 15360, from the start of the camera buffer (`network.h` lines 64–75, `main.c` line 146). The network sees the top 96 rows of the 160-row crop. A centered window would be a pointer plus 32 rows. That add is not in `main.c`. The companion review of upstream called this an accidental region of interest. The vendored `main.c` is the same call.

**[Repo-stated]** Nine layers. Weight sizes `{1312, 0, 9728, 9728, 19456, 37888, 75776, 149504, 7696}` (`network.h` lines 62–63). **[Derived]** The sum is 311088, which is `FLASH_WEIGHTS_SIZE` (`network.c` line 52). The files in `hex/` match those sizes byte for byte, and `inputs.hex` is 15360 bytes. `wc -c` on the directory is 326448 including the input vector. They are raw bytes with a `.hex` suffix, not Intel HEX text.

**[Repo-stated]** `print_perf` is called with **14138880** MACs (`network.c` line 370), and only if `NETWORK_VERBOSE` is on. Frontnet sets `NETWORK_VERBOSE` to 0 (`config.h` line 50). Dequantization is four hard-coded scales (`network.c` lines 418–425) into `x, y, z, phi` (`main.c` lines 152–158). The stamp on the UART message is `latest_state.timestamp` (`main.c` line 112), the timestamp inside the newest UART state message, not a timestamp taken at the exposure.

**[Repo-stated]** There is no PyTorch file, no QuantLib recipe, and no training set for this network. `docs/examples/nanocockpit/pulp-frontnet.md` lines 40–43 says it was trained on a person walking in a room and works less well on a person sitting at a desk. **[Opinion]** You can flash this network. You cannot retrain it from this repo. If the Christmas-package classifier is the model you want to own, it is the one with a training script, and it is also the one the Autotiler shutdown blocks.

---

## 10. Latencies and frame rates that this repo does, and does not, state

| Claim | Where it is stated | What it actually is |
| --- | --- | --- |
| Wi-Fi streamer "60 FPS sensor timing" | `Makefile` lines 37–44, doc line 77 | Two Himax registers, 266 lines and 264 integration lines. No measured fps. **[Repo-stated]** |
| Release streamer is pipelined QVGA raw, own access point | `tools/build/build` line 7, `release.yml` | A build configuration. Host fps is whatever the link delivers. **[Repo-stated]** |
| Profiling `q_ms`, `tx_ms`, `frame_ms` | `wifi-img-streamer.c` lines 211–217 | FreeRTOS tick deltas, printed only if `OUTPUT_PROFILING_DATA=1`. Off in the release flags. **[Repo-stated]** |
| Demosaic 2.33 s → 33.3 ms at 50 MHz | `simple_kernel_example/README.md` lines 13–18 | A teaching measurement, SDK 3.8.1, not the camera pipeline. **[Repo-stated]** |
| Face detection cluster at 75 MHz because 175 MHz drops frames | `FaceDetection/main.c` line 339 | A comment. No count of failures. **[Repo-stated]** |
| NanoCockpit streamer 60 Hz, Frontnet 30 Hz, cluster 175 MHz, FC 246 MHz | the two `config.h` files | Sensor and clock *targets*. **[Repo-stated]** |
| Viewer fps from GAP8 timestamps | `streamer.py` lines 401–414 | Computed on the host when the stream is running. Not run here. **[Repo-stated]** |
| 14.138880 million MACs | `network.c` line 370 | The generator's constant. Printed only if verbose, which is off. **[Repo-stated]** |
| Paper: 48 Hz closed loop, 3.8 MAC/cycle, 72 Hz of 160×160, 55 ms mean RTT, 150 frame/s | the PDF, via the companion review | Not a constant, a log, or a test in this repository. **[Derived]** |

**[Derived]** SPI payload ceilings, from the baud rates and the buffer sizes, with no handshake and no Wi-Fi: original CPX at 10 MHz, one QVGA frame, 63 ms; NanoCockpit at 30 MHz, one 160×160 frame, 6.8 ms. The real rates are lower. The original path pays a GPIO round trip per 1020-byte packet (about 78 trips per QVGA frame). NanoCockpit pays it per 4092-byte DMA and does not memcpy the pixels into the SPI buffer.

---

## 11. Side by side with NanoCockpit

This is the comparison that matters for the evaluation. The left column is the original examples, the right column is the vendored NanoCockpit that this same repo builds with SDK 3.8.1.

| Question | Original examples | NanoCockpit in this tree |
| --- | --- | --- |
| OS | FreeRTOS tasks, each with a stack. `PMSIS_OS = freertos`. CI path `GCC_RISCV_FREERTOS`. | pulp-os. Coroutines in `coroutine.h`: a function returns at `CO_YIELD` / `CO_WAIT`, resume is a `switch` on `__LINE__`. CI path `GCC_RISCV`. |
| Who can stall the system | A task that never blocks stalls only its priority. These demos are all priority `IDLE+1`, so they round-robin. | One fabric-controller event loop. A coroutine that never hits `CO_YIELD` or `CO_WAIT` stalls the camera, the UART, and SPI. `coroutine.h` lines 245–247 says to yield at least every few hundred microseconds. Nothing enforces it. |
| Cluster | Fork/join from a FreeRTOS task, and the camera callback waits for it (classification, face detection). | Fork/join from a coroutine via `pi_cluster_send_task_to_cl_async`. The coroutine waits on a `pi_task`. Cluster cores do not run coroutines. |
| Camera buffers | 1 (start-stop) or 3 (pipelined). Pipelined mode drops any frame that completes while a send is in progress. | 2. Buffer N is in the callback while buffer N+1 is captured. The callback's result is kept. |
| Sensor between frames | Start-stop: off. Pipelined: left in streaming mode, CPI stays armed. | `himax_stop` after every DMA. Frames that start while the CPI is stopped are lost. |
| Overlap of compute and transfer | Encode and SPI run while the sensor keeps exposing, and those exposed frames are discarded. No cluster in the streamer. | After layer 0 the camera buffer is released (`network.c` 292–306) while layers 1–8 run. SPI send of the frame is started in parallel with inference (`main.c` 110–130) and then waited on. |
| Copy of the image onto SPI | `memcpy` into a 1020-byte packet, then another copy into the SPI buffer, at 10 MHz. | The image pointer is the SPI tail. 30 MHz one way. 7.2 MHz if the GAP8 must receive. |
| Packet to the flight controller | CPX to `CPX_T_STM32` through the ESP32 (`stm_gap8_cpx.c`). Console logs go to the cfclient the same way. | UART1 at 115200, a private framing, not CPX. No cfclient console. The follow-me controller that would close the loop is not in this tree. |
| Wi-Fi setup | Either the Crazyflie firmware's menuconfig, or `SETUP_WIFI_AP` (SSID `WiFi streaming example`, TCP port 5000, viewer default `192.168.4.1`). | The ESP32 image brings up `aideck-nanocockpit` and `aideck.local`. The GAP8 does not send SSID packets. |
| Quantized network you can rebuild | MobileNetV2, TFLite int8, then NNTool, then Autotiler. Blocked without `LibTile.a`. | PULP-Frontnet, DORY output already in the tree, weights in `hex/`. No training recipe. |
| SDK | `bitcraze/aideck:latest`, GAP SDK 4.22.0-15, image built May 2024. | `bitcraze/aideck:3.8.1`, GAP SDK 3.8.1, image built July 2021. |

**[Opinion]** The NanoCockpit design wins on the thing John is evaluating: a network that stays on the cluster while the next frame is already in the other buffer, and a SPI send that does not copy the pixels. The original Wi-Fi streamer, after the 2026 pipeline change, solves a narrower problem well: keep the sensor streaming so the frame you send is fresh, and drop rather than queue. It is the right tool for collecting a dataset. It is not a runtime for a closed-loop network.

**[Derived]** The coroutine memory claim in the paper (the companion review: 18 bytes and 8× less than a FreeRTOS stack) is about the resume record in the paper's figure. `co_fn_ctx_t` in this tree embeds a `pi_task_t` (`coroutine.h`). The compiled object is larger than 18 bytes. What the code does, and what is worth copying, is that a suspend is a return and there is no per-task stack. The FreeRTOS demos pay `configMINIMAL_STACK_SIZE * 2` or `* 4` per task (`wifi-img-streamer.c` lines 257–277), on top of the 80 KB SPI queue.

**[Derived]** A concrete bug that the two-buffer design allows, and that a single FreeRTOS camera task does not: `camera_callback` in `pulp-frontnet/main.c` lines 83–132 keeps `inference_done`, `streamer_tx_done`, `frame_done`, and `inference_args` in function-scope statics, and there is one global `inference_ctx` (line 78). `camera_task` can have two callbacks in flight, because it only waits for a buffer's callback when that buffer is needed again (`camera.c` lines 141–143). `CO_WAIT` wakes every waiter. Two callbacks then share one `streamer_t` and one `cpx` send slot. `cpx_spi_send_async` aborts if a send is already posted. A build with `CAMERA_BUFFERS` 1 does not have two contexts. I did not run the binary. This is the control flow, and it is the same control flow the companion review found upstream.

---

## 12. Building and flashing on a Crazyflie 2.1 Brushless with an AI-deck 1.1

Nothing in this section was flashed. The commands are the ones the repository documents, with the corrections from the build in section 14.

### 12.1 What the airframe changes, and what it does not

**[Repo-stated]** This repository has no STM32 firmware, no motor map, and no `cf21bl` platform. The GAP8 and ESP32 images do not read the Crazyflie OTP. They cannot tell a brushed 2.1 from a brushless 2.1.

**[Derived]** The AI-deck 1.1 is the board this code was written for: Himax, GAP8, NINA, the SPI pads in both CPX implementations, deck product id that the Crazyflie firmware calls `bcAI`. The deck connector on the Crazyflie 2.1 Brushless is the same deck connector. The GAP8 binaries in this repo do not need a brushless port.

**[Opinion]** The flight controller under the deck does.

- Over-the-air flashing of `examples/` (`cfloader ... deck-bcAI:gap8-fw`) needs a Crazyflie firmware that still has the AI-deck driver and the original ESP32 image. `docs/development/switching-firmware.md` lines 55–69 says current Crazyflie release packages replace the ESP32 firmware with the NanoCockpit one and leave the GAP8 bootloader alone. After that upgrade, flash `aideck_esp.bin` back before using `examples/`, or skip the radio and use JTAG.
- Do not put the 2021.06 Crazyflie image from upstream NanoCockpit (the tree the companion review covers) on a brushless airframe. That review found only brushed `CF20` / `CF21` motor maps and no `cf21bl` platform. Bitcraze added the brushless platform in the 2024.10 firmware, which is not in either of John's GAP8 trees. A brushless airframe should keep a current Crazyflie firmware.
- Closed-loop "follow the person" is not in this repository. The UART pose from PULP-Frontnet has no consumer here (`pulp-frontnet.md` lines 45–49). Flying that loop means porting upstream's STM32 app onto a `cf21bl` firmware. That port is the companion review's section 10, and it is a port, not a rebuild of this repo.
- Crazyradio: the docs call `cfloader` and `radio://0/80/2M/E7E7E7E7E7` (`docs/development/flashing.md` lines 93–100). This repo does not pin cflib. Whether Crazyradio 2.0 enumerates is a property of the cfloader you install, not of these sources. JTAG flashing of NanoCockpit does not use the radio at all.

**[Opinion]** A useful order on the brushless airframe: leave the STM32 and the NRF on the firmware the aircraft shipped with (or a current Bitcraze release that knows `cf21bl`), confirm the AI-deck enumerates, then choose one of the two GAP8/ESP32 pairs below. Do not downgrade the flight controller to match a 2021 GAP8 tree.

### 12.2 Path A, original examples, no JTAG required after the bootloader is present

Use this to get pictures on a laptop and to see the cfclient console. It does not run PULP-Frontnet.

1. Confirm the ESP32 is the original firmware. If the deck advertises `aideck-nanocockpit`, or a Crazyflie upgrade was just applied, flash `aideck_esp.bin` from the aideck-esp-firmware releases first (`docs/development/switching-firmware.md` lines 24–27). The green LED blinks and the console shows `ESP32: I (483) SYS: Initialized` when that firmware is up (lines 36–42).
2. Either flash the released `aideck_gap8_wifi_img_streamer_with_ap.bin` with `cfloader flash <file> deck-bcAI:gap8-fw -w <uri>`, or build it. The build that matches the release is:

```bash
docker run --rm -v "${PWD}:/module" bitcraze/aideck \
  tools/build/make-example examples/other/wifi-img-streamer clean build image \
  CAMERA_RESOLUTION=qvga CAPTURE_MODE=pipelined STREAM_ENCODING=raw \
  SENSOR_FRAME_RATE=default SETUP_WIFI_AP=1
```

The binary is `examples/other/wifi-img-streamer/BUILD/GAP8_V2/GCC_RISCV_FREERTOS/target.board.devices.flash.img`. Section 14 built this, without `SETUP_WIFI_AP`, and the flash image was 59424 bytes.

3. Join the access point `WiFi streaming example`. From `examples/other/wifi-img-streamer`, `python opencv-viewer.py`. Default address `192.168.4.1` port 5000. Install `opencv-python` on the host. The SSID setup packet has the bug in section 13; if the access point does not appear, set the Wi-Fi from the Crazyflie menuconfig instead (`docs/examples/wifi-streamer.md` lines 22–37) and build with `SETUP_WIFI_AP` left at 0.
4. Hello world is the same docker image and the same `cfloader` line, with `examples/other/hello_world_gap8`. The console line is the proof the radio path works before you care about the camera.

**[Repo-stated]** `cfloader` needs the GAP8 bootloader. A previous NanoCockpit JTAG flash removed it. Restoring it is a JTAG step (`docs/development/switching-firmware.md` lines 20–23).

### 12.3 Path B, NanoCockpit, JTAG, the network

Use this to run the framework John is evaluating. It needs the Olimex ARM-USB-TINY-H (or another FTDI JTAG the SDK's OpenOCD accepts), the 20-to-10 adapter, and Linux if the container must see the USB device (`docs/development/flashing.md` line 20).

1. Flash the ESP32 with aideck-esp-firmware-v2, using that repository's instructions. When it is up, the deck creates `aideck-nanocockpit` and the green LED blinks (`switching-firmware.md` lines 47–50). A current Crazyflie release may have done this step already (lines 64–65).
2. On the host, from `nanocockpit/client/aideck_cpx_streamer`, run `uv sync` while the laptop still has internet. The package wants Python 3.14 (`pyproject.toml` line 16). Then join `aideck-nanocockpit`.
3. Build and flash. The container command is `docs/development/flashing.md` lines 36–50. Inside it:

```bash
source /gap_sdk/configs/ai_deck.sh
cd nanocockpit/examples/streamer    # or pulp-frontnet
make clean build image
# flash only when the JTAG cable is on the GAP8 header, left side of the deck:
make flash
```

Do not pass `-j` to a single `make build image`. Section 14 did, and `gapy` started reading the ELF before the linker had written it (`ELFError: Magic number does not match`). `make -j$(nproc) build && make image` linked. CI's `make build image` is serial and is the command that matches the workflow file.

4. `make run` loads into L2 and does not touch flash (`flashing.md` lines 53–64). Use it for `coroutine` and `queue` so you do not wear out the "I overwrote the bootloader" cycle while reading logs. Ctrl-C prints errors the doc says to ignore.
5. `uv run plt_viewer` shows the frames. PULP-Frontnet adds a second panel with `x, y, z, phi` (`pulp-frontnet.md` lines 34–38). The pose also goes out UART1. The stock Crazyflie firmware does not fly from it.

**[Derived]** The streamer flash image from section 14 is 57376 bytes, L2 static 52448 bytes (10% of 512 KB). The Frontnet flash image is 589016 bytes because the 311088 bytes of weights and the 15360-byte test vector are in the read-only filesystem. L2 static is 84128 bytes (16%). The big L2 allocations happen at runtime (section 7).

**[Opinion]** Build the streamer first and look at 60 Hz HALF frames in the viewer. Then build Frontnet, same container, same JTAG, same ESP32 image. If the viewer shows frames and the pose panel stays empty, the failure is the UART consumer on the STM32, which this repo does not include, not the GAP8 flash.

### 12.4 What not to do today

- Do not expect `docker run ... aideck-with-autotiler` to pull. That image is not on Docker Hub, and `LibTile.a` is not inside `bitcraze/aideck:latest` or `:3.8.1`.
- Do not point the NanoCockpit Makefiles at the `latest` image. They are built and tested on 3.8.1, and the Himax start/stop path is written against that SDK's private struct.
- Do not flash the GAP8 and the ESP32 from different combinations. Section 3.
- Do not flash a brushless Crazyflie with the 2021.06 firmware from the other NanoCockpit tree.

---

## 13. Gaps and likely bugs

Highest-confidence items. Each is labeled.

1. **Wi-Fi setup writes the source route into the receive packet.** `wifi-img-streamer.c` lines 79–80 sets `wifi_tx_packet.route.destination` and then `wifi_rx_packet.route.source`. The transmit source stays 0. Targets are numbered from 1 (`cpx.h` lines 33–39). The same two lines are copied into `FaceDetection/main.c` lines 158–159, inside `#ifdef SETUP_WIFI_AP`, which that Makefile does not define. The release script does define it for the streamer (`tools/build/build` line 7). **[Repo-stated]** **[Opinion]** If the ESP32 router checks the source, the access point never comes up. If it ignores the source, you only lose the identity of the sender.

2. **SSID length is short by the command byte.** `wifi-img-streamer.c` lines 85–87 copies `sizeof(ssid)` bytes (23, the 22-character string plus NUL) into `wifi_ctrl->data` and sets `dataLength` to that 23. The byte before `data` is the command. `WIFI_CTRL_WIFI_CONNECT` on the next lines sets `dataLength` to 2, which is command plus one data byte. The SSID packet does not follow that pattern. **[Derived]** A receiver that trusts `dataLength` reads the command and 22 payload bytes, and stops before the NUL. The face-detection copy has the same lengths.

3. **`cpxSendPacket` does not send.** `lib/cpx/src/cpx.c` lines 97–99 returns `true`. Callers that use the timeout API believe the packet was queued. Nothing in the demos calls it; they call `cpxSendPacketBlocking`. **[Repo-stated]**

4. **`memset` of the CPX queue table uses the wrong size.** `cpx.c` line 134. **[Derived]** Only `queues[0]` is filled with `0x0F`, and function ids start at 1, so the demos survive. A function id of 0 would `xQueueSend` to `0x0F0F0F0F`.

5. **Face detection streams past the end of `ImageOut`.** Allocation is 64×48 = 3072 bytes (`main.c` lines 48–49 and 277). The send length is `STREAM_W * STREAM_H` = 160×120 = 19200 (lines 385–388). The header width and height are 324×244 (lines 129–130). **[Derived]** The GAP8 reads 16128 bytes past `ImageOut` into whatever `pmsis_l2_malloc` placed next (the integral images are the previous allocations). The viewer then reshapes 19200 bytes into 244×324 and throws. The doc's advice to edit the viewer does not match the viewer, which takes its size from the header.

6. **`sendBufferViaCPX` copies `sizeof(packet->data)` even on the last chunk.** `FaceDetection/main.c` line 146 uses the full buffer size as the `memcpy` length, while `size` may be smaller. **[Derived]** The last chunk reads past the source. The Wi-Fi streamer's `cpx_stream_send_buffer` uses `chunk_size` (`cpx_stream.c` line 71) and does not have this bug.

7. **Two Frontnet callbacks, one set of statics.** Section 11. **[Derived]** Can trip the single-flight SPI send. `CAMERA_BUFFERS` is 2 in the checked-in config.

8. **`compute_frame_len_lines` is called with the arguments swapped.** `himax.c` lines 183 and 233. **[Derived]** Integer frame rates still compute. The line count is not the line count.

9. **CPX send FIXME.** `nanocockpit/lib/cpx/cpx.c` lines 127–133. **[Repo-stated]**

10. **Default NanoCockpit SPI drops every received byte.** `cpx_spi.c` lines 311–316. The paper's bidirectional stack is an opt-in that the demos leave off, and turning it on drops the clock from 30 MHz to 7.2 MHz. **[Repo-stated]**

11. **The network reads 96 rows from the top of a 160-row buffer.** Section 9.3. **[Derived]**

12. **`uart_printf` writes through `_uart`, which `uart_init` never assigns.** `nanocockpit/lib/uart.c` lines 35 and 78–91. `DEBUG_PRINT` in `debug.h` line 37 calls `uart_printf`. It is behind `VERBOSE` paths that the demos do not all enable. **[Opinion]** Dead until something calls it, and then it is a null dereference.

13. **`NETWORK_TEST_INPUT` is a comment.** `pulp-frontnet/config.h` line 53. No C file reads it. The GVSOC camera and UART lines in `pulp-frontnet/Makefile` lines 37–39 are commented out. **[Repo-stated]** **[Opinion]** The simulator path is not wired in this copy.

14. **Classification never sets the cluster frequency.** `classification.c` line 149 is commented out. `FREQ_CL=170` is only an Autotiler budget input. **[Repo-stated]**

15. **Original SPI is 10 MHz and copies every packet twice**, into a queue sized for one QVGA frame (`com.c` lines 135, 70–71; `cpx.c` lines 86–95; `cpx_stream.c` lines 61–75). **[Repo-stated]** **[Derived]** This is the throughput gap versus NanoCockpit, independent of any paper number.

16. **Not bugs, and not in the tree:** a brushless motor driver, a follow-me controller, QuantLib, a DORY invocation, `LibTile.a`, flight logs, and any measured closed-loop latency. The Frontnet weights are in the tree. The classifier weights are a zip you have to unpack, and then you still cannot generate kernels.

---

## 14. What was built in this environment

The VM started without Docker, without a RISC-V toolchain, and without a GAP SDK. `make -C examples/other/wifi-img-streamer` failed with `Makefile:74: /pmsis_rules.mk: No such file or directory`.

Docker 29.1.3 was installed and `dockerd` was started. Two images were pulled from Docker Hub:

- `bitcraze/aideck:3.8.1` (digest `sha256:0988d477e97f...`), GAP SDK tag `release-v3.8.1`, GCC 7.1.1, Python 3.6.9.
- `bitcraze/aideck:latest` (digest `sha256:038197df9cb8...`), GAP SDK `release-v4.22.0-15-ga2302650`, GCC 7.1.1, Python 3.8.10.

`LibTile.a` is absent from both. `examples/image_processing/FaceDetection` `make GenTile` in the `latest` image failed with `LibTile.a: No such file or directory`. Classification was not submitted to a full `make`; it links the same library (`model_rules.mk` lines 64–67) and stops at the same file.

**Wi-Fi streamer, built.** `bitcraze/aideck:latest`, `tools/build/make-example examples/other/wifi-img-streamer clean build image` with `CAMERA_RESOLUTION=qvga CAPTURE_MODE=pipelined SENSOR_FRAME_RATE=default STREAM_ENCODING=raw`. Exit 0. `pip3 install numpy==1.22.3` succeeded and printed a pillow/imageio warning. One assembler warning, `ignoring changed section attributes for .data`, came from the SDK, not from this repo. Output:

- ELF `BUILD/GAP8_V2/GCC_RISCV_FREERTOS/wifi-img-streamer`, 898308 bytes on disk (includes debug).
- `riscv32-unknown-elf-size -A`: `.text` 49384, `.rodata` 1800, `.data` 2016, `.bss` 10812.
- Linker map `__heapl2ram_size` = `0x70548` (460104 bytes).
- Flash image `target.board.devices.flash.img`, 59424 bytes.

`SETUP_WIFI_AP` was not set on this particular command, so this binary is the pipelined streamer without the access-point packets. The release script adds that flag. The image was not flashed.

**NanoCockpit streamer, built.** `bitcraze/aideck:3.8.1`, `source ai_deck.sh`, `make -C nanocockpit/examples/streamer clean`, then `make build image` with no `-j`. Exit 0. Linker print:

```text
L2:     52448 B  /  512 KB   (10.00%)
fc_tcdm: 5708 B  /  16380 B  (34.85%)
```

`riscv32-unknown-elf-size`: text `0xb454`, data `0x1c0c`, bss `0x12c4`, dec 58148. Flash image 57376 bytes.

The same target with `make -j$(nproc) build image` failed: `gapy` reported `ELFError: Magic number does not match` while the linker was still writing the ELF. Serial `make build && make image` is the command that matches CI. **[Opinion]** The rules file starts the image recipe too early under `-j`. Do not "fix" it by editing the SDK inside the container.

**PULP-Frontnet, built.** Same image, `make -j$(nproc) build && make image`. Exit 0. The generated layers compiled with the warnings that `app/app.mk` demotes (`-Wno-error=discarded-qualifiers`, `int-conversion`, and the others). Linker print:

```text
L2:     84128 B  /  512 KB   (16.05%)
fc_tcdm: 6284 B  /  16380 B  (38.36%)
```

`size`: text `0x12cb8`, data `0x1f08`, bss `0x1568`, dec 90408. Flash image `target.board.devices.flash.img`, 589016 bytes, which includes the weight files and `inputs.hex`.

**Not built:** hello world, com-test, stm_gap8, test_camera, uart, the simple kernel, image manipulations, classification (blocked on `LibTile.a`), the small NanoCockpit examples (`coroutine`, `queue`, `queue_async`, `cpx`). They use the same two images and the same rules. The host viewer was not run (`uv`, Python 3.14, a Wi-Fi network). No OpenOCD, no flash, no GVSOC.

Build products were deleted after the sizes were recorded and are not part of the change set.

---

## 15. Short verdict

**[Repo-stated]** John's fork is Bitcraze `aideck-gap8-examples` at `322361f`, with no local commits. It contains two GAP8 stacks that must be paired with two different ESP32 firmwares. The original stack is FreeRTOS, 10 MHz SPI, a memcpy per packet, and a 2026 three-buffer camera path that drops frames on purpose so the one it sends is fresh. The NanoCockpit stack is pulp-os coroutines, a two-buffer camera, 30 MHz one-way SPI with the image as the DMA tail, and a DORY PULP-Frontnet that releases the frame after the first layer. Both stacks compiled in the published Docker images. The Autotiler networks did not, because `LibTile.a` is no longer downloadable and is not in the images.

**[Derived]** The closed-loop flight in the paper is not in this repository. The brushless airframe does not require a different GAP8 binary. It does require that the STM32 firmware stay a current brushless firmware, and that the ESP32 firmware match the GAP8 stack you flash.

**[Opinion]** On this hardware, this week: flash the NanoCockpit streamer over JTAG, look at 160×160 frames, then flash PULP-Frontnet the same way. Use the original Wi-Fi streamer when you want a dataset and you still have the GAP8 bootloader. Leave classification and face detection until a copy of `LibTile.a` turns up. Do not expect either image in this repo to fly the brushless airframe by itself.
