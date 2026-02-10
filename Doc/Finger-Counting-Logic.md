# Finger Counting Logic Analysis

## Overview

The **x-cube-n6-ai-hand-landmarks** application implements raised finger counting (0 to 5) by leveraging the 21 landmarks (keypoints) detected by the hand landmark model. The logic is entirely contained in the `count_raised_fingers()` function in the `Src/app.c` file.

---

## Processing Pipeline

```
Camera → Palm Detector → Hand Landmark → Post-processing → Finger Counting → Display
```

### Detailed Steps

1. **Palm Detector**: detects the hand's palm and produces a ROI (Region Of Interest).
2. **Hand Landmark**: from the ROI, the model infers 21 landmarks in normalized coordinates (x, y) within the ROI frame.
3. **`ld_post_process()`** (`Src/ld.c`): normalizes the 63 output floats (21 × 3 coordinates) into 21 `ld_point_t` points with x and y between 0 and 1.
4. **`count_raised_fingers()`** (`Src/app.c`): analyzes the decoded landmarks in screen coordinates to determine how many fingers are raised.
5. **Display**: the finger count (0–5) is displayed in the top-left corner of the screen.

---

## Landmark Convention (MediaPipe Hand)

The model uses the MediaPipe convention with 21 landmarks indexed as follows:

```
          ┌─ TIP (4)
          │
          ├─ IP  (3)         ┌─ TIP (8)    ┌─ TIP (12)   ┌─ TIP (16)   ┌─ TIP (20)
          │                  │              │              │              │
          ├─ MCP (2)         ├─ DIP (7)     ├─ DIP (11)   ├─ DIP (15)   ├─ DIP (19)
          │                  │              │              │              │
          ├─ CMC (1)         ├─ PIP (6)     ├─ PIP (10)   ├─ PIP (14)   ├─ PIP (18)
          │                  │              │              │              │
          │                  ├─ MCP (5)     ├─ MCP (9)    ├─ MCP (13)   ├─ MCP (17)
          │                  │              │              │              │
          └──────────────────┴──────────────┴──────────────┴──────────────┘
                                            │
                                        Wrist (0)
```

| Finger  | CMC/MCP | PIP | DIP | TIP |
|---------|---------|-----|-----|-----|
| Thumb   | 1 (CMC), 2 (MCP) | — | 3 (IP) | 4 |
| Index   | 5       | 6   | 7   | 8   |
| Middle  | 9       | 10  | 11  | 12  |
| Ring    | 13      | 14  | 15  | 16  |
| Pinky   | 17      | 18  | 19  | 20  |

---

## Counting Algorithm

### Step 0: Decode landmarks to screen coordinates

Before counting, each raw landmark (normalized 0–1 in the ROI frame) is converted to screen coordinates by `decode_ld_landmark()`:

```c
decoded->x = roi->cx + (lm->x - 0.5) * w * cos(rotation) - (lm->y - 0.5) * h * sin(rotation);
decoded->y = roi->cy + (lm->x - 0.5) * w * sin(rotation) + (lm->y - 0.5) * h * cos(rotation);
```

This transformation applies:
- **Translation**: centering on the ROI center (`roi->cx`, `roi->cy`)
- **Rotation**: compensation for hand rotation angle (`roi->rotation`)
- **Scaling**: adaptation to ROI size (`w`, `h`)

### Step 1: Estimate palm center

```c
palm_cx = (lm[0].x + lm[9].x) / 2.0f;
```

The horizontal palm center is approximated as the **midpoint** between:
- `lm[0]` = **Wrist**
- `lm[9]` = **Middle finger MCP** (base of middle finger)

This point serves as a **reference** only for thumb detection.

### Step 2: Thumb detection (special logic)

```c
tip_dist = |lm[4].x - palm_cx|;   // Distance from thumb TIP to palm center
ip_dist  = |lm[3].x - palm_cx|;   // Distance from thumb IP to palm center

if (tip_dist > ip_dist)  → Thumb raised ✋
```

