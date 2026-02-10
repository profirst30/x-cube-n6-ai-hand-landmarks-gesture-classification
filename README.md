# Hand Gesture Recognition with STM32N6

**Real-time finger counting (0-5) using AI-powered hand landmark detection on STM32N6570-DK**

> **Based on**: [STMicroelectronics x-cube-n6-ai-hand-landmarks](https://github.com/STMicroelectronics/x-cube-n6-ai-hand-landmarks)   
> **Key Enhancement**: Added intelligent finger counting algorithm with optimized display

---

## What I Built

This project adapts STMicroelectronics' hand landmark detection demo into a **practical gesture recognition system**. While the original project focused on detecting 21 hand landmarks, I extended it to:

- **Analyze landmark geometry** to count raised fingers (0-5)
- **Implement custom display rendering** with large, clear digits (72px black font)
- **Document the algorithm** in detail for developers to understand and extend

The system runs two AI models sequentially on the STM32N6570-DK board:

1. **Palm Detection** — Detects hand position in camera frame
2. **Hand Landmark Detection** — Identifies 21 key points (MediaPipe convention)
3. **Finger Counting** *(my addition)* — Analyzes landmarks to recognize raised fingers

---

## Why This Matters

Real-time gesture recognition on embedded devices opens doors for:
- **Touchless interfaces** in medical/industrial environments
- **Human-robot interaction** without external sensors
- **Accessibility solutions** for hands-free device control
- **Educational demos** of edge AI capabilities

This implementation demonstrates that complex vision tasks can run efficiently on microcontrollers with NPU acceleration.

---

This README provides a quickstart guide. For deeper technical details, see the [Doc](./Doc/) folder.

## Demo

![Real-time finger counting demo](_htmresc/demo_classification_handlandmarks.gif)

*The system detects hand landmarks and counts raised fingers in real-time, displaying the count (0-5) on the STM32N6570-DK LCD*

---

## My Enhancements

### 🎯 Finger Counting Algorithm
Developed a robust gesture recognition algorithm that:
- Analyzes **21 MediaPipe hand landmarks** to determine finger states
- Uses **geometric heuristics** (palm center, tip-to-PIP comparisons)
- Handles **thumb detection** separately via horizontal distance
- Achieves **real-time performance** (every frame during tracking)

📖 **[Read the full algorithm documentation](Doc/Finger-Counting-Logic.md)**

### 🖥️ Enhanced Display
- **Custom scaled font rendering** (72px digits) for clear visibility
- **Black text** on light background for maximum contrast
- **Top-left placement** to avoid obscuring hand tracking visuals

### 📚 Comprehensive Documentation
- **[Finger Counting Logic](Doc/Finger-Counting-Logic.md)** — Algorithm deep-dive with code explanations
- Preserved original STMicroelectronics documentation for hardware/build instructions

---

## Original Features (from STMicroelectronics)

The base project demonstrates advanced STM32N6 capabilities:

- Multi-threaded application flow (FreeRTOS)
- NPU-accelerated quantized AI model inference
- GPU2D for resize and rotation operations
- Sequential multi-model execution pipeline
- 21-point hand landmark detection (MediaPipe convention)
- Dual DCMIPP pipelines with ISP support
- DCMIPP cropping, decimation, and downscaling
- LTDC dual-layer display implementation
- Boot from external flash (no internal flash on STM32N6)

---

## Documentation

- **[Finger Counting Logic](Doc/Finger-Counting-Logic.md)** *(my addition)* — Detailed algorithm explanation
- [Application Overview](Doc/Application-Overview.md) *(original)* — System architecture
- [Boot Overview](Doc/Boot-Overview.md) *(original)* — Boot modes and flash programming
- [Camera Build Options](Doc/Build-Options.md) *(original)* — Camera module configurations

---

## Getting Started

### Prerequisites

**Hardware:**
- [STM32N6570-DK](https://www.st.com/en/evaluation-tools/stm32n6570-dk.html) Discovery Board
- USB-C to USB-C cable (for sufficient power via CN6 ST-LINK port)
- Supported camera module (IMX335, STEVAL-55G1MBI, STEVAL-66GYMAI, or STEVAL-1943-MC1)

**Software:**
- [STM32CubeProgrammer](https://www.st.com/en/development-tools/stm32cubeprog.html) v2.18.0+
- [STEdgeAI](https://www.st.com/en/development-tools/stedgeai-core.html) v3.0.0
- IDE: [STM32CubeIDE](https://www.st.com/en/products/development-tools/software-development-tools/stm32-software-development-tools/stm32-ides/stm32cubeide.html) v1.17.0+ or IAR EWARM 9.40.1 (with N6 patch)

![STM32N6570-DK Board](_htmresc/STM32N6570-DK.png)

*STM32N6570-DK with MB1854B IMX335 camera module*

---

### Quick Start (Prebuilt Binaries)

The fastest way to test the finger counting demo:

1. **Set board to Development Mode** ([see boot modes](#boot-modes))
2. **Program the prebuilt firmware**: `Binary/x-cube-n6-ai-hand-landmarks-dk.hex`
3. **Set board to Boot from Flash** mode
4. **Power cycle** the board
5. **Position your hand** in front of the camera and see the finger count displayed!

#### Programming via STM32CubeProgrammer CLI

```bash
export DKEL="<STM32CubeProgrammer_N6_Install_Folder>/bin/ExternalLoader/MX66UW1G45G_STM32N6570-DK.stldr"
STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el $DKEL -hardRst -w Binary/x-cube-n6-ai-hand-landmarks-dk.hex
```

📖 **[GUI programming instructions](Doc/Program-Hex-Files-STM32CubeProgrammer.md)**

---

### Building from Source

Want to modify the finger counting algorithm or experiment with the code?

#### Prerequisites
Before building, program the AI model weights (only needed once):
```bash
export DKEL="<STM32CubeProgrammer_N6_Install_Folder>/bin/ExternalLoader/MX66UW1G45G_STM32N6570-DK.stldr"
STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el $DKEL -hardRst -w Model/palm_detector_data.hex
STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el $DKEL -hardRst -w Model/hand_landmark_data.hex
```

#### Build Options

**Using Makefile** (my preferred method):
```bash
make -j8  # Parallel build with 8 threads
```

**Using STM32CubeIDE:**
1. Open `STM32CubeIDE/STM32N6570-DK/.project`
2. Build and run

**Using IAR EWARM:**
1. Open `EWARM/STM32N6570-DK/x-cube-n6-ai-hand-landmarks-dk.eww`
2. Build and run

#### Development Mode (RAM Loading)

Set board to **Development Mode**, build, then use GDB for debugging:

```bash
# Terminal 1: Start GDB server
ST-LINK_gdbserver -p 61234 -l 1 -d -s -cp <stm32cubeprogrammer-bin-dir> -m 1 -g

# Terminal 2: Load firmware
arm-none-eabi-gdb build/Project.elf
(gdb) target remote :61234
(gdb) load
(gdb) continue
```

#### Boot from Flash (Persistent Firmware)

1. **Build** the application (see build options above)
2. **Sign** the binary:
   ```bash
   STM32_SigningTool_CLI -bin build/Project.bin -nk -t ssbl -hv 2.3 -o build/Project_sign.bin
   ```
3. **Program** FSBL + signed application:
   ```bash
   STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el $DKEL -hardRst -w FSBL/ai_fsbl.hex
   STM32_Programmer_CLI -c port=SWD mode=HOTPLUG -el $DKEL -hardRst -w build/Project_sign.bin 0x70100000
   ```
4. **Set board to Boot from Flash** mode and power cycle

---

## Boot Modes Reference

The STM32N6 has **no internal flash**. Firmware must be stored in external flash for persistence.

| Mode | Purpose | Jumper Position |
|------|---------|----------------|
| **Development Mode** | Load to RAM for debug / Program external flash | ![Dev mode](_htmresc/STM32N6570-DK_Dev_mode.png) |
| **Boot from Flash** | Run from external flash after programming | ![Boot from flash](_htmresc/STM32N6570-DK_Boot_from_flash.png) |

---

## Serial Console

Monitor application logs via ST-LINK virtual COM port:
- **Baud rate**: 115200 bps
- **Parity**: None
- **Stop bits**: 1

---

## Technical Details

### Finger Counting Algorithm
The counting logic uses geometric analysis of MediaPipe hand landmarks:
- **Thumb**: Horizontal distance from palm center
- **Index/Middle/Ring/Pinky**: Vertical TIP-to-PIP comparison

Result: 0-5 raised fingers detected per frame.

📖 **[Deep dive into the algorithm](Doc/Finger-Counting-Logic.md)**

### Display Implementation
Custom `display_scaled_char()` function renders Font24 glyphs at 3× scale:
- 24px → **72px height** for visibility across the room
- Black text on light background
- Top-left corner placement (non-intrusive)

### AI Models
- **Palm Detector**: centerface.tflite (detects hand region)
- **Hand Landmark**: mobilefacenet_int8.onnx (21 keypoints, MediaPipe format)
- **Runtime**: STEdgeAI 3.0.0 with NPU acceleration

---

## Troubleshooting

**"Possible mismatch in ll_aton library used" error:**
- Ensure STEdgeAI version matches v3.0.0
- Follow [STEdgeAI update instructions](https://stedgeai-dc.st.com/assets/embedded-docs/stneuralart_faqs_update_version.html)

**Finger count incorrect:**
- Check hand orientation (fingers should point upward)
- Ensure good lighting for landmark detection
- See [Known Limitations](Doc/Finger-Counting-Logic.md#known-limitations)

---

## Credits & License

- **Original Project**: [STMicroelectronics x-cube-n6-ai-hand-landmarks](https://github.com/STMicroelectronics/x-cube-n6-ai-hand-landmarks)
- **Adaptations**: Finger counting algorithm, scaled display, enhanced documentation by profirst30
- **License**: See [LICENSE.md](LICENSE.md) for details

---

## Contributing

Found a bug or have an improvement idea? Feel free to:
- Open an issue describing the problem/feature
- Submit a pull request with your changes
- Fork the repo for your own experiments

**Focus areas for contribution:**
- Improving finger counting accuracy
- Adding new gesture types (peace sign, thumbs up, etc.)
- Multi-hand support
- Performance optimizations

---
