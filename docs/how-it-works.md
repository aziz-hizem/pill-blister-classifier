# How It Works

A step-by-step walkthrough of the detection pipeline in `pill_classifier.py`. Line numbers refer to that file.

---

## 1. What does this program do?

This program takes a photo of a blister pill pack and automatically:
- Counts how many pills are in the pack
- Detects the color of the pills (white or colored)
- Labels each pill with a letter indicating its color (W, R, B, G, Y, O, etc.)
- Detects empty blister cells (slots where a pill is missing)
- Detects color anomalies (if one pill looks different from the rest)
- Shows a visual result with outlines drawn around each pill

---

## 2. Libraries Used

| Library | Role |
|---|---|
| OpenCV (cv2) | All image processing: reading images, color conversion, thresholding, contour detection |
| NumPy (np) | Fast math on pixel arrays |
| Tkinter | The graphical user interface (the window showing results) |
| argparse, pathlib | Handling file paths and command-line arguments |
| dataclasses | Structured containers to pass data between pipeline steps |

---

## 3. Data Structures (lines 15–56)

Three data classes act as containers that carry results between steps:

**ProcessedImages** (line 16): The final output of the whole pipeline. Contains:
- original_bgr: raw image loaded from disk
- white_balanced_bgr: color-corrected image
- mask: binary image where white = pill pixels
- annotated_bgr: image with red outlines drawn around detected pills
- pill_count: the final pill count
- empty_cell_count: number of missing pills
- color_anomaly: True if pills don't all look the same

**PillColorClass** (line 34): An enum with 3 possible values — WHITE, COLORED, UNKNOWN. The entire detection strategy depends on which value this is.

**WhitePillResult** (line 52): Holds intermediate results of the white pill sub-pipeline including debug images for each step.

---

## 4. The Full Pipeline — process_image() (lines 986–1104)

This is the master function that runs everything in order when an image is loaded.

### Step 1 — Load the image (line 1007)
```
bgr = cv2.imread(str(image_path))
```
Reads the image from disk. OpenCV uses BGR channel order (Blue, Green, Red) instead of RGB.

### Step 2 — Rectify the pack (line 1011)
```
rectified = _rectify_pack(bgr, (800, 500))
```
**Problem:** The blister pack may be photographed at an angle — tilted or skewed.
**Solution:** Detect the outline of the whole pack and apply a perspective warp to make it look like the camera was directly above it, producing a clean 800×500 pixel crop.

How _rectify_pack() works (lines 285–331):
1. Resize image to 700px wide for faster processing
2. Convert to HSV, take the brightness (V) channel
3. Blur + Otsu threshold → binary image separating pack from background
4. Find contours, take the largest one (that is the pack)
5. Fit a minimum-area rectangle (cv2.minAreaRect) around it → 4 corner points
6. Apply perspective transform (cv2.getPerspectiveTransform) mapping those 4 tilted corners to a flat rectangle
7. Rotate 90° if needed to match landscape orientation

### Step 3 — White balance correction (line 1012)
```
white_balanced, _ = correct_white_balance(rectified)
```
**Problem:** Cameras introduce a color cast. Warm lighting makes everything look orange/yellow. Cool lighting makes everything look blue. This distorts pill colors.
**Solution:** Use the blister pack's metallic foil as a neutral reference. The foil is physically gray, so in a perfect image its R, G, B values should be equal. Measure the actual foil color, compute per-channel gain factors, and apply them to every pixel.

Example: if the foil measures R=200, G=170, B=160 and the target is 180:
- gain_R = 180/200 = 0.9 (reduce red)
- gain_G = 180/170 = 1.06 (slightly boost green)
- gain_B = 180/160 = 1.125 (boost blue)

These gains correct the entire image before any color classification happens. This must happen before color classification because a camera color cast could make white pills look yellowish and trigger a wrong classification.

Special case (lines 782–784): if the foil itself is gold or copper colored (mean_r - mean_b > 40), correction is skipped because that foil is not a neutral reference.

