# PCD Assignment 02: Image Enhancement

## Identity

| | |
|---|---|
| **Name** | Muhammad Fathan Romiza |
| **Student ID** | 25/560518/PA/23623 |
| **University** | Universitas Gadjah Mada |
| **Course** | Digital Image Processing (Pengolahan Citra Digital) |

## About This Project

This repository contains the second assignment for the Digital Image Processing course. The assignment focuses on **image enhancement**: implementing point operations and spatial operations **from scratch**, building an **automatic image classifier** based on statistical features, and selecting an enhancement method according to that diagnosis.

The central premise is that no single enhancement method works universally. Dark, bright, low-contrast, and blurred images each require a different treatment, so applying one method blindly across all images can degrade quality rather than improve it. Fourteen 512×512 test images are used, and every result is judged with quantitative metrics rather than visual impression alone.

## Objective

- Implement point operations (**negative, logarithmic, gamma, contrast stretching, histogram equalization, thresholding**) from scratch.
- Implement spatial operations (**mean filter, Gaussian filter, unsharp masking, Laplacian sharpening**) from scratch using convolution.
- Automatically **classify image type** from statistical features and branch into the appropriate processing pipeline.
- Evaluate results using four metrics: **mean**, **standard deviation**, **Shannon entropy**, and **Laplacian variance**.
- Identify failure modes of the classifier and the operator selection, and document corrective recommendations.

## Theoretical Basis

### Point Operations

Depend only on the pixel's own value — no neighborhood involved.

| Method | Function | Description |
|---|---|---|
| **Negative** | `negative()` | `out = (L - 1) - r`, with `L = 256`. Inverts intensity. |
| **Logarithmic** | `log()` | `out = c · log(1 + r)`, with `c = (L - 1) / log(1 + R_max)`. Compresses the dynamic range, brightens dark regions. |
| **Gamma** | `gamma()` | `out = c · r^γ`, with `c = (L - 1) / (R_max^γ)`. Default `γ = 0.4` (brightening). |
| **Contrast Stretching** | `stretch()` | Linear min–max rescaling to a target range `[a, b]`, default `[0, 255]`. |
| **Histogram Equalization** | `hist()` | CDF-based lookup table: `lut = round((cdf - cdf_min) / (total - cdf_min) · 255)`. |
| **Thresholding** | `grey()` | Intensity band binarization: pixels within `[low, high]` → 255, otherwise → 0. |

### Spatial Operations

Involve neighboring pixels through convolution kernels.

| Method | Function | Kernel / Formula |
|---|---|---|
| **Mean Filter** | `smooth()` | 3×3 box kernel, `ones(3,3) / 9`. |
| **Gaussian Filter** | `gauss()` | 3×3 Gaussian kernel built by `gaussian_kernel(size=3, sigma=0.5)`, applied via `cv2.filter2D`. |
| **Unsharp Masking** | `avg()` | `sharpened = img + amount · (img - blurred)`, with `amount = 0.7` and a 3×3 box blur. |
| **Laplacian Sharpening** | `lap()` | 8-neighbor Laplacian kernel, then `sharpened = img + 0.2 · lap`. |

> **Naming note:** `avg()` is *not* an averaging filter — it is unsharp masking. `grey()` is thresholding, and `smooth()` is the mean filter. The function names are kept as-is for fidelity with the submitted notebook.

### Evaluation Metrics

| Metric | What it measures |
|---|---|
| **Mean** | Brightness level. |
| **Standard Deviation** | Global contrast. |
| **Shannon Entropy** | Information content. |
| **Laplacian Variance** | Edge sharpness / focus (higher = sharper). |

## Automatic Classification

The classifier converts the input to grayscale, computes three features, then walks a **sequential `if-elif` chain**:

| Order | Condition | Label |
|---|---|---|
| 1 | `mean < 85` | `gelap` (dark) |
| 2 | `mean > 170` | `terang` (bright) |
| 3 | `std < 45` | `kontras rendah` (low contrast) |
| 4 | `lap_var < 100` | `buram` (blurred) |
| 5 | otherwise | `normal` |

Each label triggers a different branch inside `eksperiment()`:

| Label | Processing applied |
|---|---|
| `gelap` | Logarithmic, Gamma |
| `terang` | Gamma, Negative |
| `kontras rendah` | Contrast Stretching, Histogram Equalization, Thresholding `[0, 100]` |
| `buram` | Mean Filter, Gaussian, Unsharp Masking |
| `normal` | None — original is displayed unchanged |
| `noise` | Mean Filter, Gaussian, Unsharp Masking (**unreachable branch**) |

## How to Run

This project is built as a **Google Colab notebook** (`.ipynb`). To run it:

1. Open [Google Colab](https://colab.research.google.com/).
2. Click **File** > **Upload notebook**.
3. Upload the `PCD_Assignment02.ipynb` file from this repository.
4. Run all cells sequentially (**Runtime** > **Run all**).
5. The notebook fetches the 14 test images directly from Google Drive, resizes each to 512×512, classifies it, and displays the enhancement results with `matplotlib`.

> **Note:** No local installation is required. All dependencies are pre-installed in the Google Colab environment.
>
> **Note:** The test images are referenced by Google Drive file ID and downloaded at runtime via `requests`. The Drive links must remain publicly accessible, otherwise `response.raise_for_status()` will abort the run.

## Repository Structure

```
pcdassignment2/
├── PCD_Assignment02.ipynb        # Google Colab notebook (main code)
├── PCD_Assignment02_Report.pdf   # Written analysis report
└── README.md                     # This file
```

## Dependencies

- Python 3
- NumPy
- OpenCV (`cv2`)
- Pillow (`PIL`)
- Matplotlib
- Requests
- IPython (`display`)

## Findings and Known Limitations

The submitted notebook is the working baseline. Its failure modes were diagnosed in the report rather than patched in code, and are documented here deliberately:

1. **The blur threshold is too loose.** Calibration by progressive Gaussian blurring showed Laplacian variance collapsing from thousands to tens at only `sigma 1.0–1.5`. The threshold of `100` therefore failed to catch visibly blurred images — **not a single image was classified as `buram`**. An appropriate threshold is around **250–300**.
2. **The `if-elif` ordering misprioritizes.** The most blurred image in the entire dataset fell into the `kontras rendah` branch because the `std` check runs before the blur check, so the root cause was never addressed.
3. **Histogram equalization damages color.** `hist()` equalizes a *combined* histogram across all three RGB channels, causing significant color shift (inter-channel differences increased 2–3×) and rainbow noise. It should be applied only to the **luminance channel** (e.g. YCrCb).
4. **`lap()` is never called.** The Laplacian sharpening function is implemented but unreferenced, even though the `buram` branch is precisely where it is needed.
5. **The `noise` branch can never be triggered.** `classify_image()` never returns `'noise'`, making that branch dead code.
6. **Some operators add no information.** The negative operation in the `terang` branch and thresholding in the `kontras rendah` branch produced stagnant or collapsed entropy, i.e. they destroyed information without adding any.
7. **Gamma polarity is wrong for bright images.** The default `γ = 0.4` brightens, but the `terang` branch needs to darken. This caused severe saturation — reaching **43.9% saturated pixels** in one image — while systematically reducing contrast and entropy.
8. **`normal` is a blind spot.** The four images labeled `normal` received no processing at all, even though some were visibly blurred.

Classification outcome across the 14 images: **4 bright, 4 dark, 4 normal, 2 low-contrast, 0 blurred**.

> **Metrics note:** the notebook itself computes `mean`, `std`, and Laplacian variance for the classifier. Shannon entropy and the full four-metric comparison are reported in `PCD_Assignment02_Report.pdf`.

## Recommendations

- Raise the Laplacian variance threshold to **250–300**.
- Prioritize the blur check, or replace the chain with a **weighted composite score**.
- Apply **adaptive gamma** (`γ > 1` for bright images, `γ < 1` for dark images).
- Perform histogram equalization **only on the luminance channel**.
- **Activate `lap()`** for the `buram` branch.
- **Remove unused classification branches** (the `noise` case).
- **Cap the maximum value** in the logarithmic transform to prevent excessive saturation.

## Conclusion

All operators were successfully implemented from scratch. Automatic classification worked effectively for dark, bright, low-contrast, and normal images, but failed entirely for blurred images due to a loose threshold. There is no universal enhancement method — suitability depends on image type, and a misdiagnosis directly leads to an inappropriate method selection. Enhancement success must be judged using multiple metrics, because an increase in brightness can be accompanied by a decrease in contrast that makes the image look worse rather than better.