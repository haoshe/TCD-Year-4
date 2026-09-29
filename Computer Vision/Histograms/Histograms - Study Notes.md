# Histograms: Study Notes

*Computer Vision, TCD Year 4. Based on the lecture slides (from Dawson-Howe, *A Practical Introduction to Computer Vision with OpenCV*, 2014) and the two lecture transcripts.*

**Topics**
1. 1D histograms (greyscale), including smoothing
2. Colour histograms (1D per channel) and colour spaces
3. 3D histograms and quantisation
4. Histogram equalisation
5. Histogram comparison (metrics and Earth Mover's Distance)
6. Histogram back projection
7. Summary table, exam tips and practice questions

---

## 0. Why histograms?

A histogram is a **summary of image content**. It counts **how many pixels have each value** and **ignores where they are**.

- **Motivating example:** a person walking through surveillance video. If the person can be segmented out, you can build a colour histogram of the top, middle and bottom sections of their body, then describe or match their clothes ("black jacket, jeans, grey hair"). This helps with **tracking** and with **describing people to a human operator**.
- Histograms are usually most useful for a **region** (an object or a person), not for the whole image.

> **中文解释：** 直方图就是"统计每个灰度/颜色值出现了多少次"，完全不管像素在图像中的**位置**。所以它是一种对图像内容的"摘要"。整张图的直方图往往用处不大，但对**分割出来的物体**（比如一个人、一辆车）计算直方图就很有用，可以用来识别衣服颜色、跟踪目标。

---

## 1. 1D Histograms (greyscale)

### Definition and algorithm
A histogram gives the **frequency of each brightness value**.

```
Initialise:  h(z) = 0          for all values of z   (z = 0..255)
Compute:     h(f(i,j))++       for all pixels (i,j)
```

- The x-axis shows the grey levels 0–255 (the **bins**). The y-axis shows the **number of pixels** with that grey level.
- Summing all bins gives `rows × cols`, the total number of pixels.

### Is it useful?
| Property | Meaning |
|---|---|
| **Global information** | Describes the whole image or region, with no spatial information |
| **Useful for classification?** | Possibly. For example, build a histogram per segmented snooker ball and classify it as red, pink, blue, etc. |
| **Not unique** | Shuffling all the pixels gives **exactly the same histogram**. Many different images share one histogram. |

> **中文解释：** 直方图**不唯一**：把图像像素随机打乱，直方图完全不变。所以不同的图可能有相同的直方图。

### OpenCV (C++, from slides)
```cpp
MatND gray_histogram;
const int* channel_numbers = { 0 };
float channel_range[] = { 0.0, 255.0 };
const float* channel_ranges = channel_range;
int number_bins = 256;                       // one bin per grey level
calcHist( &gray_image, 1, channel_numbers, Mat(),
          gray_histogram, 1, &number_bins, &channel_ranges );
```
`calcHist` needs: the image(s), **channel indices**, a mask (`Mat()` means none), the output, the dimensionality, the **number of bins** and the **value ranges**.

#### Understanding the `calcHist` code

**What the code does.** It counts how many pixels in `gray_image` have each grey level (0, 1, 2, … 255) and stores the counts in `gray_histogram`. When it finishes, `gray_histogram.at<float>(100)` holds the number of pixels whose value is 100.

**Why the arguments look odd.** `calcHist` is a general function. It can build a histogram from **several images at once**, over **several channels**, in **several dimensions**. For example, a 2D histogram of Hue against Saturation takes two channels, two bin counts and two ranges. Because of that, almost every argument is an **array** or a **pointer to an array**, even when you only need one of something. That's where the `&`, `*` and `{ }` come from.

**Line by line**

```cpp
MatND gray_histogram;
```
This is the output. `MatND` is an old name for `cv::Mat`, so a plain `Mat` works too. After the call it's a 256×1 matrix of `float`, one count per bin.

```cpp
const int* channel_numbers = { 0 };
```
This says which channel(s) to count. A grey image has only channel 0. It's meant to be an array holding one element, `{0}`. **But this line is subtly wrong:** it declares a *pointer* and sets it to `0`, which is a null pointer, not an array containing 0. It only works because `calcHist` treats a null `channels` argument as "use channels 0, 1, 2…". Write it like this instead:
```cpp
int channel_numbers[] = { 0 };
```

```cpp
float channel_range[] = { 0.0, 255.0 };
const float* channel_ranges = channel_range;
```
This gives the range of pixel values to count, as `{min, max}`. `calcHist` wants **one range per dimension**, which is an array of arrays (`const float**`). So you make one range, point to it, and later pass `&channel_ranges`, a pointer to that pointer. With a 2D histogram you'd have `const float* ranges[] = { hue_range, sat_range };`.

**Second bug:** the upper bound is **exclusive**. With `{0, 255}`, pixels with value 255 fall outside the range and **are not counted**, so bin 255 always stays empty. Use this:
```cpp
float channel_range[] = { 0.0f, 256.0f };
```

```cpp
int number_bins = 256;
```
This is how many bins (buckets) to split the range into. 256 bins over 0–256 means each bin is exactly one grey level wide. With 64 bins, each bin would hold 4 grey levels (0–3, 4–7, …).

**The call itself**

```cpp
calcHist( &gray_image,      // images:   pointer to an array of input images
          1,                // nimages:  how many images are in that array
          channel_numbers,  // channels: which channel(s) to use -> {0}
          Mat(),            // mask:     empty Mat = use every pixel
          gray_histogram,   // hist:     output
          1,                // dims:     1-D histogram (grey level only)
          &number_bins,     // histSize: bins per dimension (array of dims ints)
          &channel_ranges );// ranges:   value range per dimension (array of dims ranges)
```

- **`&gray_image`, `1`**: the function expects an array of images plus a count. You have one image, so you pass its address and say 1.
- **`Mat()` mask**: an empty matrix means "no mask". If you passed an 8-bit image the same size as the input, only pixels where the mask is non-zero would be counted. That's useful for getting the histogram of just one region.
- **`1` dims**: a 1D histogram, since there's one variable: brightness.
- **`&number_bins`, `&channel_ranges`**: arrays with one entry per dimension. With one dimension, the address of a single variable works as a one-element array.

**Corrected version**

```cpp
Mat gray_histogram;
int   channel_numbers[] = { 0 };
float channel_range[]   = { 0.0f, 256.0f };   // upper bound is exclusive
const float* channel_ranges = channel_range;
int   number_bins = 256;

calcHist(&gray_image, 1, channel_numbers, Mat(),
         gray_histogram, 1, &number_bins, &channel_ranges);

// e.g. how many pixels are pure black?
float black_count = gray_histogram.at<float>(0);
```

The two fixes are to make `channel_numbers` a real array and to set the upper range to 256 so white pixels (255) are counted.

> **中文解释：** `calcHist` 是通用函数（可处理多张图、多通道、多维直方图），所以几乎每个参数都是**数组**或**指向数组的指针**，即使只有一个元素也要这样传。幻灯片代码有两个小问题：① `const int* channel_numbers = { 0 };` 其实是**空指针**，不是数组，应写成 `int channel_numbers[] = { 0 };`；② 范围的**上界是开区间**，`{0, 255}` 会漏掉灰度 255 的像素，应写成 `{0, 256}`。

Python equivalent:
```python
hist = cv2.calcHist([gray], [0], None, [256], [0, 256])
```

### 1.1 Smoothing a histogram
- **Local minima and maxima** are useful (for example, for choosing thresholds), but raw histograms are very **noisy**.
- The fix is to **smooth** the histogram with a local average:

$$h_{new}(v) = \frac{h(v-1) + h(v) + h(v+1)}{3}$$

**Problem: what happens at the ends (v = 0 and v = 255)?** Options:

| Option | What it does | OpenCV border type |
|---|---|---|
| Do not compute | Leave the end values blank or unchanged | — |
| Wraparound | Treat the histogram as circular: use h(255) as the neighbour of h(0) | `BORDER_WRAP` |
| Duplicate values | Copy the end value outward | `BORDER_REPLICATE` |
| Reflect values | Mirror the values at the boundary | `BORDER_REFLECT` |
| Constant values | Assume a fixed value (for example 0) outside | `BORDER_CONSTANT` |

- **Key point from the lecture:** all of these options are used in OpenCV image processing. **Boundary effects are very noticeable in images**, and the more processing you apply, the more they can damage the result.
- If you average only the two available values at the ends, a **normalised histogram no longer sums to 1**.

> **中文解释：** 平滑 = 取相邻三个值的平均来去噪。难点在**边界**：第 0 格和第 255 格左边/右边没有邻居，可以不计算、环绕 (wrap)、复制 (duplicate)、镜像 (reflect) 或补常数 (constant)。这个"边界问题"在后面二维图像滤波中会反复出现，很重要。

Slide code (it deliberately skips both ends, i.e. "do not compute"):
```cpp
MatND smoothed_histogram = histogram.clone();
for (int i = 1; i < histogram.rows - 1; ++i)
    smoothed_histogram.at<float>(i) =
        (histogram.at<float>(i-1) + histogram.at<float>(i) + histogram.at<float>(i+1)) / 3;
```

---

## 2. Colour Histograms (1D per channel)

- Compute a **separate 1D histogram for each channel**.
- The result **depends on the choice of colour space**.

Example from the lecture: a snooker table image (slide 8).

| Colour space | What the histograms show |
|---|---|
| **RGB** | **Red:** a huge peak at **low** values (the green cloth has almost no red) and a peak at the **high** end (red balls, pink ball, brown ball and skin all have strong red). **Green:** a strong peak from the **green cloth**. **Blue:** a response from the blue ball, and also from the cloth, because it is a "bluey-green". |
| **CMY** | C = 255 − R, M = 255 − G, Y = 255 − B, the **inverse of RGB**. Its histograms are the RGB ones **flipped horizontally**, so there is **no new information**. |
| **YUV** | Y = luminance (like a greyscale image); U and V = chrominance. U and V have **no obvious meaning**, so the histograms are **hard to interpret**. |
| **HLS** | **L:** the intensity distribution. **S:** a large peak at **high saturation**, because the image is very colourful. **H (hue):** **the most useful channel**. The histogram is drawn in colour (red → yellow → green → cyan → blue → magenta → red), and the huge **green peak** is the **snooker table cloth**. |

**Colour-space background that matters for histograms** (from the *Colour Images* lecture in the Images topic):
- **OpenCV stores images as BGR**, not RGB, so channel 0 is **blue**.
- **Hue is circular.** Red sits at **both ends** of the hue range, so 0 and the maximum are neighbours. On a hue histogram, a red object can appear as **two peaks at opposite ends**. (Smoothing with **wraparound** at the ends makes sense here.)
- **OpenCV ranges:** H = **0–179** (degrees ÷ 2), and L and S = **0–255**. Use a hue range of `[0, 180]` in `calcHist`, not `[0, 256]`.
- **Hue and saturation are unreliable** when luminance is **very low or very high** (near black or white), or when saturation is **very low** (greys). In those cases the hue is essentially noise. Consider masking out such pixels before building a hue histogram.
- **CMY is not supported directly in OpenCV.** It is only used for printing, and it adds nothing for processing.

> **中文解释：** 彩色图像可以每个通道单独算一个一维直方图。颜色空间的选择很关键：
> - **RGB**：红色通道低端大峰 = 绿色桌布几乎没有红色；高端峰 = 红球、粉球、棕球、皮肤。
> - **CMY** = 255 − RGB，直方图只是 RGB 的水平翻转，**没有新信息**。
> - **YUV** 的 U/V 没有直观含义，不好解读。
> - **HLS 中的色调 H 最有意义**（绿色大峰 = 台球桌的绿色桌布）。
> - 注意：色调是**环形**的（红色同时在 0 和最大值两端）；OpenCV 中 H 的范围是 **0–179**；亮度过低/过高或饱和度很低时，色调基本是**噪声**。

OpenCV (C++): split the image into channels, then call `calcHist` on each, here with **64 bins**:
```cpp
vector<Mat> channels(image.channels());
split(image, channels);
int number_bins = 64;
for (int chan = 0; chan < image.channels(); chan++)
    calcHist(&(channels[chan]), 1, channel_numbers, Mat(),
             histogram[chan], 1, &number_bins, &channel_ranges);
```
**Fewer bins** give less noise and are easier to process, but are **less precise**. There is a trade-off.

---

## 3. 3D Histograms

### Why 3D?
- **Channels are not independent.** Hue, saturation and intensity are related (think of "dark red" or "light blue").
- **Better discrimination** comes from considering **all channels at once**, so that you see combinations of values.
- **Slide 10 example (fruit):** plotted in 3D RGB space, the red/orange fruit form one **cluster** (high R) and the green apples form another (high G). Separate 1D histograms would mix these combinations together, while a 3D histogram keeps them apart.

### Problem: the number of cells
With 3 channels, each cell corresponds to one (c1, c2, c3) combination:

| Bits per channel | Bins per channel | Total cells |
|---|---|---|
| 8 | 256 | 256³ = **16,777,216** |
| 6 | 64 | 64³ = 262,144 |
| 4 | 16 | 16³ = 4,096 |
| 2 | 4 | 4³ = **64** |

- At 8 bits per channel, the histogram is usually **bigger than the image**, so it is no longer a summary.
- The solution is to **reduce quantisation** (use fewer bins per channel). You rarely go anywhere near 256 bins per channel.

> **中文解释：** 三个通道之间是相关的，所以用**三维直方图**同时统计 (R,G,B) 组合更有区分度。但 8 位时有 1677 万个格子，比图像本身还大，所以必须**降低量化级数**（比如每通道 4 位 = 16 个 bin，或 2 位 = 4 个 bin）。公式：格子数 = (每通道 bin 数)³。

### Road-sign example (slide 11)
**Takeaway:** even with only **2 bits per channel (64 cells)**, a 3D histogram still captures the main colours in the scene. The white sign, red border, dark arrow and green leaves each show up in their own cell. (How the picture is drawn doesn't need to be learned.)

### OpenCV (C++): a single call on the whole image
```cpp
int channel_numbers[] = { 0, 1, 2 };
int* number_bins = new int[image.channels()];
for (ch = 0; ch < image.channels(); ch++) number_bins[ch] = 16;   // same bins per channel (typical)
float ch_range[] = { 0.0, 255.0 };
const float* channel_ranges[] = { ch_range, ch_range, ch_range };
calcHist(&image, 1, channel_numbers, Mat(), histogram,
         image.channels(), number_bins, channel_ranges);
```
Differences from 1D: **all channel numbers are passed**, the image is **not split**, **dims = 3**, and there is **an array of bin counts** (one per channel).

#### Understanding the 3D `calcHist` code

This code builds **one 3D histogram** that counts **(B, G, R) combinations**, using **16 bins per channel**, so 16 × 16 × 16 = 4,096 cells. It follows the same pattern as the 1D code, but every "one of something" becomes "three of something".

```cpp
int channel_numbers[] = { 0, 1, 2 };
```
This says which channels to use: all three. In OpenCV a colour image is stored as **BGR**, so 0 = blue, 1 = green and 2 = red. Unlike the 1D slide code, this one is a real array, which is the correct way to write it.

```cpp
int* number_bins = new int[image.channels()];
```
This creates an array to hold **the number of bins for each channel**. `image.channels()` is 3, so it's an array of 3 ints. `new int[...]` creates the array at run time, because the size comes from the image.

```cpp
for (ch = 0; ch < image.channels(); ch++) number_bins[ch] = 16;
```
This fills the array with `{16, 16, 16}`, so each channel is split into 16 bins. Each bin covers 256 / 16 = 16 values (0–15, 16–31, …). This is the **reduce quantisation** idea: 16 bins = 4 bits per channel, so there are 4,096 cells instead of 16.7 million. (On the slide, `ch` isn't declared. It should be `for (int ch = 0; ...)`.)

```cpp
float ch_range[] = { 0.0, 255.0 };
```
This is the value range for **one** channel, as `{min, max}`. As in the 1D case, the upper bound is **exclusive**, so this should be `{0, 256}`. Otherwise pixels with value 255 aren't counted.

```cpp
const float* channel_ranges[] = { ch_range, ch_range, ch_range };
```
This is an array of **3 pointers**, one range per dimension. All three channels have the same range, so all three point to the same `ch_range`:
```
channel_ranges → [ ptr, ptr, ptr ]
                    ↓    ↓    ↓
                 [ 0, 256 ]   (the same array each time)
```

```cpp
calcHist(&image, 1, channel_numbers, Mat(), histogram,
         image.channels(), number_bins, channel_ranges);
```

| Argument | Meaning |
|---|---|
| `&image, 1` | one input image, the **whole colour image** (not split) |
| `channel_numbers` | use channels {0, 1, 2} |
| `Mat()` | no mask, so every pixel is counted |
| `histogram` | the output: a **16 × 16 × 16** array of counts |
| `image.channels()` | dims = **3**, a 3D histogram |
| `number_bins` | `{16, 16, 16}`, bins per dimension |
| `channel_ranges` | range per dimension |

`number_bins` and `channel_ranges` are passed **without `&`**, because they're already arrays. In the 1D code they were single variables, so they needed `&` to act as one-element arrays.

After the call, `histogram.at<float>(b, g, r)` gives the number of pixels whose blue, green and red values fall in bins `b`, `g` and `r`.

**Side by side with 1D**

| | 1D (grey) | 3D (colour) |
|---|---|---|
| Channels | `{0}` | `{0, 1, 2}` |
| Split image? | per-channel loop in the colour 1D case | **no**, one call on the whole image |
| dims | 1 | **3** |
| Bins | `&number_bins` (one int) | `number_bins` (array of 3) |
| Ranges | `&channel_ranges` (one pointer) | `channel_ranges` (array of 3 pointers) |
| Output size | 256 | 16 × 16 × 16 |

**Cleaner version**

```cpp
Mat histogram;
int   channel_numbers[] = { 0, 1, 2 };
int   number_bins[]     = { 16, 16, 16 };
float ch_range[]        = { 0.0f, 256.0f };
const float* channel_ranges[] = { ch_range, ch_range, ch_range };

calcHist(&image, 1, channel_numbers, Mat(), histogram,
         3, number_bins, channel_ranges);
```

This uses a plain array instead of `new`, so there's no memory to free. The slide never calls `delete[]`, which is a small memory leak.

> **中文解释：** 这段代码对整张彩色图（不拆分通道）计算一个**三维直方图**。`channel_numbers = {0,1,2}` 表示用 B、G、R 三个通道；`number_bins = {16,16,16}` 表示每个通道分 16 个 bin（降低量化，共 4096 个格子）；`channel_ranges` 是 3 个指针组成的数组，每个都指向同一个范围 `{0, 256}`。与一维相比：dims = 3，bins 和 ranges 都变成"每维一个"的数组，所以传参时**不用加 `&`**。

Python: `cv2.calcHist([img], [0,1,2], None, [16,16,16], [0,256,0,256,0,256])`

---

## 4. Histogram Equalisation

### Purpose
- Enhance **contrast** when the values are bunched together (example: a dark photo of the Campanile against a bright sky).
- It **helps humans, not computers**. It is for visualisation and enhancement.
- Humans can reportedly distinguish **700–900 grey levels** (based on research with medical professionals). We only use 256.
- **Goal:** a **flat histogram**, where every value is used equally often.

### What happens to colours?
- **Equalising R, G and B separately distorts the colours.**
- Instead, convert to a space such as HLS, **equalise only the luminance (L) channel**, and **leave hue and saturation alone**. You can also work on a greyscale image.

> **中文解释：** 直方图均衡化 = 把挤在一起的灰度值"拉开"，让直方图尽量平坦，从而增强对比度。**只对人眼有用，对计算机帮助不大。** 注意：不能对 RGB 三个通道分别均衡，否则颜色会失真；应该转到 HLS，**只均衡亮度通道 L**。

### Algorithm (a lookup table built from the cumulative histogram)
```
// h[x] = histogram of luminance values of image f(i,j)
pixels_so_far = 0
num_pixels    = image.rows * image.cols
for input = 0 to 255
    pixels_so_far = pixels_so_far + h[input]              // cumulative count
    LUT[input]    = (pixels_so_far * 256) / (num_pixels + 1)
// Apply the lookup table:
for every pixel f(i,j)
    g(i,j) = LUT[ f(i,j) ]
```
- `pixels_so_far / num_pixels` is the **fraction of pixels at or below this level** (the cumulative distribution, CDF).
- The output level is proportional to it. If half the pixels have been processed, the output is about **127–128**.
- **Why `+1`?** It keeps the output **≤ 255**, because 256 is not a valid value. Even at the last level, where pixels_so_far = num_pixels, 256·N/(N+1) < 256.

> **中文解释：** 核心思想：输出灰度 ∝ **累计直方图 (CDF)**。"已经处理了一半像素"就映射到约 128。分母加 1 是为了保证结果最大只到 255，不会越界到 256。

#### Understanding the equalisation algorithm

**The big idea: brightness by rank.** Equalisation gives each pixel a new brightness based on its **rank**: what fraction of the image is darker than or equal to it.
- A pixel darker than almost everything → output near **0**
- A pixel in the middle (half the image is darker) → output near **128**
- A pixel brighter than almost everything → output near **255**

It's like grading on a curve. If every student scored between 50 and 53, you rank them and spread the grades evenly from 0 to 100. The ranks stay the same, but the differences become visible. Because every output level gets roughly an equal share of the pixels, the output histogram becomes roughly **flat**, which is the goal of equalisation.

**Line by line**

```
pixels_so_far = 0
num_pixels    = image.rows * image.cols
```
`pixels_so_far` is a running total. `num_pixels` (N) is the total number of pixels.

```
for input = 0 to 255
    pixels_so_far = pixels_so_far + h[input]
```
This goes through the grey levels from darkest to brightest and keeps adding up the histogram. After level `input`, `pixels_so_far` = **the number of pixels with value ≤ input**. That's the **cumulative histogram**. `pixels_so_far / num_pixels` turns the count into a **fraction** between 0 and 1. That's the **CDF**, which is exactly the rank described above.

```
    LUT[input] = (pixels_so_far * 256) / (num_pixels + 1)
```
This scales that fraction up to the 0–255 range and stores it in a **lookup table (LUT)**. `LUT[input]` answers "what should grey level `input` become?" It's an ordinary array of 256 numbers, and the division is integer division, so it rounds down.

```
for every pixel f(i,j)
    g(i,j) = LUT[ f(i,j) ]
```
Each pixel's old value is replaced with the value from the table. The table is computed **once** (256 entries), so this step is fast, just one array lookup per pixel.

**Worked example.** A **low-contrast** image of 16 pixels that only uses levels 50–53, with 4 pixels at each level. N = 16, so N + 1 = 17.

| input | h[input] | pixels_so_far | fraction | LUT = ⌊pixels_so_far × 256 / 17⌋ |
|---|---|---|---|---|
| 0–49 | 0 | 0 | 0 | 0 |
| **50** | 4 | 4 | 25% | ⌊1024 / 17⌋ = **60** |
| **51** | 4 | 8 | 50% | ⌊2048 / 17⌋ = **120** |
| **52** | 4 | 12 | 75% | ⌊3072 / 17⌋ = **180** |
| **53** | 4 | 16 | 100% | ⌊4096 / 17⌋ = **240** |
| 54–255 | 0 | 16 | 100% | 240 |

| Before | 50 | 51 | 52 | 53 |
|---|---|---|---|---|
| **After** | 60 | 120 | 180 | 240 |

The levels were squashed into 4 values (almost invisible differences). Now they're spread across nearly the whole range, which is much higher contrast. The pixels with value 51 were at the 50% mark, so they became ~128 (120 here, because of the rounding with such a small image). This example also shows the **gaps**: only 4 output levels are used, with empty levels between them, because all pixels with the same input always go to the same output.

**Why `+1`?** Look at the **last** level, where `pixels_so_far = N` (every pixel has been counted).
- **Without +1:** `N × 256 / N = 256`. That's **invalid**, because an 8-bit pixel can only go up to 255.
- **With +1:** `N × 256 / (N + 1)` is slightly **less** than 256, so it rounds down to **255** at most (in the example, 240).

So the `+1` makes the division always a bit smaller than it would be, which keeps the result in range. With a real image (N in the hundreds of thousands), the effect on other levels is tiny.

> **中文解释：** 均衡化的核心是**按"排名"重新分配亮度**：一个像素比图中百分之多少的像素亮（或一样亮），它的新灰度就是 255 的百分之多少。`pixels_so_far` 是**累计直方图**（≤ 当前灰度的像素个数），除以总像素数就是 **CDF**（0 到 1 之间的比例）。乘以 256 得到新灰度，存进**查找表 LUT**，最后每个像素查表替换即可。例子：只有 50–53 四个灰度的低对比度图，均衡化后变成 60、120、180、240，对比度大大提高。分母加 1 是为了让最后一级的结果 < 256（最大 255），避免越界。

### Why the result has gaps and peaks
- Equalisation **moves whole bins**. It **never splits one bin** across several output levels, because that would mean arbitrarily assigning identical pixels to different levels.
- So the output histogram is **not truly flat**. It has **gaps (empty levels) and peaks** (the slide 14 example spreads 6 equal bins out with gaps between them).

**My own worked example** (8 grey levels 0–7, 16 pixels, formula `LUT = cum·8/(N+1)`):

| Input level | 0 | 1 | 2 | 3 | 4–7 |
|---|---|---|---|---|---|
| h | 4 | 4 | 4 | 4 | 0 |
| cumulative | 4 | 8 | 12 | 16 | 16 |
| LUT = ⌊cum·8/17⌋ | 1 | 3 | 5 | 7 | 7 |

Levels 0,1,2,3 map to 1,3,5,7. The full range is now used, but levels 0, 2, 4 and 6 are **empty gaps**.

> **中文解释：** 均衡化后的直方图会出现"梳子状"的**空隙和尖峰**，因为同一个灰度的所有像素只能整体映射到同一个新灰度，不能拆开。

### Possible improvement (mentioned in the lecture)
Human discrimination is best for **mid-grey levels** and worse for very dark or very bright ones, so you could target a **bell-shaped (normal) distribution** instead of a flat one.

### OpenCV (C++)
```cpp
vector<Mat> channels(hls_image.channels());
split(hls_image, channels);
equalizeHist(channels[1], channels[1]);   // channel 1 = L in HLS
merge(channels, hls_image);
```
Python:
```python
hls = cv2.cvtColor(img, cv2.COLOR_BGR2HLS)
h, l, s = cv2.split(hls)
l = cv2.equalizeHist(l)
out = cv2.cvtColor(cv2.merge([h, l, s]), cv2.COLOR_HLS2BGR)
```

---

## 5. Histogram Comparison

### When is it useful?
- **For finding similar whole images:** not very useful. **Metadata/tags** are the safer approach (this is what search engines do).
- **Colour histograms are not unique:** blue could be sea or sky.
- In the lecture demo (about 20 images), a museum image matched itself at 1.0, a similar building scored about 0.78, but completely different scenes still scored about **0.70**. The scores **do not separate** similar images well.
- **For comparing regions:** very useful, for example tracking **the same object or person** (a red car, someone in jeans and a white T-shirt) across frames.

> **中文解释：** 用直方图比较**整张图**效果不好（蓝色可能是天空也可能是大海，完全不同的图也能得到 0.7 的分数）；但用来比较**分割出的物体**（同一个人、同一辆车）非常有效。

### Metrics (N = number of bins, $\bar{h}_k = \frac{1}{N}\sum_i h_k(i)$)

| Metric | Formula | Perfect match | OpenCV flag |
|---|---|---|---|
| **Correlation** | $\dfrac{\sum_i (h_1(i)-\bar h_1)(h_2(i)-\bar h_2)}{\sqrt{\sum_i (h_1(i)-\bar h_1)^2 \sum_i (h_2(i)-\bar h_2)^2}}$ | **1** (range −1…1) | `CV_COMP_CORREL` / `HISTCMP_CORREL` |
| **Chi-Square** | $\sum_i \dfrac{(h_1(i)-h_2(i))^2}{h_1(i)+h_2(i)}$ | **0** (higher = worse) | `CV_COMP_CHISQR` / `HISTCMP_CHISQR` |
| **Intersection** | $\sum_i \min(h_1(i), h_2(i))$ | Highest value (1 for histograms normalised to sum to 1) | `CV_COMP_INTERSECT` / `HISTCMP_INTERSECT` |
| **Bhattacharyya** | $\sqrt{1 - \dfrac{1}{\sqrt{\bar h_1 \bar h_2 N^2}} \sum_i \sqrt{h_1(i)\,h_2(i)}}$ | **0** (range 0…1) | `CV_COMP_BHATTACHARYYA` / `HISTCMP_BHATTACHARYYA` |

- **Bhattacharyya** is widely used, notably in **mean shift** tracking (covered later in the course).
- **Weakness of bin-by-bin metrics:** if you shift a histogram by one grey level (add 1 to every pixel), the two histograms **look almost identical but can score very badly**, especially if they are noisy. Bins are only compared with the **same** bin. This motivates the Earth Mover's Distance.

> **中文解释：** 要记住每个指标"完全匹配"时的值：**相关性 = 1，卡方 = 0，交集 = 最大，巴氏距离 = 0**。这些指标都是"逐格比较"，所以直方图整体平移一格，结果就会变得很差，这也是引入 EMD 的原因。

### Earth Mover's Distance (EMD)
- **Intuition:** the **minimum cost of turning one distribution into the other**, like moving piles of earth on a building site. Cost = amount moved × distance moved.
- It compares **across neighbouring bins**, so a small shift gives a **small** distance.

**1D solution:**
$$EMD(-1) = 0$$
$$EMD(i) = h_1(i) + EMD(i-1) - h_2(i)$$
$$\text{Earth Mover's Distance} = \sum_i |EMD(i)|$$

`EMD(i)` is the running surplus or deficit: the amount of "earth" that must be carried past bin i to the next bin.

**My own small worked example:** $h_1 = [0, 3, 1, 0]$, $h_2 = [0, 1, 2, 1]$

| i | h₁ | h₂ | EMD(i) = h₁ + EMD(i−1) − h₂ |
|---|---|---|---|
| 0 | 0 | 0 | 0 + 0 − 0 = **0** |
| 1 | 3 | 1 | 3 + 0 − 1 = **2** |
| 2 | 1 | 2 | 1 + 2 − 2 = **1** |
| 3 | 0 | 1 | 0 + 1 − 1 = **0** |

Distance = |0| + |2| + |1| + |0| = **3**. Check: move 1 unit from bin 1 to bin 2 (cost 1) and 1 unit from bin 1 to bin 3 (cost 2), giving a total of 3. ✓

(In the lecture's slide 20 example, the two histograms differ only slightly and the total is **21**, made up of about 12 + 5 + 4 from the three regions that differ.)

- **Colour (3D) EMD is much harder to compute.** There is no simple running-sum solution.

> **中文解释：** EMD（推土机距离）= 把一个直方图"搬运"成另一个直方图所需的最小代价（搬运量 × 搬运距离）。一维时只需做**累计差值**再取绝对值求和。它的优点是：直方图只平移一点点，距离也只增加一点点。

### OpenCV (C++)
```cpp
normalize(histogram1, histogram1, 1.0);
normalize(histogram2, histogram2, 1.0);
double matching_score = compareHist(histogram1, histogram2, CV_COMP_CORREL);
// alternatives: CV_COMP_CHISQR, CV_COMP_INTERSECT, CV_COMP_BHATTACHARYYA, or EMD()
```
**Always normalise the histograms before comparing** them, so that images or regions of different sizes are comparable.
(Technical note: `normalize(h, h, 1.0)` uses the L2 norm by default. Use `NORM_L1` for sum = 1 or `NORM_INF` for max = 1.)

### 中文详解：直方图比较与 EMD

#### 1. 为什么要比较直方图？
目的是判断**两张图（或两个区域）的颜色分布是否相似**。
- **比较整张图：效果不好。** 直方图不唯一：蓝色可能是**天空**，也可能是**大海**。课上的演示：同一张图自己和自己比 = 1.0，相似的建筑 ≈ 0.78，但**完全不同的场景也有 ≈ 0.70**。分数拉不开差距，所以区分不了。找相似图片时，更可靠的是用**元数据/标签**。
- **比较区域/物体：非常有用。** 例如在视频的不同帧里跟踪**同一个人**（牛仔裤 + 白 T 恤）或**同一辆红色汽车**。

**比较之前一定要先归一化（normalise）**：大区域像素多、小区域像素少，不归一化的话，数量不同就没法比。

#### 2. 四种比较指标（metrics）
这四种都是**逐格比较（bin-by-bin）**：只拿第 i 格和第 i 格比。

| 指标 | 直观含义 | 完全相同时 | 越相似 |
|---|---|---|---|
| **相关性 Correlation** | 两个直方图的形状是否"一起升、一起降" | **1** | 越接近 1 |
| **卡方 Chi-Square** | 每一格的差值平方，除以该格总量，再求和 | **0** | 越小 |
| **交集 Intersection** | 每一格取两者中**较小**的值再求和，也就是两个直方图"重叠"的部分 | **最大**（归一化后 = 1） | 越大 |
| **巴氏距离 Bhattacharyya** | 衡量两个分布的重叠程度，0 = 完全相同，1 = 完全不重叠 | **0** | 越小 |

**考试一定要记住的"完全匹配值"：相关 = 1，卡方 = 0，交集 = 最大，巴氏 = 0。** 巴氏距离用得很广，后面课程的 **mean shift 跟踪**会用到它。

#### 3. 逐格比较的致命缺点
假设把一张图的**每个像素都加 1**（整体稍微变亮一点点）：
- 看起来两张图几乎一模一样；
- 但直方图**整体平移了一格**，第 i 格的内容全部跑到了第 i+1 格；
- 逐格比较时，每一格都对不上，**分数会很差**（直方图有噪声时更明显）。

也就是说，逐格比较**不知道"相邻的格子其实很接近"**。这就是引入 EMD 的原因。

#### 4. 推土机距离（Earth Mover's Distance, EMD）

**4.1 直观理解。** 把两个直方图想象成两种**土堆的摆放方式**：直方图 1 = 现在的土堆，直方图 2 = 想要的土堆形状，每一格的高度 = 那里有多少土。

**EMD = 把土堆 1 搬成土堆 2 所需的最小工作量；工作量 = 搬运的土量 × 搬运的距离。**
- 土只需要搬到**隔壁格** → 距离小 → EMD 小；
- 土需要搬到**很远的格** → 距离大 → EMD 大。

这正好解决了逐格比较的问题：**直方图只平移一点，EMD 也只增加一点。**（前提：两个直方图的土的总量要一样，所以要先归一化。）

**4.2 一维计算公式**

$$EMD(-1) = 0$$
$$EMD(i) = h_1(i) + EMD(i-1) - h_2(i)$$
$$\text{距离} = \sum_i |EMD(i)|$$

**EMD(i) 的含义**：处理完第 i 格后，**还需要从第 i 格往第 i+1 格搬多少土**。
- 正数：第 i 格及左边的土**多了**，要往右搬；
- 负数：第 i 格及左边的土**不够**，要从右边往左搬；
- 每跨过一个格子边界，距离是 1，所以把每个 |EMD(i)| 加起来就是总工作量。

计算步骤：**从左往右走，"当前格的土 + 上一格传过来的土 − 这一格需要的土"，传给下一格**。

**4.3 例子 1：为什么 EMD 比逐格比较好**

$$h_1 = [0, 4, 0, 0],\quad h_2 = [0, 0, 4, 0],\quad h_3 = [0, 0, 0, 4]$$

h₂ 是 h₁ 右移 **1 格**，h₃ 是 h₁ 右移 **2 格**。

**逐格比较**：h₁ 和 h₂ 没有任何一格重叠，h₁ 和 h₃ 也没有，两者的**交集都是 0**，得到"完全不像"的结论，**分不出哪个更接近**。

**EMD（h₁ → h₂）**：

| i | h₁ | h₂ | EMD(i) = h₁ + EMD(i−1) − h₂ |
|---|---|---|---|
| 0 | 0 | 0 | 0 + 0 − 0 = **0** |
| 1 | 4 | 0 | 4 + 0 − 0 = **4** |
| 2 | 0 | 4 | 0 + 4 − 4 = **0** |
| 3 | 0 | 0 | 0 + 0 − 0 = **0** |

距离 = 0 + 4 + 0 + 0 = **4**（4 份土各搬 1 格）

**EMD（h₁ → h₃）**：

| i | h₁ | h₃ | EMD(i) |
|---|---|---|---|
| 0 | 0 | 0 | **0** |
| 1 | 4 | 0 | 4 + 0 − 0 = **4** |
| 2 | 0 | 0 | 0 + 4 − 0 = **4** |
| 3 | 0 | 4 | 0 + 4 − 4 = **0** |

距离 = 0 + 4 + 4 + 0 = **8**（4 份土各搬 2 格）

**结论**：EMD 知道平移 1 格（4）比平移 2 格（8）更接近，而逐格比较认为两者一样差。

**4.4 例子 2：上面的例子**

$$h_1 = [0, 3, 1, 0],\quad h_2 = [0, 1, 2, 1]$$

| i | h₁ | h₂ | EMD(i) | 含义 |
|---|---|---|---|---|
| 0 | 0 | 0 | **0** | 不用搬 |
| 1 | 3 | 1 | 3 + 0 − 1 = **2** | 第 1 格多了 2 份，要往右搬 |
| 2 | 1 | 2 | 1 + 2 − 2 = **1** | 收到 2 份，加上自己 1 份，留 2 份，还多 1 份，继续往右 |
| 3 | 0 | 1 | 0 + 1 − 1 = **0** | 收到 1 份，正好 |

距离 = 0 + 2 + 1 + 0 = **3**。验证：1 份土从第 1 格 → 第 2 格（代价 1），1 份土从第 1 格 → 第 3 格（代价 2），共 **3** ✓

**4.5 彩色 EMD。** 一维时只需从左往右做**累计差值**，非常简单；**三维（彩色）直方图没有这种简单的"从左往右"解法**，土可以往很多方向搬，要解一个优化问题，**计算难得多**。

#### 5. 考试要点
1. 比较**整张图**效果差（蓝色 = 天空还是大海？），比较**区域/物体**效果好（跟踪）。
2. 比较前**先归一化**。
3. 四个指标的**完全匹配值**：相关 1，卡方 0，交集最大，巴氏 0。
4. 逐格比较的缺点：直方图**平移一格**，分数就很差。
5. EMD = 最小**搬运代价**（土量 × 距离），能处理平移。
6. 会**手算一维 EMD**（从左往右累计，取绝对值求和）。
7. 彩色 EMD 很难算。

---

## 6. Histogram Back Projection

### Purpose
**Select pixels of a particular colour, based on samples.** Examples: all skin pixels, or all red pixels.

### Steps (learn these five)
1. **Obtain a representative sample set** of the colours (for example, patches of skin from many images).
2. **Histogram those samples**, usually as a 3D, quantised histogram.
3. **Normalise the histogram so that its maximum value is 1.0.**
4. **Back project** the normalised histogram onto any image f(i,j). For each pixel, look up the histogram cell for its colour and use that cell's value as the output.
5. The result is a **"probability" image p(i,j)** showing how similar each pixel of f(i,j) is to the sample set.

```
for every pixel (i,j):
    p(i,j) = normalised_hist[ bin(f(i,j)) ]
```

> **中文解释：** 反向投影：先用样本（比如肤色块）建立颜色直方图并归一化（最大值 = 1），然后对新图像的每个像素，查它的颜色落在直方图哪个格子，把那个格子的值作为输出。结果是一张"**概率图**"，越亮表示越像样本颜色（越可能是皮肤）。

### Practical issues
- The **sample set must be large enough.** The lecture example is too small for skin detection in general.
- The histogram distribution should be **continuous, without gaps**. Sparse samples leave holes, so you may need **smoothing** or **coarser quantisation**.
- The output is **not binary**. You usually need to **threshold** it, and it is typically one step in a larger pipeline.

### OpenCV (C++, from slides)
```cpp
calcHist(&hls_samples_image, 1, channel_numbers, Mat(),
         histogram, image.channels(), number_bins, channel_ranges);
normalize(histogram, histogram, 1.0);
Mat probabilities = histogram.BackProject(hls_image);   // wrapper used in the book's code
```
In standard OpenCV (Python), the function is `calcBackProject`:
```python
hist = cv2.calcHist([hls_samples], [0, 1, 2], None, [16,16,16], [0,256,0,256,0,256])
cv2.normalize(hist, hist, 0, 255, cv2.NORM_MINMAX)
prob = cv2.calcBackProject([hls_img], [0, 1, 2], hist, [0,256,0,256,0,256], 1)
_, mask = cv2.threshold(prob, 50, 255, cv2.THRESH_BINARY)
```

### 中文详解：直方图反向投影

#### 1. 它是用来干什么的？
**根据样本，找出图像中某种特定颜色的像素。** 例如：找出所有**皮肤**像素（人脸检测、手势识别）、所有**红色**像素，或某个**特定物体**（比如一件衣服）的颜色。

和前面几节的区别：
- 前面的直方图是"**从图像 → 统计出直方图**"；
- 反向投影反过来：拿一个**已有的直方图**，"**投回**"到一张新图像上，看每个像素像不像这个直方图代表的颜色。这就是"**反向**"的意思。

#### 2. 五个步骤（一定要背）
1. **获取有代表性的样本**：比如从很多张图里剪下皮肤小块。
2. **对样本计算直方图**：通常是三维、降低量化的直方图（比如 HLS，每通道 16 个 bin）。
3. **归一化，使直方图的最大值 = 1.0**。
4. **把归一化直方图"反向投影"到任意图像 f(i,j) 上**：对每个像素，看它的颜色落在直方图的哪个 bin，就把那个 bin 的值作为输出。
5. **得到一张"概率图" p(i,j)**：每个像素的值（0 到 1）表示它和样本颜色**有多像**。

核心伪代码：
```
对每个像素 (i,j)：
    p(i,j) = 归一化直方图[ 像素 f(i,j) 所在的 bin ]
```
也就是**查表**：像素颜色 → 找到 bin → 取出那个 bin 的值。

#### 3. 手算例子（为了简单，只看色调 H，4 个 bin）

**第 1–2 步：样本直方图。** 从皮肤样本中取了 100 个像素，统计结果：

| bin | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| 样本像素数 | 10 | **60** | 30 | 0 |

意思是：皮肤的颜色**大多数落在 bin 1**，一部分在 bin 2，很少在 bin 0，bin 3 完全没有。

**第 3 步：归一化，最大值 = 1。** 每个值都除以最大值 60：

| bin | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| 归一化值 | 10/60 ≈ **0.17** | 60/60 = **1.0** | 30/60 = **0.5** | 0/60 = **0** |

这就是一张**查找表**：颜色落在哪个 bin，就能查到它有多像皮肤。bin 1 → 最像（1.0）；bin 3 → 完全不像（0）。

**第 4 步：投影到新图像。**

*步骤 A：每个像素的色调值 → 它属于哪个 bin。* 一张新图像，3×3 = 9 个像素，每个像素都有一个**色调值 H**（OpenCV 里范围是 0–179）：

```
每个像素的色调值 H：
   60   70  150
  100   80  160
   20  170  140
```

把 0–179 分成 4 个 bin，每个 bin 宽 180 / 4 = 45：

| bin | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| 色调范围 | 0–44 | 45–89 | 90–134 | 135–179 |

逐个像素判断它落在哪个 bin：

| 像素位置 | 色调值 | 落在哪个范围 | bin |
|---|---|---|---|
| 左上 | 60 | 45–89 | **1** |
| 上中 | 70 | 45–89 | **1** |
| 右上 | 150 | 135–179 | **3** |
| 左中 | 100 | 90–134 | **2** |
| 正中 | 80 | 45–89 | **1** |
| 右中 | 160 | 135–179 | **3** |
| 左下 | 20 | 0–44 | **0** |
| 下中 | 170 | 135–179 | **3** |
| 右下 | 140 | 135–179 | **3** |

放回原来的位置：

```
每个像素所在的 bin：
  1  1  3
  2  1  3
  0  3  3
```

*步骤 B：每个像素的 bin → 查表得到"像不像皮肤"。*

| 像素位置 | bin | 查表：bin 的值 |
|---|---|---|
| 左上 | 1 | **1.0** |
| 上中 | 1 | **1.0** |
| 右上 | 3 | **0** |
| 左中 | 2 | **0.5** |
| 正中 | 1 | **1.0** |
| 右中 | 3 | **0** |
| 左下 | 0 | **0.17** |
| 下中 | 3 | **0** |
| 右下 | 3 | **0** |

放回原来的位置，就是**概率图**：

```
概率图 p(i,j)：
  1.0   1.0   0
  0.5   1.0   0
  0.17  0     0
```

*一个像素走一遍整个流程：*

```
左上：色调值 60  →  落在 45–89，属于 bin 1   →  查表 bin 1 = 1.0  →  输出 1.0（很像皮肤）
右上：色调值 150 →  落在 135–179，属于 bin 3 →  查表 bin 3 = 0    →  输出 0（不是皮肤）
```

**每个像素都做同样的事：色调值 → 找 bin → 查表 → 输出。** 这就是"反向投影"的全部内容。

**第 5 步：解读结果。**
- 值为 **1.0** 的像素 → **很可能是皮肤**
- 值为 **0** 的像素 → **不是皮肤**
- 0.5、0.17 → **有点像**，不确定

把概率图显示成灰度图：**越亮 = 越像皮肤，越暗 = 越不像**。

#### 4. 为什么要归一化到"最大值 = 1"？
- 原始直方图是**像素个数**（10、60、30……），数值大小取决于样本有多少，没有统一标准。
- 除以最大值后，所有值都在 **0 到 1** 之间，**最像样本的颜色 = 1**，方便解读，也方便后面设阈值。

注意：这里是"**最大值 = 1**"，不是"**总和 = 1**"。所以结果只是"**像不像**"的相对程度，不是严格数学意义上的概率。这就是为什么课件里的"probability"要加引号。

#### 5. 为什么常用 HLS 而不是 RGB？
- 在 RGB 中，同一块皮肤**在亮处和暗处**，R、G、B 三个值**全都会变**；
- 在 HLS 中，**亮度变化主要体现在 L 通道**，色调 H 和饱和度 S 相对稳定；
- 所以用 HLS 的颜色直方图，在不同光照下更容易认出"同一种颜色"。

#### 6. 实际使用中的问题
**① 样本要足够多。** 如果样本太少，直方图只覆盖了**一小部分**皮肤颜色；其他人、其他光照下的皮肤颜色可能落在**空的 bin**（值 = 0），被误判为"不是皮肤"。课上的例子样本就**太少**，不能用于一般的皮肤检测。

**② 直方图要连续、没有空洞。** 样本稀疏时，直方图会出现"**空洞**"：某个 bin 是 0，但它两边的 bin 都很高。实际上那个颜色很可能也是皮肤，只是样本里恰好没有。解决方法：
- **平滑直方图**（和第 1.1 节的平滑一样，取相邻 bin 的平均）；
- **降低量化**（用更少的 bin，每个 bin 更宽，空洞就少了）。

**③ 输出不是二值的。** 概率图是 0 到 1 之间的连续值，**不是"是/不是"**。通常还需要**设一个阈值**（比如 > 0.5 算皮肤），才能得到二值的掩码。反向投影一般只是**大系统中的一步**。

#### 7. OpenCV 代码（C++，课件）
```cpp
calcHist(&hls_samples_image, 1, channel_numbers, Mat(),
         histogram, image.channels(), number_bins, channel_ranges);  // 第 2 步：样本直方图
normalize(histogram, histogram, 1.0);                               // 第 3 步：归一化
Mat probabilities = histogram.BackProject(hls_image);                // 第 4–5 步：反向投影
```

#### 8. 考试要点
1. **目的**：根据样本选出特定颜色的像素（例如皮肤）。
2. **五个步骤**：取样本 → 算直方图 → 归一化（最大值 = 1）→ 反向投影 → 得到概率图。
3. **核心操作**：每个像素查它所在 bin 的值，本质就是**查表**。
4. **输出**：一张"概率图"，越亮越像样本颜色；**不是二值的**，需要**阈值**。
5. **实际问题**：样本要够多；直方图要连续无空洞（**平滑**或**降低量化**）；通常只是大系统中的一步。

---

## 7. Summary

| Technique | Input → Output | Main use | Who benefits |
|---|---|---|---|
| 1D histogram | image → counts per grey level | Summarise, find thresholds | Computer |
| Smoothing | histogram → smoother histogram | Reduce noise (watch the **boundaries**) | Computer |
| Colour histogram (per channel) | image → 3 × 1D histograms | Colour analysis (**HLS hue** is most useful) | Computer |
| 3D histogram | image → (c1,c2,c3) counts | Better discrimination; needs **reduced quantisation** | Computer |
| Equalisation | image → image | Contrast enhancement (**luminance only**) | **Humans** |
| Comparison | 2 histograms → score | Match **objects or regions** (not whole images) | Computer |
| Back projection | sample histogram + image → probability image | **Select colours** (for example skin) | Computer |

### Exam tips
- **Histograms are global and not unique.** Spatial information is lost.
- **Cell count** = (bins per channel)^channels. Be able to compute 16,777,216 / 262,144 / 4,096 / 64.
- **Equalisation:** it is the **cumulative** histogram → LUT. Know why there are **gaps**, why you **only equalise luminance**, and why there is a **+1**.
- **Metric perfect-match values:** Correlation 1, Chi-square 0, Intersection max, Bhattacharyya 0.
- **Bin-by-bin metrics fail on shifted histograms**, and **EMD** fixes this. Be able to compute 1D EMD by hand.
- **Back projection:** know the 5 steps. The output is a **probability image**, not binary, so you threshold it.
- **Smoothing boundaries:** do not compute / wrap / duplicate / reflect / constant.
- **Colour spaces:** CMY = the inverse of RGB, so no new information. **Hue is circular** (red at both ends; OpenCV H = 0–179), and hue is meaningless for near-black, near-white or unsaturated pixels.

---

## 8. Practice Questions

1. Why is a histogram "not unique"? Give an example.
2. An RGB image uses 5 bits per channel in a 3D histogram. How many cells are there?
3. Why shouldn't you equalise each RGB channel separately? What should you do instead?
4. Why does an equalised histogram contain gaps?
5. Compute the 1D EMD between $h_1 = [2, 2, 0, 0]$ and $h_2 = [0, 2, 2, 0]$.
6. Why might correlation give a poor score for two histograms that look almost identical?
7. List the five steps of histogram back projection. What does the output represent?
8. Name three ways of handling the boundaries when smoothing a histogram.

<details>
<summary><b>Answers</b></summary>

1. Only the counts are stored, not the positions. Randomly shuffling the pixels (or, say, a blue sky versus a blue sea) gives the same histogram.
2. 2⁵ = 32 bins per channel, so 32³ = **32,768** cells.
3. It **distorts the colours**, because the channels change independently. Convert to HLS (or similar), equalise **L only**, then convert back.
4. All pixels in one input bin map to the **same** output level. Bins are moved, never split, so some output levels stay empty.
5. EMD: i0: 2−0 = 2; i1: 2+2−2 = 2; i2: 0+2−2 = 0; i3: 0. Sum = **4**. (Move 2 units from bin 0 to bin 2 = 2 × 2 = 4 ✓)
6. Bin-by-bin metrics only compare the **same bin**. A shift of one grey level misaligns every bin, especially in noisy histograms. EMD handles this.
7. Sample the colours → histogram them → normalise to max 1.0 → back project onto the image → probability image p(i,j), showing each pixel's similarity to the sample colours.
8. Any three of: do not compute, wraparound, duplicate, reflect, constant.
</details>

---

*Note: parts of the first transcript were garbled (repeated lines in the RGB/CMY and HLS sections). Section 2 was completed using the slides and the* Colour Images *lecture transcript in the Images topic, which uses the same snooker-table example.*

*Related:* 1D histograms are also used to choose thresholds automatically (Otsu's method). See the Binary Vision notes.