How correct_white_balance() works (lines 748–795):
1. Convert to HSV and find foil pixels: medium brightness (80–230), low saturation (≤ 60)
2. Check that enough foil is visible (at least 10% of image)
3. Measure average R, G, B of foil pixels
4. Compute gain per channel: gain = target_gray (180) / actual_mean
5. Multiply every pixel by the gains, clamp to 0–255

### Step 4 — Detect blister contour (line 1014)
```
blister_contour = detect_blister_contour(rectified)
```
**Problem:** We want to restrict pill detection to the actual blister area, not surrounding labels or background.
**Solution:** Find the rectangular outline of the blister pack.

How detect_blister_contour() works (lines 336–361):
1. Convert to LAB color space — separates lightness from color, better for edge detection
2. Blur the L (lightness) channel, then apply adaptive thresholding: compare each pixel to its local neighborhood average instead of one global value
3. Also threshold on gray > 235 to exclude pure-white regions (labels, packaging)
4. Morphological close + open to solidify the contour
5. Find contours, take the largest, wrap in convex hull, fit minimum-area rectangle → 4 corner points returned

### Step 5 — Classify pill color (line 1013)
```
color_result = classify_pill_color(white_balanced)
```
**Problem:** White pills need a completely different detection method than colored pills. We must decide which path to take.
**Solution:** Analyze the saturation of non-dark pixels across the image.

How classify_pill_color() works (lines 800–864):
1. Convert to HSV
2. Exclude very dark pixels (Value ≤ 40) — those are shadows
3. Among the remaining pixels, count those with Saturation > 45
4. If that fraction exceeds 4% of sampled pixels → COLORED
5. Otherwise → WHITE
6. If COLORED: build a saturation-weighted hue histogram to find the dominant hue

S (Saturation) = how vivid/colorful a pixel is. 0 = gray, 255 = pure vivid color.
V (Value) = brightness. 0 = pure black, 255 = pure white.

---

## 5. White Pill Pipeline (lines 1046–1077)

Runs when color_class == WHITE.

### Step 5a — Otsu brightness threshold inside blister (lines 191–204)
```
thresh = _otsu_in_mask(gray, search, brightness_floor)
white_mask = (gray >= thresh).astype(np.uint8) * 255
white_mask = cv2.bitwise_and(white_mask, search)
```
Otsu's method automatically finds the threshold value that best separates two groups of pixels (bright pills vs darker foil) by minimizing variance within each group. The result is never allowed below brightness_floor (205 by default) because white pills are genuinely bright — if Otsu returns 150, we still use 205.
The mask is then AND'd with the blister contour so only pixels inside the blister are considered.

Result (White mask panel): pills appear as white blobs, but the foil's dotted texture also partially appears as white speckles.

### Step 5b — Morphological cleanup and hole fill (lines 207–209)
```
cleaned = _morph_cleanup(white_mask, morph_open_iter, morph_close_iter, morph_kernel)
cleaned = _fill_holes(cleaned)
```
Two morphological operations:
- MORPH_OPEN (erode then dilate): removes small noise speckles (the foil dots are too small to survive erosion)
- MORPH_CLOSE (dilate then erode): fills small dark gaps inside pills (score lines, logos on pill surface)

Then _fill_holes() flood-fills from the image corners inward, filling any dark region completely surrounded by white. This makes pills solid filled discs.

Result (Cleaned mask panel): clean solid white circles, foil speckles gone.

### Step 5c — Distance transform and Watershed (lines 212, 98–129)
```
pill_mask, dist_vis = _watershed_separate(cleaned, bgr, separation)
```
**Problem:** Two touching pills appear as one connected blob.
**Solution:** Distance transform + Watershed algorithm.

1. Distance transform (cv2.distanceTransform): for every white pixel, compute its distance to the nearest black pixel (nearest edge). Pill centers are furthest from any edge → they become bright peaks in the resulting image.
2. Threshold at 40% of the maximum distance value → only definite pill-center pixels remain (sure foreground)
3. Dilate the mask → sure background
4. sure_background minus sure_foreground = unknown boundary region between pills
5. Watershed: treat the distance map as a topographic map. Flood fills from each center peak outward simultaneously. Where two floods meet = boundary between two pills. The algorithm draws dividing lines there.

