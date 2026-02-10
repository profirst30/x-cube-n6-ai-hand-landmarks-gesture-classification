# x-cube-n6-ai-hand-landmarks Copilot Instructions

## Project Overview

This is an embedded computer vision application for STM32N6570-DK that demonstrates **sequential AI model execution** for hand landmark detection:
1. **Palm detector** (pd) finds the hand's palm in the frame
2. **Hand landmark detector** (hl) identifies 21 landmarks on the detected hand
3. GPU2D performs resize/rotation between models to prepare input for the landmark detector

**Critical architectural constraint**: STM32N6 has **no internal flash** - firmware must be programmed to external flash or loaded into SRAM (dev mode).

## Build Systems & Toolchains

Three build systems are maintained in parallel:
- **IAR EWARM 9.40.1** + N6 patch: `EWARM/STM32N6570-DK/*.ewp`
- **STM32CubeIDE 1.17.0**: `STM32CubeIDE/STM32N6570-DK/.project`
- **Makefile (GCC)**: Root `Makefile` with modular includes in `mks/*.mk`

### Makefile Architecture Pattern

The root `Makefile` orchestrates builds using **modular include files**:
- `mks/ai.mk` - AI runtime and neural network library integration
- `mks/fw.mk` - STM32Cube HAL drivers and BSP components
- `mks/freertos.mk` - FreeRTOS kernel and CMSIS-RTOS2 wrapper
- `mks/cmw.mk` - Camera middleware (DCMIPP, ISP, sensor drivers)
- `mks/ipl.mk` - Image processing library
- `mks/nema.mk` - NemaGFX GPU library
- `mks/gcc.mk` / `mks/iar.mk` - Toolchain-specific rules

**When adding dependencies**: Update the corresponding `.mk` file, not the root Makefile.

### Key Build Commands

```bash
# Build all sensors variants
make

# Build specific sensor (IMX335, VD66GY, VD55G1, VD1943, OV5640)
make SENSOR=IMX335

# Clean
make clean

# Program to external flash (requires STM32CubeProgrammer in PATH)
export DKEL="<STM32CubeProgrammer_N6>/bin/ExternalLoader/MX66UW1G45G_STM32N6570-DK.stldr"
STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el $DKEL -hardRst -w Binary/x-cube-n6-ai-hand-landmarks-dk.hex
```

## Boot Modes & Development Flow

**Set boot mode via physical switches** (see README diagrams):
- **Development mode**: Load firmware into SRAM for debugging (lost on power cycle)
- **Boot from flash**: Run from external flash via FSBL (First Stage Boot Loader)

### One-Time Setup Required

**Before first run**, program AI model weights to external flash:
```bash
# These .hex files contain pre-compiled model weights at fixed addresses
Model/palm_detector_data.hex    # @ 0x70380000
Model/hand_landmark_data.hex    # @ 0x70580000
```

This only needs to be done once unless you regenerate models.

## AI Model Integration Pattern

Models are generated using **STEdgeAI (v3.0.0)** with the `Model/generate-n6-model.sh` script:

```bash
# Generates C API wrappers and binary weight files
stedgeai generate --target stm32n6 --st-neural-art <config>.json \
  --model <model>.tflite
```

**Generated files per model**:
- `stai_<model>.{c,h}` - High-level STAI API wrapper (commit to repo)
- `<model>.c` - Network structure definition (commit to repo)
- `<model>_ecblobs.h` - External blob descriptors (commit to repo)
- `<model>_data.hex` - Model weights for external flash (program once)

**Integration checklist when adding new models**:
1. Add generated C files to `Model/` directory
2. Update `C_SOURCES` in root Makefile to include new `.c` files
3. Create init/run functions following `palm_detector_init()` / `palm_detector_run()` pattern in `Src/app.c`
4. Update `postprocess_conf.h` if postprocessing parameters change
5. Program `_data.hex` to external flash at designated address

## FreeRTOS Multi-Threading Architecture