**Why different logic?**
The thumb doesn't bend vertically like other fingers: it opens **laterally**. We therefore compare the **horizontal distance** (absolute value) of the thumb tip (TIP, index 4) to the palm center with that of the IP joint (index 3).

- **Thumb raised**: TIP is **farther** from palm center than IP → thumb is extended outward.
- **Thumb folded**: TIP is **closer** to center than IP → thumb is folded toward palm.

> ℹ️ This method works regardless of left/right hand orientation thanks to using absolute value.

### Step 3: Detection of the 4 other fingers (index, middle, ring, pinky)

For these 4 fingers, the logic is **identical and simple**:

```c
if (TIP.y < PIP.y)  → Finger raised ✋
```

| Finger  | Condition                     |
|---------|-------------------------------|
| Index   | `lm[8].y < lm[6].y`          |
| Middle  | `lm[12].y < lm[10].y`        |
| Ring    | `lm[16].y < lm[14].y`        |
| Pinky   | `lm[20].y < lm[18].y`        |

**Principle**: In screen coordinates, **Y increases downward**. If the fingertip (TIP) has a **smaller** Y coordinate than the PIP joint, it means the TIP is **higher** than the PIP → the finger is raised.

```
    TIP ● (small y = top)      Finger is RAISED ✅
         |
    DIP  ●
         |
    PIP  ● (large y = bottom)  Comparison reference
         |
    MCP  ●

    vs.

    PIP  ● (small y)           Finger is FOLDED ❌
         |
    MCP  ●
        / 
    DIP ●
       |
    TIP ● (large y = bottom)
```

> ℹ️ We compare TIP with PIP (not MCP) for more **tolerant** detection: a partially bent but still "raised" finger will be counted.

---

## Integration in Main Loop

```c
// Src/app.c — Inference loop (nn_thread)

if (is_tracking)
    disp.info.finger_count = count_raised_fingers(&rois[0], ld_landmarks[0]);
else
    disp.info.finger_count = -1;   // -1 = no hand detected
```

Counting is performed **every frame** where a hand is tracked. The value is stored in `disp.info.finger_count`:
- **-1**: no hand detected
- **0–5**: number of raised fingers

### Display

```c
// Src/app.c — display_info()

if (info->finger_count >= 0) {
    char digit = '0' + info->finger_count;
    display_scaled_char(10, 10, digit, 3, UTIL_LCD_COLOR_BLACK);
}
```

The number is displayed in **large black characters** (Font24 x3 = 72px) in the top-left corner of the LCD screen.

---

## Algorithm Visual Summary

```
┌─────────────────────────────────────────────┐
│          count_raised_fingers()             │
│                                             │
│  1. Decode 21 landmarks → screen coords    │
│                                             │
│  2. palm_cx = (Wrist.x + MCP9.x) / 2      │
│                                             │
│  3. THUMB:                                  │
│     |TIP(4).x - palm_cx| > |IP(3).x - ..| │
│     → YES = thumb raised (+1)              │
│                                             │
│  4. INDEX:   TIP(8).y  < PIP(6).y   → +1  │
│  5. MIDDLE:  TIP(12).y < PIP(10).y  → +1  │
│  6. RING:    TIP(16).y < PIP(14).y  → +1  │
│  7. PINKY:   TIP(20).y < PIP(18).y  → +1  │
│                                             │
│  Returns: count (0 to 5)                   │
└─────────────────────────────────────────────┘
```

---

## Known Limitations

| Limitation | Detail |
|------------|--------|
| **Orientation** | The algorithm works primarily with the hand oriented **fingers up**. If the hand is upside down (fingers down), Y comparisons are inverted and counting will be incorrect. ROI rotation partially compensates for this issue. |
| **Thumb** | Thumb detection by horizontal distance can fail if the thumb is bent inward at an unusual angle. |
| **No handedness** | The algorithm doesn't distinguish left/right hand, which can affect thumb accuracy in certain positions. |
| **Binary threshold** | Each finger is either raised or folded — there's no concept of "partially raised" finger. |
| **Single hand** | The system handles only one hand at a time (`PD_MAX_HAND_NB` used with index 0 only for counting). |