Result (Distance transform panel): glowing blobs, brightest at each pill center, fading to black at edges.

### Step 5d — Contour filtering, first pass (lines 216–221)
```
count, contours = _filter_contours(pill_mask, min_area_ratio, max_area_ratio, min_solidity)
```
Keep only contours where:
- Area is between 0.3% and 20% of the image (removes tiny noise and giant blobs)
- Solidity ≥ 0.50: solidity = area / convex hull area. A very irregular jagged shape has low solidity. Pills are smooth and round so solidity should be high.

Result (Detected pills pre-filter panel): pills with red outlines. Some may still look jagged.

### Step 5e — Contour filtering, second pass (line 1063)
```
contours = _filter_white_contours_by_size(contours, white_balanced.shape)
```
Compares every contour to the median size of all detected contours in this image. Rejects anything:
- Too long and flat (glare streaks, foil edge reflections) — aspect ratio > 4.0
- Much bigger or smaller than the typical pill

Result (Final detected pills post-filter panel): only genuine pills remain, labeled W.

---

## 6. Colored Pill Pipeline (lines 1026–1044)

Runs when color_class == COLORED. Cannot use brightness like white pills because colored pills and foil can have similar brightness. Instead uses three combined signals.

### Step 6a — Segmentation using three signals combined (lines 891–949)

**Signal 1: CLAHE + Adaptive Threshold (lines 901–911)**
- Convert to LAB color space, take L (lightness) channel
- Apply CLAHE (Contrast Limited Adaptive Histogram Equalization): enhances local contrast tile by tile without over-brightening. Makes subtle edges between pill and foil more visible.
- Gaussian blur to smooth noise
- Adaptive threshold: for each pixel, compare to weighted average of its 35×35 neighborhood. Pixels darker than their surroundings → marked as pill.

**Signal 2: Saturation mask (lines 913–916)**
```
s_mask = (s_channel >= 35).astype(np.uint8) * 255
```
Any pixel with saturation ≥ 35 is vivid/colored → likely a pill. The foil is achromatic (saturation ≈ 0) so this directly isolates pill pixels by color vividness.

**Signal 3: Dark mask (lines 917)**
```
dark_mask = (v_channel <= 80).astype(np.uint8) * 255
```
Note: S and V are completely separate channels. 35 and 80 are not comparable numbers — saturation measures colorfulness, value measures brightness. These are different units.
Some colored pills (dark blue, dark red) are very dark. At very low brightness, saturation values become unreliable. The dark mask catches these pills regardless of color: if brightness ≤ 80, flag it as pill. The foil is reflective and never truly dark, so this is safe.

**Combining (lines 918–919):**
```
thresh = cv2.bitwise_or(adaptive, s_mask)
thresh = cv2.bitwise_or(thresh, dark_mask)
```
A pixel is a pill if ANY ONE of the three signals fires.

**Border inversion check (lines 921–930):**
Look at the outermost 5% border of the image. If more than 50% of those pixels are activated, the background was detected instead of the pills. Flip the entire mask automatically.

**Morphological cleanup + Watershed:** Same as white pills — open to remove noise, close to fill gaps, distance transform, watershed to split touching pills.

### Step 6b — Counting and filtering (lines 952–981)
```
count, contours = _count_pills(pill_mask, white_balanced, min_area_ratio, min_circularity, stddev_max)
```
Keep contours that pass:
- Minimum area: removes tiny noise
- Circularity = 4π × area / perimeter²: pills are round. Perfect circle = 1.0. Threshold is 0.15 (very lenient, allows ovals). Very jagged shapes score much lower and are rejected.
- Standard deviation of brightness inside contour (optional): very high variation = probably background texture, not a pill.

---

## 7. How Individual Pill Colors Are Detected — _pill_color_letters() (lines 633–669)

This runs per pill, not per image. It assigns a color letter to each detected pill.

For each pill contour:
1. Create a pixel mask for just that one pill (zero out everything else)
2. Filter to only vivid pixels inside that pill: saturation ≥ 40 (removes glare/reflection spots)
3. Compute the circular mean of hue values across those pixels

