# Smart Switch Inspection System

An automated computer vision pipeline for inspecting smart switches on a production line — built as a Bachelor's final project.

The system captures an image of a smart switch, locates and crops it from the scene, applies adaptive image filtering, divides the face into logical segments, and classifies every indicator (LEDs, seven-segment displays, degree lights) as ON/OFF or reads their digit value — all without human involvement.

---

## Pipeline Overview

```
Camera / Image  →  Crop  →  Filter  →  Segment  →  Classify  →  Result dict
```

| Stage | Function | What it does |
|---|---|---|
| **Crop** | `crop_image()` | Template-matches a reference sample at multiple scales to locate the switch, then crops a padded ROI around it |
| **Filter** | `filter()` | Adaptive gamma/gain correction based on scene brightness — suppresses dark pixels, boosts lit indicators |
| **Segment** | `crop_segments()` | Divides the cropped face into named regions using a grid coordinate system |
| **Classify** | `check_lights()` | Dispatches each segment to the appropriate detector |

---

## Segment Map

The switch face is divided into an **8 × 6 grid**. Each named segment maps to a `(row, col, width, height)` cell:

| Segment | Type | Detector |
|---|---|---|
| `left_up`, `right_up` | Big LED | Connected-components area check |
| `left_down`, `middle_down`, `right_down` | Big LED | Connected-components area check |
| `left_7seg`, `right_7seg` | Seven-segment display | Blob centroid → segment classifier |
| `degrees` | Degree indicator | Horizontal-component count |
| `celcius`, `dot`, `auto` | Small LED | Mean brightness threshold |

---

## Detection Logic

### Big LEDs
Connected components are extracted from a binarized crop. A component is considered a lit indicator if its **pixel area > 100** and its **total brightness sum > 3000**. Returns `"ON"` or `"OFF"`.

### Seven-Segment Displays
Valid blobs (area > 100, sum > 3000) are found and their centroids are mapped to one of the 7 classic segments (A–G) based on their relative position inside the crop. The resulting segment vector is matched against a lookup table for digits 0–9.

### Degree Lights
Horizontal components (where `height/width < 0.6`) with area > 50 are counted. The count is the number of active degree bars.

### Small LEDs (`celcius`, `dot`, `auto`)
A simple mean brightness threshold (`> 5`) determines ON/OFF.

---

## Image Filtering

Brightness is sampled from the cropped ROI and used to set an adaptive gamma:

```
GAMMA  = 2 × floor(brightness / 10)   # crushes dark pixels
GAIN   = 2                             # boosts bright pixels
CUTOFF = 0.15                          # hard floor — clips near-black to 0
```

This makes the detector robust to variable lighting conditions on the production line.

---

## Requirements

```
opencv-python
numpy
matplotlib
```

Install with:

```bash
pip install opencv-python numpy matplotlib
```

---

## Usage

### Single image

```python
cropped_img = crop_image("path/to/photo.jpg", "samp3.jpg")
filtered_img = filter(cropped_img)
segments     = crop_segments(filtered_img)
result       = check_lights(segments)
print(result)
```

### Live camera feed

```python
import cv2, time

cap = cv2.VideoCapture(0)
while True:
    time.sleep(5)
    ret, frame = cap.read()
    if not ret:
        break
    cropped_img = crop_image(frame, "samp3.jpg")
    filtered_img = filter(cropped_img)
    segments     = crop_segments(filtered_img)
    result       = check_lights(segments)
    print(result)
    if cv2.waitKey(1) == 27:
        break

cap.release()
cv2.destroyAllWindows()
```

### Example output

```python
{
  'left_up':      'ON',
  'right_up':     'OFF',
  'left_7seg':    7,
  'right_7seg':   3,
  'degrees':      2,
  'celcius':      'ON',
  'dot':          'OFF',
  'auto':         'ON',
  'left_down':    'ON',
  'middle_down':  'OFF',
  'right_down':   'ON'
}
```

---

## Key Parameters

| Parameter | Default | Description |
|---|---|---|
| `resize` | `(960, 1280)` | Input image resize target |
| `scales` | `[0.6 … 1.4]` | Template matching scale range |
| `crop switch coefficient` | `1.5` | Padding multiplier around detected switch |
| `tighten_coef` | `0.1` | Width crop factor for narrow segments (`degrees`, `celcius`, `dot`, `auto`) |
| `big light min_area` | `100` | Minimum connected-component area |
| `big light min_sum` | `3000` | Minimum pixel brightness sum |
| `seven-seg min_area` | `100` | Same, for seven-segment blobs |
| `little light threshold` | `5` | Mean brightness cutoff for small LEDs |
| `degree area threshold` | `50` | Min area for a degree bar |
| `degree h/w threshold` | `< 0.6` | Aspect ratio gate for horizontal bars |

---

## Project Structure

```
.
├── main.ipynb       # Main pipeline notebook
├── samp.jpg           # Reference template for switch localization
└── tst.jpg/            # Test image
```

---

## How It Works — End to End

1. A camera (or saved image) captures the switch face from the production line.
2. `crop_image` uses multi-scale template matching (`cv2.matchTemplate`) against a reference photo to find and tightly crop the switch, regardless of slight position or zoom variation.
3. `filter` normalizes the brightness adaptively so lit segments are clearly bright and unlit segments are near-zero.
4. `crop_segments` slices the face into 11 named regions using a fixed grid layout calibrated to the switch's physical design.
5. `check_lights` dispatches each region to the detector that suits its type — area thresholds for LEDs, centroid-based segment decoding for seven-segment displays, and component geometry for the degree indicator.
6. The returned dictionary gives the complete inspection result and can be forwarded to a logging system, PLC, or pass/fail decision node.

---

## Author

Bachelor's final project — automated visual inspection of smart switches using classical computer vision (OpenCV).