Three main threads with **priority-based scheduling**:

```c
#define FREERTOS_PRIORITY(p) ((UBaseType_t)((int)tskIDLE_PRIORITY + configMAX_PRIORITIES / 2 + (p)))

// ISP thread (priority +2) - Camera image enhancement
isp_thread_fct() - Updates ISP configuration on vsync events

// NN thread (priority +1) - AI inference pipeline
nn_thread_fct() - Palm detection → Hand landmark → Post-processing

// Display thread (priority -2) - UI rendering
dp_thread_fct() - Draws detection results and metrics
```

**Synchronization pattern**: Use `disp.lock` (mutex) to protect shared `display_info_t` structure between nn_thread and dp_thread.

## DCMIPP Dual-Pipeline Camera Architecture

The Digital Camera Interface Pixel Pipeline (DCMIPP) runs **two simultaneous pipelines** from the same camera:

- **Pipe 1**: Full-resolution preview → PSRAM display buffer (`lcd_bg_buffer[]`) - continuous capture
- **Pipe 2**: AI input resolution → NN input buffer (`nn_input_buffers[]`) - continuous capture with drop capability

**Frame buffer management**: Uses custom `bqueue_t` (blocking queue) to manage double-buffered `nn_input_buffers` with separate free/ready semaphores. This allows the NN thread to drop frames if inference is slower than capture rate.

## Memory Layout & Critical Sections

```c
// PSRAM (external RAM) - Large buffers
lcd_bg_buffer[4]     // 4608 KB - Display background (800x480x3x4 RGB888)
lcd_fg_buffer[2]     // 1500 KB - Display overlay (800x480x2x2 ARGB4444)
nn_input_buffers[2]  // 221 KB - AI input (192x192x3x2 RGB888)

// NPU RAM (0x34200000) - NPU activations
activations          // 1152 KB

// External Flash (xSPI2) - Read-only model data
palm_detector_data   // 1.14 MB @ 0x70380000
hand_landmark_data   // 3.21 MB @ 0x70580000
```

**Cache coherency**: Use `CACHE_OP()` macro wrapper that checks `is_cache_enable()` before cache operations:
```c
CACHE_OP(SCB_CleanInvalidateDCache_by_Addr(buffer, size));
```

## Configuration & Customization

**`Inc/app_config.h`** - Primary configuration file:
- `CAMERA_FLIP` - Sensor orientation (CMW_MIRRORFLIP_NONE/MIRROR/FLIP/FLIP_MIRROR)
- `HAS_ROTATION_SUPPORT` - Toggle GPU2D rotation (0=MVE resize only, 1=NemaGFX rotation)
- `DISPLAY_DELAY` - Frame buffering depth

**`Inc/postprocess_conf.h`** - Detection thresholds:
- `AI_PD_MODEL_PP_CONF_THRESHOLD` (0.5f) - Palm detection confidence threshold
- `AI_PD_MODEL_PP_IOU_THRESHOLD` (0.4f) - Non-maximum suppression IOU threshold
- `AI_PD_MODEL_PP_MAX_BOXES_LIMIT` (20) - Maximum palm detections

## Key Debugging Features

- **Console output**: UART1 @ 115200 baud via ST-Link (CN6)
- **LCD metrics overlay**: Shows CPU load, inference times, FPS
- **Button controls** (STM32N6570-DK):
  - USER1: Toggle landmark visualization
  - TAMP: Toggle palm detection boxes

## Common Gotchas

1. **Always program model weights first** - The `.hex` files in `Model/` must be flashed to external memory before the app will work
2. **Power via USB-C to USB-C** - ST-Link requires sufficient power; USB-A adapters may not provide enough current
3. **OTP fuses configured** - Board xSPI IOs are pre-configured for 200MHz operation
4. **Static scratch buffers** - `pScratch` in `app.c` is pre-allocated for MVE resize operations
5. **Anchors are hardcoded** - `pd_anchors.c` contains 2016 anchor boxes baked into the palm detector postprocessing