Why circular mean? Hue is an angle (0°–180° in OpenCV representing 0°–360°). Normal averaging fails near the wrap-around point — the average of 1° and 179° should be 0°, not 90°. Instead, each hue angle is converted to a point on the unit circle using complex numbers (exp(i×angle)), all points are averaged as vectors, and the angle of the result is the true circular mean.

4. Map the mean hue to a color name using hue_to_color_name() (lines 867–886):

| Hue range (0–360°) | Color name | Letter shown |
|---|---|---|
| 0–14° or 345–360° | Red | R |
| 15–37° | Orange | O |
| 38–69° | Yellow | Y |
| 70–154° | Green | G |
| 155–194° | Cyan | C |
| 195–254° | Blue | B |
| 255–284° | Purple | P |
| 285–344° | Pink/Magenta | P |

For white pills: if saturation inside the pill is below 40, the letter W is assigned directly without computing hue.

---

## 8. How Color Anomaly Is Detected — detect_color_anomaly() (lines 567–630)

Runs after all pills are labeled. Checks: are all pills the same color, or is there an intruder?

Requires at least 2 pills to compare. Three checks run in order:

### Check 1 — Saturation spike (lines 604–607)
```
median_sat = float(np.median(per_pill_sats))
if any(abs(s - median_sat) > 45 for s in per_pill_sats):
    return True
```
Compute the median saturation across all pills. If any single pill differs from the median by more than 45 → anomaly. Catches cream or white pills mixed into a colored batch (much lower saturation than neighbors).

### Check 2 — Color name mismatch (lines 612–614)
```
pill_names = [hue_to_color_name(h) for h in per_pill_hues]
if len(set(pill_names)) > 1:
    return True
```
Map every pill's hue to a color name. If they don't all map to the same name → anomaly. Specifically catches pills that are numerically close in hue but visually different colors, like yellow vs green (only ~30° apart — a numeric threshold alone would miss this).

### Check 3 — Hue distance fallback (lines 616–624)
```
consensus_angle = circular mean of all pill hues
for h in per_pill_hues:
    d = abs(h - consensus_hue)
    if min(d, 180.0 - d) > 25.0:
        return True
```
Compute the circular mean hue of all pills together as a consensus. If any individual pill's hue deviates from that consensus by more than 25° → anomaly. min(d, 180-d) handles the hue wrap-around correctly.

If all three checks pass → no anomaly → green light in the UI.
If any check triggers → anomaly detected → red light in the UI.

For white pills (lines 627–630): check if any pill has saturation more than 40 units above the median. If yes → a colored pill is mixed in → anomaly.

---

## 9. How Empty Cells Are Detected — detect_empty_cells() (lines 364–564)

The function never directly detects empty cells visually. Instead it infers where pills SHOULD be based on the grid structure, then checks if those positions are actually empty.

### Phase 1 — Measure detected pills (lines 388–415)
- Compute average pill area and radius (lines 388–392)
- Compute center of each pill using image moments: cx = m10/m00, cy = m01/m00 (lines 395–399)
- Compute minimum distance from any pill to the blister contour edge using pointPolygonTest (lines 407–415). This becomes the required standoff — no inferred empty cell can be closer to the edge than the closest real pill.

### Phase 2 — Build the grid (lines 427–467)
- Group pill centers into rows by clustering Y-coordinates: if two pills have Y values within 90% of avg_radius of each other, they are in the same row (lines 428–437)
- Column spacing = median of minimum x-gaps between consecutive pills per row. Using minimum is robust: if a pill is missing in the middle, the gap doubles, so taking the minimum ignores that inflated gap (lines 440–448)
- Row spacing = median of differences between consecutive row Y-averages (lines 452–457)
- Column positions = cluster all pill X-coordinates across all rows. Positions within 45% of col_spacing of each other belong to the same column (lines 460–467)

### Phase 3 — Extrapolate the full grid (lines 469–503)
Fill in positions that might be entirely missing:
- Interior gaps: if two detected rows are 2× row_spacing apart, insert a missing row between them (lines 469–475)
- Beyond detected range: walk outward from the first and last row/column, adding one spacing at a time, as long as the position is still inside the blister contour (lines 480–503)

### Phase 4 — Verify each candidate position (lines 513–562)
For every grid position with no detected pill, run these checks in order:
1. Skip if a pill center is already nearby (lines 518–522)
2. Skip if the position overlaps any real pill contour boundary (lines 528–532)
3. Skip if outside the blister contour or too close to its edge (line 535)
4. Foil color check (lines 542–554): sample a tiny patch of pixels at that position. Check that median saturation ≤ 70 (achromatic = foil) and brightness is between 50–220 (not a shadow, not a white pill). Only if it looks like bare foil → confirmed empty cell.
5. Draw a circle the size of an average pill at that position → orange outline on the final image (lines 558–562)

---

## 10. Functions Reference Table

| Function | Lines | What it does |
|---|---|---|
| _rectify_pack() | 285–331 | Detects the pack outline and applies perspective warp to flatten a tilted image |
| correct_white_balance() | 748–795 | Uses the foil as a neutral reference to remove camera color cast |
| classify_pill_color() | 800–864 | Decides WHITE or COLORED based on saturation of non-dark pixels |
| detect_blister_contour() | 336–361 | Finds the rectangular outline of the blister area using adaptive thresholding on the LAB L channel |
| segment_white_pills() | 161–237 | Full white pill pipeline: Otsu threshold, morph cleanup, fill holes, watershed, contour filter |
| _otsu_in_mask() | 66–74 | Computes Otsu threshold only on pixels inside the blister contour |
| _morph_cleanup() | 77–86 | Morphological open + close to remove noise and fill gaps |
| _fill_holes() | 89–95 | Flood-fills from corners to fill dark holes inside pill blobs |
| _watershed_separate() | 98–129 | Distance transform + watershed to split touching pills |
| _filter_contours() | 132–158 | Keeps contours within area bounds and above solidity threshold |
| _segment_pills() | 891–949 | Colored pill segmentation using CLAHE + adaptive threshold + saturation mask + dark mask |
| _count_pills() | 952–981 | Filters colored pill contours by area, circularity, brightness stddev |
| _filter_white_contours_by_size() | 694–743 | Rejects glare streaks and size outliers by comparing to median pill size |
| _pill_color_letters() | 633–669 | Computes circular mean hue per pill and maps to a color letter |
| hue_to_color_name() | 867–886 | Maps an OpenCV hue value (0–179) to a human-readable color name |
| detect_color_anomaly() | 567–630 | Checks if all pills are the same color using saturation, color name, and hue distance |
| detect_empty_cells() | 364–564 | Infers the pack grid from pill positions, extrapolates it, then verifies empty slots with foil color check |
| _draw_pill_letters() | 672–683 | Draws a color letter on each pill in the annotated image |
| process_image() | 986–1104 | Master pipeline function: runs all steps in order and returns ProcessedImages |
| ImagePipelineUI | 1128–1346 | Tkinter GUI: displays step-by-step panels, pill count, color label, status lights |

---

## 11. UI and Display

The UI (ImagePipelineUI, lines 1128–1346) shows the pipeline steps visually as panels.

For white pills, 8 panels are shown:
1. Original
2. White balanced
3. White mask (pre-filter) — Otsu threshold result
4. Cleaned mask (pre-filter) — after morph cleanup and hole fill
5. Distance transform (pre-filter) — glowing centers
6. Detected pills (pre-filter) — after first contour filter
7. Final mask (post-filter) — after size filter
8. Final detected pills (post-filter) — final result with W labels

For colored pills, 6 panels are shown:
1. Original
2. White balanced
3. Pill sample mask — shows which pixels were flagged as colored
4. Mask (pills white) — the pill segmentation mask
5. Binary — inverted mask
6. Detected pills — final result with color letters

Two status lights appear in the top bar:
- Empty cells light: green = all slots filled, red = at least one empty slot detected
- Color anomaly light: green = all pills same color, red = at least one pill looks different

Navigation buttons (Previous / Next) cycle through all images in the images/ folder next to the script.
