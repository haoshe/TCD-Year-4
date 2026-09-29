# Computer Vision — Chapter: Images
*Notes compiled from lecture slides + transcripts (Camera Models & Digital Images, Colour Images, Noise & Smoothing)*

---

## 1. Camera Models

### 1.1 Physical components of a camera
A camera = three physical parts:
- **Photosensitive image plane** — historically film, now a 2D array of photosensitive sensor elements (CCD/CMOS).
- **Housing** — the light-tight box/body (can be huge, like an old studio camera, or tiny, like a phone camera).
- **Lens** — focuses light from the 3D world onto the image plane.

> **Why this matters:** the lens introduces **distortion** (and can cause things to be out of focus if rays don't converge exactly on the sensor plane). To do maths on images, we need a simplified mathematical model of "camera" first, then add distortion correction on top. That correction is covered later in the course under geometric transformations — for now, just know it exists and is handled separately.

### 1.2 The simple pinhole camera model
Historically, before lenses, cameras were literally a box with a tiny hole (pinhole) in it. Light rays from the scene pass through this single point and land upside-down on the back of the box. This is the **conceptual basis** for the camera model used in computer vision — even though real cameras use lenses, once lens distortion is corrected for, the maths reduces to the pinhole model.

**The model** relates a 3D world point to a 2D image point:

$$
\begin{bmatrix} i \cdot w \\ j \cdot w \\ w \end{bmatrix}
=
\begin{bmatrix} f_i & 0 & c_i \\ 0 & f_j & c_j \\ 0 & 0 & 1 \end{bmatrix}
\begin{bmatrix} x \\ y \\ z \end{bmatrix}
$$

Term by term:
- **(x, y, z)** — a 3D point in the real world (Euclidean space). The lecturer deliberately avoids calling these "X, Y" when talking about the *image*, to stop you confusing them with image coordinates.
- **(i, j)** — the corresponding 2D point on the image plane (equivalently "row, column"). This is the pixel location.
- **w** — a scaling factor, needed to make the linear-algebra formulation work (it lets a 3D→2D perspective projection be written as a matrix multiply). You divide it out at the end to recover actual (i, j).
- **fᵢ, fⱼ** — combine the camera's **focal length** with the pixel scaling of the sensor (i.e., "how many pixels per mm", separately in each axis, since pixels aren't always perfectly square).
- **cᵢ, cⱼ** — the **optical centre**: where the camera's optical axis actually intersects the image plane (rarely exactly the geometric centre of the sensor).

**Simplified form** (once distortion is removed — this is what OpenCV effectively works with):
$$ i = f_i \cdot \frac{x}{z} + c_i \qquad j = f_j \cdot \frac{y}{z} + c_j $$

> **Intuition:** this is just "perspective divide" — things further away (larger z) appear smaller/closer to the centre, because you divide x and y by z.

### 1.3 Practical uses of the pinhole model

**(a) Estimating real-world distance to an object of known size**
If you know an object's real width $W_{mm}$ and can measure its width in pixels $W_{pixels}$ in the image, you can compute how far away it is:
$$ D_{mm} = D_{camera} \cdot \frac{W_{mm}}{W_{pixels}} $$
where $D_{camera}$ comes from Pythagoras using the focal length and pixel offset from the optical centre.

**(b) Estimating distance travelled between two frames**
If an object's apparent size changes between two frames (i.e. you compute two distances $D_1$ and $D_2$ using the method above, plus the angle θ between the two viewing directions), you can find the distance moved using the **law of cosines**. This is a classic building block for tasks like visual odometry (estimating camera/robot motion from images).

---

## 2. Digital Images

### 2.1 Images are theoretically continuous
Physically/theoretically, an image is a continuous 2D function of *reflected scene brightness* — for every real-valued position (i, j), there's a brightness value. A computer can't store a continuous function, so we need a **discrete representation**. This requires two separate steps:
1. **Sampling** — deciding *where* to measure (turning a continuous plane into a discrete grid of points).
2. **Quantisation** — deciding *what values* those measurements can take (turning continuous brightness into a limited set of discrete numbers).

These are independent problems — you could sample finely but quantise coarsely, or vice versa.

### 2.2 Sampling
The **sensor** is a 2D array of photosensitive elements. Two physical realities cause problems:
- **Non-photosensitive gaps** between elements — in theory, something in the real world could fall entirely in a gap and simply not be captured.
- **Each element has a non-zero area**, not a single point. A pixel on the boundary between object and background ends up averaging light from *both*, producing a "blended" colour/grey value that belongs properly to neither — this is a source of edge artefacts you'll see again later in the course.

**How many samples (pixels) do we need?**
This is entirely task-dependent:
- Too many samples → wasted storage and wasted computation, without added benefit.
- Too few samples → you literally lose the ability to see/detect what you care about.
- Example from the lecture: to estimate what *percentage* of an image is fruit (a colour-based question), quite low resolution suffices. To recognise individual fruit types, you need much higher resolution.

Practical rule of thumb: **use the resolution the task actually needs**, not the maximum available.

OpenCV: `resize(image, smaller_image, Size(image1.cols/2, image.rows/2));`

### 2.3 Quantisation
Once you have discrete sample points, each point's brightness must be represented as a **discrete numeric value**.
- Typically **8 bits per pixel** for greyscale (0–255). This is a legacy/practical choice — a byte has always been 8 bits, and processing "less than a byte" (individual bits) is actually *harder* for a computer, not easier — so there's no real efficiency win to using fewer bits, even though it saves storage space.
- Nothing stops you using more (16, 24 bits) — some scientific/medical imaging does this — or fewer.

**Effect of reducing bit depth:** as you drop from 8 → 6 → 4 → 2 bits, you introduce **false/artificial contours** — visible "banding" — especially in regions where brightness changes slowly and smoothly (e.g. a clear sky). This happens *regardless* of how many bits you use in total; it's just more visible at low bit depth. Humans might not consciously notice it at 8→6 bits, but a computer processing the raw numbers "notices" a hard discontinuity where there wasn't one in the real world.

**Trade-off:** fewer grey levels/colours = less storage & often *easier* downstream processing (fewer distinct values to reason about), but at the cost of losing the ability to distinguish similar objects and introducing these artificial contours.

OpenCV-style masking to reduce bit depth (keep only the top `num_bits` most-significant bits):
```cpp
void ChangeQuantisationGrey( Mat &image, int num_bits )
{
  CV_Assert( (image.type() == CV_8UC1) && (num_bits >= 1) && (num_bits <= 8) );
  uchar mask = 0xFF << (8-num_bits);
  for (int row=0; row < image.rows; row++)
    for (int col=0; col < image.cols; col++)
      image.at<uchar>(row,col) = image.at<uchar>(row,col) & mask;
}
```
*(This just zeroes out the least-significant bits — a bitmask trick, not a "real" requantisation, but computationally trivial.)*

---

## 3. Colour Images

### 3.1 Why colour, and why it was avoided historically
Greyscale-only images were the norm for decades, mostly because processing was *computationally* expensive (the lecturer mentions single images once taking ~1–1.5 hours to process!). Colour triples the amount of data and complexity, so it was avoided when possible.

Colour images typically have **3 channels** combining:
- **Luminance** — brightness information.
- **Chrominance** — colour information.

With 8 bits/channel × 3 channels → **~16.8 million possible colours** (256³), vs. 256 grey levels for greyscale. More information, but much more to process — though colour often makes certain tasks *easier* (e.g. segmenting a blue sky, or telling a red coat apart from surroundings) precisely because it adds discriminative information that greyscale collapses away.

### 3.2 RGB — Red, Green, Blue
The most common representation, closely tied to how camera sensors physically work: sensor elements are photosensitive to overlapping ranges of wavelengths centred roughly around:
- Red ≈ 700 nm
- Green ≈ 546 nm
- Blue ≈ 436 nm

Each is described by a **spectral sensitivity curve**, not a single wavelength — real sensors respond to a *range* of wavelengths, weighted differently. Note: the green sensitivity curve is broad, covering most of the visible spectrum — which is why a green channel alone often looks close to a greyscale image.

**Converting RGB → Greyscale:** a weighted sum, not a simple average:
$$ Y = 0.299R + 0.587G + 0.114B $$
Green dominates the weighting because human (and sensor) sensitivity to green is highest. Note this exact formula isn't universal — different cameras/sensors could justify different weights — but this is the standard one used.

**The Bayer pattern:** most colour cameras don't measure full RGB at every physical sensor site. Instead each site (each square in the slide's diagram) has a colour filter over it and senses **only one colour**. The filters are laid out in a mosaic built from one repeating 2×2 block:

```
R G      ← rows like this: red, green, red, green, ...
G B      ← rows like this: green, blue, green, blue, ...
```

- **Green** is on **half** of all sites, in a checkerboard: every 2nd site along *every* row and column.
- **Red** is on **a quarter** of all sites: every 2nd site, but only in every *other* row (the R-G rows).
- **Blue** is on **a quarter** of all sites: every 2nd site, but only in the *other* rows (the G-B rows).

So each 2×2 block holds **2 green : 1 red : 1 blue**. When the lecturer says "every fourth pixel is red", that means **1 in 4 of all sites** is red. It does *not* mean red is spaced 4 apart along a row. Along its own row, red is every 2nd site, just like green.

> **Why extra green?** Human vision is most sensitive to green, and green carries most of the brightness information (remember $Y = 0.299R + 0.587G + 0.114B$). So the sensor spends more of its sites on green.

**The lecturer's "48 vs 12 pixels" point:** the slide's diagram has 8 × 6 = **48 sites**: 24 green, 12 red, 12 blue. That is exactly **12 complete 2×2 blocks** (4 across × 3 down), and each block is the smallest unit with all three colours measured. The lecturer argues that an honest full-colour image from this sensor would therefore be only **4 × 3 = 12 pixels**, one per block. Camera makers instead advertise it as a **48-pixel** camera: they output one pixel per site and **interpolate (demosaic)** the two missing colours at each site from its neighbours. So in every output pixel, one colour value was measured and the other two were estimated.

> **中文：** interpolate = **插值**；demosaic = **去马赛克**（也叫"解马赛克"）。
> 传感器上每个格子只测到一种颜色（红、绿或蓝），缺少的另外两种颜色，要用周围格子测到的值**估算**出来。这个估算过程就是**插值**；专门用在 Bayer 马赛克上的插值，叫**去马赛克**。
> **例子：** 一个红色格子只测到了红色。它的绿色值，取上下左右四个绿色邻居的平均；蓝色值，取四个对角蓝色邻居的平均。这样这个像素就有了完整的 RGB，其中红色是测量的，绿色和蓝色是估算的。

Practical takeaway: what comes out of a camera (e.g. a JPEG) has already been heavily processed and partly estimated before you ever see it. Don't treat it as "raw truth". Some high-end cameras let you download the **raw** sensor data instead, which gives you the un-interpolated mosaic.

**Important OpenCV quirk:** OpenCV stores colour images as **BGR**, not RGB (reversed channel order) — a common gotcha.
```cpp
Mat bgr_image, grey_image;
// Colour -> greyscale: 3 channels in, 1 channel out,
// using the weighted formula Y = 0.299R + 0.587G + 0.114B
cvtColor(bgr_image, grey_image, CV_BGR2GRAY);

// Split the colour image into its 3 separate channels (NOT a greyscale conversion)
vector<Mat> bgr_images(3);
split(bgr_image, bgr_images);
Mat& blue_image = bgr_images[0];   // channel 0 = Blue in OpenCV (order is B, G, R)
```
Two different things happen here:
- **`cvtColor(..., CV_BGR2GRAY)`** is the real greyscale conversion. It combines all three channels with the weighted formula into one brightness value per pixel.
- **`split`** only separates the channels. Each of the 3 results is a single-channel image, so it *looks* grey when displayed, but it holds only one colour's brightness. It is **not** the proper greyscale image.
  *Example:* a bright red apple looks bright in the red channel, dark in the blue channel, and medium grey in the real greyscale image.

**Example processing code** (per-pixel colour inversion, i.e. 255 − value):
```cpp
void InvertColour( Mat& input_image, Mat& output_image )
{
  // Only accept colour images: 8-bit unsigned (0-255), 3 channels (B, G, R)
  CV_Assert( input_image.type() == CV_8UC3 );
  // Start the output as a full copy of the input
  output_image = input_image.clone();
  // Visit every pixel (row, col) and every channel (B, G, R) of that pixel
  for (int row=0; row < input_image.rows; row++)
    for (int col=0; col < input_image.cols; col++)
      for (int channel=0; channel < input_image.channels(); channel++)
        // Flip the value: 0 -> 255, 255 -> 0, 100 -> 155
        output_image.at<Vec3b>(row,col)[channel] = 255 - input_image.at<Vec3b>(row,col)[channel];
}
```
This does **not** produce greyscale. The output is still a **3-channel colour image**, with every colour flipped, like an old film negative:
- white (255, 255, 255) → black (0, 0, 0)
- blue (255, 0, 0) in BGR → yellow (0, 255, 255)

> **What is `at<Vec3b>`?** It's how you **read or write one pixel** of a colour image.
>
> - **`.at<...>(row, col)`** returns the pixel at that position. It takes **row first, then column** (i.e. y, x, not x, y). It returns a *reference*, so you can both read and assign to it:
>   ```cpp
>   Vec3b p = image.at<Vec3b>(10, 20);   // read the pixel at row 10, column 20
>   image.at<Vec3b>(10, 20) = p;         // write it back
>   ```
> - **`<Vec3b>`** tells OpenCV how to interpret the pixel's bytes: a **Vec**tor of **3** **b**ytes (each 0–255), i.e. one colour pixel with **[0] = Blue, [1] = Green, [2] = Red**.
>   ```cpp
>   Vec3b pixel = image.at<Vec3b>(row, col);
>   uchar blue = pixel[0], green = pixel[1], red = pixel[2];
>   ```
> - **The type must match the image**, or you read garbage or crash:
>
>   | Image type | Meaning | Use |
>   |---|---|---|
>   | `CV_8UC3` | colour, 3 × 8-bit | `at<Vec3b>` |
>   | `CV_8UC1` | greyscale, 1 × 8-bit | `at<uchar>` |
>   | `CV_32FC1` | 1 channel of float | `at<float>` |
>
>   That's why `InvertColour` starts with `CV_Assert(input_image.type() == CV_8UC3)`. For a greyscale image you'd use `at<uchar>` and get one number, as in `ChangeQuantisationGrey` (§2.3).
A faster (but less readable) version uses raw pointer arithmetic instead of `.at<>()` — useful for real-time processing, but the lecturer explicitly recommends learning/using the clearer `.at<>()` style first.
```cpp
int image_rows = image.rows;
int image_columns = image.cols;
for (int row=0; row < image_rows; row++) {
  uchar* value = image.ptr<uchar>(row);
  uchar* result_value = result_image.ptr<uchar>(row);
  for (int column=0; column < image_columns; column++) {
    *result_value++ = *value++ ^ 0xFF;   // XOR with 0xFF == 255-value for unsigned bytes
    *result_value++ = *value++ ^ 0xFF;
    *result_value++ = *value++ ^ 0xFF;
  }
}
```

### 3.3 CMY — Cyan, Magenta, Yellow
- **Subtractive** colour model (used in printers), as opposed to RGB's **additive** model.
- Simple inverse relationship to RGB: $C = 255-R$, $M = 255-G$, $Y = 255-B$.
- *Why subtractive?* RGB starts from black (no light) and adds colour to reach white — that's how screens work (emitting light). Printing starts from white paper and *removes* light by adding ink, working back towards black — hence "subtractive."
- **Not supported directly in OpenCV** (trivial to compute manually if ever needed, so no built-in function).

> **中文：** **减色模型**（用于打印机），与 RGB 的**加色模型**相对。
> - Subtractive colour model = **减色模型**（也叫"减色法"）；Additive colour model = **加色模型**（也叫"加色法"）
> - **加色（RGB）**：从黑色开始，**加入**红、绿、蓝光，三种光全部叠加就是白色。屏幕发光，所以用加色。
> - **减色（CMY）**：从白纸开始，每加一层墨水，就**吸收（减去）**一部分光，三种墨水全部叠加就接近黑色。打印机用墨水，所以用减色。

### 3.4 YUV
- Used for analogue TV (PAL, NTSC).
- Separates luminance (Y) from chrominance (U, V).
- Conversion from RGB:
$$ Y = 0.299R + 0.587G + 0.114B \qquad U = 0.492(B-Y) \qquad V = 0.877(R-Y) $$
- **Why it's interesting:** early television engineers discovered humans perceive **luminance far more sharply than chrominance** — so they allocated bandwidth unevenly: 4 samples of Y for every 1 of U and 1 of V (so 2/3 of transmitted data is luminance, 1/3 is colour information split two ways). This works because human vision tends to "snap" colour perception to luminance edges/boundaries and doesn't notice the lower colour resolution. Not commonly used for CV processing directly, though some research does use it.

> **中文：** 早期的电视工程师发现，人眼对**亮度**的感知比对**色度**敏锐得多。所以他们不平均分配带宽：每传 4 个 Y（亮度）样本，只传 1 个 U 和 1 个 V（色度）样本。这样，传输的数据里 **2/3 是亮度**，**1/3 是颜色信息**（由 U 和 V 平分）。这个做法行得通，是因为人眼倾向于把颜色"贴合"到亮度的边缘和边界上，察觉不到颜色的分辨率其实更低。YUV 一般不直接用于计算机视觉处理，不过也有一些研究会用到它。
> - 术语：Luminance = **亮度**；Chrominance = **色度**；Bandwidth = **带宽**；Sample = **样本（采样）**

### 3.5 HLS — Hue, Luminance, Saturation
The lecturer's preferred representation — separates luminance (brightness/greyscale) from two human-intuitive chrominance values:
- **Hue** — the "actual colour" (red, yellow, green, blue, magenta, back to red) — **0°–360°** conceptually, but note it's **circular**: 359° and 0° are neighbours, not opposites. In OpenCV specifically it's scaled to **0–179** (i.e., 360/2, since OpenCV's 8-bit channels only go to 255 and 360 doesn't fit in a byte — halving keeps values within a single byte).
- **Luminance** — 0 to 1 (brightness/greyscale-equivalent). 0 = black, 1 = white, 0.5 = the purest version of the colour.
- **Saturation** — 0 to 1, "how full/vivid" the colour is — distance from the central (grey) axis of the HLS **double cone** (two cones joined base to base: black at the bottom tip, white at the top tip). Low saturation ≈ washed-out/grey; high saturation ≈ vivid, pure colour.
- ⚠️ **OpenCV ranges:** 0–1 is the theoretical range. In an 8-bit OpenCV image, **L and S are stored as 0–255**, and H as 0–179.

> **中文详解：HLS（色调、亮度、饱和度）**
>
> HLS 用**人比较直观的三个问题**来描述颜色，而不是像 RGB 那样说"红、绿、蓝各有多少"：
> 1. **是什么颜色？** → **Hue 色调**
> 2. **有多亮？** → **Luminance 亮度**
> 3. **颜色有多"浓"？** → **Saturation 饱和度**
>
> 最大的好处是把**亮度**（明暗）和**色度**（颜色本身）分开了。在 RGB 里，一个物体从阳光下走进阴影，R、G、B 三个值会同时变化；在 HLS 里，主要只有 L 变，H 基本不变。所以要找"红色的东西"（比如红色路标）时，看 H 就行，不太受光照影响。这也是讲师偏爱 HLS 的原因。
>
> **1. Hue 色调：是什么颜色**
> - 把所有颜色排成一个**色环**：红 → 黄 → 绿 → 青 → 蓝 → 品红 → 回到红。
> - 用**角度**表示：0° 红，60° 黄，120° 绿，180° 青，240° 蓝，300° 品红，360° 又回到红。
> - **它是一个圈**：359° 和 0° 挨在一起，都是红色，不是两个极端。所以不能直接对色调做普通的平均。比如 1° 和 359° 都是红，直接平均得到 180°，是青色，完全错了。
> - **OpenCV 里是 0–179**：每个通道只有 8 位，最大 255，放不下 360，所以 OpenCV 把角度除以 2。例如蓝色 240°，在 OpenCV 里存成 120。
>
> **2. Luminance 亮度：有多亮**
> - 范围 0 到 1：**0 是黑色，1 是白色，0.5 是"最纯"的颜色**。可以理解成这个颜色变成黑白照片后的明暗程度。
> - 例子：纯红 RGB(255, 0, 0) 的 L = 0.5；深红 RGB(128, 0, 0) 的 L ≈ 0.25，更暗。
>
> **3. Saturation 饱和度：颜色有多"浓"**
> - 范围 0 到 1：**0 是完全灰色（没有颜色），1 是最鲜艳、最纯的颜色**。
> - 例子：纯红 RGB(255, 0, 0) 的 S = 1，非常鲜艳；灰粉红 RGB(200, 150, 150) 的 S ≈ 0.31，像"褪色"了；灰色 RGB(128, 128, 128) 的 S = 0，完全没有颜色。
>
> **把三者放在一起想象：双圆锥**（两个圆锥底对底拼在一起）
> - **上下方向 = 亮度 L**：底尖是黑，顶尖是白，中间最宽处是 L = 0.5。
> - **绕着轴转一圈 = 色调 H**：转到哪个角度，就是哪种颜色。
> - **离中心轴的距离 = 饱和度 S**：越靠外越鲜艳，中心轴上是纯灰色。
>
> 两头是尖的，因为接近纯黑或纯白时，颜色都会消失。这也解释了下面的"陷阱"：**亮度太高或太低、或者饱和度很低时，H 基本没有意义**，因为在尖端附近或中心轴附近，已经分不出是什么颜色了。
>
> **注意：** 在 OpenCV 的 8 位图像里，L 和 S 存的是 **0–255**，H 是 **0–179**；0–1 只是理论范围。
>
> 术语：Hue = **色调**；Luminance = **亮度**；Saturation = **饱和度**；Chrominance = **色度**；circular = **循环的（首尾相接）**

**Conversion from RGB:**
$$ L = \frac{\max(R,G,B) + \min(R,G,B)}{2} $$
$$ S = \begin{cases} \dfrac{\max-\min}{\max+\min} & L < 0.5 \\[4pt] \dfrac{\max-\min}{2-(\max+\min)} & L \geq 0.5 \end{cases} $$
$$ H = \begin{cases} 60 \cdot (G-B)/S & \text{if } R = \max \\ 120 + 60 \cdot (B-R)/S & \text{if } G = \max \\ 240 + 60 \cdot (R-G)/S & \text{if } B = \max \end{cases} $$
*(You are not expected to memorise this derivation — the point is just that a well-defined, reversible conversion exists.)*

```cpp
cvtColor(bgr_image, hls_image, CV_BGR2HLS);
// Hue ranges from 0 to 179 in OpenCV.
```

**⚠️ Practical pitfalls with HLS (important, easy to get wrong):**
1. **Circular hue** — you cannot apply normal arithmetic/statistics directly to hue values (e.g. averaging hue=1° and hue=359° naively gives ≈180°, which is wrong — the "true" average is ≈0°). Standard vision operations often need to either work in RGB and convert back, or use adapted circular-aware math.
2. **Very low or very high luminance** (near black or near white) → hue and saturation become close to **meaningless/undefined** — you must check luminance isn't extreme before trusting hue/saturation.
3. **Very low saturation** (near-grey pixels) → hue is essentially **noise**, since there's barely any real colour information driving it, and small pixel variations can flip the computed hue wildly.

### 3.6 Other colour spaces (aware-of level only — not required to memorise)
- **HSV** — Hue, Saturation, Value — similar concept to HLS, slightly different channel definitions.
- **YCrCb** — a scaled version of YUV, common in image/video compression.
- **CIE XYZ** — standard reference colour space; channel responses modelled on the human eye's cone responses. Defined by the CIE (Commission Internationale de l'Éclairage / International Commission on Illumination).
- **CIE L\*u\*v\*** — perceptually uniform colour space: equal numeric differences ≈ equal *perceived* colour differences.
- **CIE L\*a\*b\*** — device-independent space covering all humanly-perceivable colours (notably, **RGB does not** cover the full range of colours humans can perceive).
- **Bayer** — the raw sensor mosaic pattern itself (see §3.2); relevant if working with raw camera sensor data rather than an already-interpolated image.

**Takeaway the lecturer emphasises:** you really only need to *know well* **RGB** and **HLS** (plus greyscale). The rest are good to be aware exist, but not to memorise formulas for.

---

## 4. Noise

### 4.1 What is noise, and where does it come from?
Virtually **all real acquired images contain noise** (an image with zero noise is basically only possible if artificially/synthetically generated). Noise degrades images and complicates processing — ideally, running the same processing on two images of an *unchanged* scene should give the same result, but noise can cause it not to.

**Causes** (many small contributing factors that add up):
- The environment
- The imaging device itself
- Electrical interference
- The digitisation process
- Transmission (e.g. satellite image transmission)

### 4.2 Measuring noise — Signal-to-Noise Ratio (SNR)
$$ \text{S/N ratio} = \frac{\sum_{(i,j)} f^2(i,j)}{\sum_{(i,j)} v^2(i,j)} $$
where $f$ is the (clean) signal and $v$ is the noise. Higher SNR = better (less relative noise). Both terms are squared (this is a power ratio, analogous to how SNR is defined generally in signal processing/engineering).

### 4.3 Two key noise types
| Type | Description |
|---|---|
| **Gaussian noise** | Most common/realistic; a value drawn from a Gaussian (normal) distribution — defined by **mean and standard deviation** — is added/subtracted per pixel. Represents typical camera sensor noise; often subtle enough that you don't consciously notice it, but it's there. |
| **Salt-and-pepper noise** | "Impulse" noise — a pixel is pushed to the **maximum or minimum** possible value (pure white or pure black speckles), rather than a small perturbation. |

The more noise present, the *less* effective any smoothing/correction technique will be — noise cannot generally be fully "removed," only **attenuated**. (The lecturer is careful to avoid the word "remove," preferring "attenuate" throughout.)

> **A subtlety worth understanding:** you can't perfectly evaluate a denoising technique on a real photo, because you don't actually know how much of the "noise" you're removing was genuine sensor noise vs. real fine detail in the scene — unless you start from a fully synthetic (noise-free) image and add known noise yourself. But then you don't know how well results generalise back to real images. This is a recurring theme in computer vision evaluation.

---

## 5. Smoothing (Noise Reduction)

Smoothing = the general technique for attenuating noise, by looking at a pixel's **neighbourhood** and recombining the values in some way. Two broad categories:

### 5.1 Averaging filters (linear) — local averaging & Gaussian
Both work by **convolution**: sliding a small weighted mask (kernel) over the image and computing a weighted sum of the neighbourhood at each position:
$$ f(i,j) = \sum_{(m,n)} h(i-m, j-n) \cdot g(m,n) $$

**Local average mask (3×3):**
$$ h = \frac{1}{9}\begin{bmatrix}1&1&1\\1&1&1\\1&1&1\end{bmatrix} $$
Every neighbour weighted equally.

**Gaussian masks** — weight neighbours according to a Gaussian curve, giving more weight to the centre pixel and less to pixels further away:
$$ h = \frac{1}{10}\begin{bmatrix}1&1&1\\1&2&1\\1&1&1\end{bmatrix} \qquad h = \frac{1}{16}\begin{bmatrix}1&2&1\\2&4&2\\1&2&1\end{bmatrix} $$

Larger/wider Gaussian kernels can be used for a more mathematically "complete" Gaussian shape — the lecturer mentions having used kernels over 100×100 pixels in some contexts. There's an inherent **trade-off**: more smoothing reduces noise more, but also blurs/damages real image detail (especially edges) more.

```cpp
blur(image, smoothed_image, Size(3,3));
GaussianBlur(image, smoothed_image, Size(5,5), 1.5);  // last param = standard deviation
```

**Acceptable-results check:** does it (a) suppress small image noise, while (b) not blurring edges too badly? These two goals are in tension — that tension is the core trade-off of smoothing.

> **中文详解：平均滤波与高斯滤波**
>
> **这一节在做什么？** 目标是**平滑（smoothing）**：减弱图像里的噪声。噪声就是某些像素值"随机跳了一下"，和周围不一致。如果把每个像素换成**它和周围邻居的平均值**，这些随机的跳动就会被拉平。
>
> **1. 卷积（convolution）：怎么"取平均"**
>
> 拿一个小方格（叫**掩模 mask** 或**卷积核 kernel**，比如 3×3），里面每格有一个**权重**：
> 1. 把核的中心对准某个像素。
> 2. 核的 9 个格子分别乘上它下面的 9 个像素值。
> 3. 把 9 个乘积加起来，得到这个像素的新值。
> 4. 把核**滑动**到下一个像素，重复，直到走遍整张图。
>
> 公式 $f(i,j) = \sum_{(m,n)} h(i-m, j-n) \cdot g(m,n)$ 说的就是这件事：$g$ 是**输入图像**，$f$ 是**输出图像**，$h$ 是**卷积核**（权重），$\sum$ 就是"把邻域里所有的乘积加起来"。（公式里的 $i-m$ 表示核是"翻转"后再乘的。这门课里的核都是对称的，翻不翻结果一样，可以忽略。）
>
> **2. 局部平均核（local average）**
> - 9 个格子权重都是 1，也就是**每个邻居一样重要**。
> - 前面的 **1/9** 是为了让权重加起来等于 1，这样整张图的平均亮度不会变亮或变暗。
> - 效果：新值 = 9 个像素的**普通平均数**。
>
> **例子：** 一块均匀的区域，中间有一个噪点：
> ```
> 10  10  10
> 10 100  10     ← 中间的 100 是噪点
> 10  10  10
> ```
> 平均后中间像素 = (8×10 + 100) / 9 = 180 / 9 = **20**。噪点从 100 降到了 20，被大大减弱了。
>
> **3. 高斯核（Gaussian）**
> - 权重**不再相同**：以 1/16 的核为例，**中心最大（4），上下左右次之（2），四个角最小（1）**。
> - 意思是：**离中心越近的邻居越重要**。权重的分布像一个"钟形曲线"（高斯曲线 / 正态分布），所以叫高斯核。
> - 1/16 同样是为了让权重总和等于 1（1+2+1+2+4+2+1+2+1 = 16）。1/10 的核也是一样的道理。
>
> **同一个例子用 1/16 高斯核：** 中心 4 × 100 = 400；上下左右 4 个 × 2 × 10 = 80；四个角 4 个 × 1 × 10 = 40；合计 520 / 16 = **32.5**。
> 结果（32.5）比普通平均（20）更接近原来的中心值：高斯核平滑得**更温和**，对原图的破坏也更小，所以实际中用得最多。更大的高斯核（5×5，甚至讲师提到的 100×100 以上）能更完整地表示高斯曲线，也能平滑得更强。
>
> **4. OpenCV 代码**
> - `blur(image, smoothed_image, Size(3,3));`：局部平均，3×3 的核，每个邻居权重一样。
> - `GaussianBlur(image, smoothed_image, Size(5,5), 1.5);`：高斯平滑，5×5 的核。1.5 是**标准差 σ（sigma）**，控制钟形曲线有多"宽"：σ 越大，远处邻居的权重越大，图像越模糊。
> - 核的尺寸一般用**奇数**（3、5、7…），这样才有一个确切的中心像素。
>
> **5. 核心矛盾（trade-off）：去噪 vs 保留边缘**
>
> 平滑**分不清噪声和真实的细节**，两者都会被拉平。例子：一条黑白分界线（边缘），一行像素是：
> ```
> 平滑前：  0    0    0  255  255  255     ← 边缘很清晰，一步跳变
> 平滑后：  0    0   85  170  255  255     ← 用宽度 3 取平均
> ```
> 原本一步就从黑跳到白，平滑后变成了逐渐过渡，**边缘被"糊"开了**。
>
> 所以判断平滑效果，要同时看：**(a) 噪声有没有被压下去？**（越平滑越好）和 **(b) 边缘有没有被糊得太厉害？**（越平滑越差）。这两个目标**互相冲突**，选核的大小、选 σ，本质上就是在两者之间找平衡。这也是为什么接下来会讲**中值滤波（median filter）**：它去噪的同时，对边缘的破坏小得多。
>
> 术语：smoothing = **平滑**；convolution = **卷积**；mask / kernel = **掩模 / 卷积核**；weighted sum = **加权和**；neighbourhood = **邻域**；Gaussian = **高斯**；standard deviation = **标准差**；edge = **边缘**；trade-off = **权衡**

### 5.2 Median filter (non-linear)
Instead of averaging, take the **median** value of the neighbourhood.

**Worked example (3×3 = 9 values):**
`11 18 20 21 23 25 25 30 250` → **Median = 23**, but **Average = 47**.
The single outlier (250 — e.g. a salt-and-pepper spike) drags the average way up, but barely moves the median. This is exactly why median filtering is so effective against salt-and-pepper noise specifically: it's naturally robust to extreme outliers, as long as they don't make up *most* of the neighbourhood (e.g. 4–5 of 9 values being extreme would start shifting the median too).

**Strengths:**
- Not (much) affected by noise, especially isolated outliers.
- Doesn't blur edges nearly as much as averaging filters — regions stay sharp with clear boundaries between them.
- Can be applied iteratively (repeatedly) to keep improving the result.

**Weaknesses:**
- **Damages thin lines and sharp corners** — it effectively changes the shape of small/fine features. Think of the median as a **majority vote**: any structure that covers **less than half** of the window gets voted out, whether it's noise or a real detail. With a plus-shaped window (centre + up/down/left/right), a right-angle corner aligned with the window keeps its tip (3 of 5 pixels are object), but the tip of a 45° corner has only 2 of 5, so it gets shaved off and the corner is rounded. Thin lines suffer the same way: a 1-pixel line covers only 3 of 9 pixels in a 3×3 square window, so it disappears entirely. Every window shape favours some directions over others, so no shape can preserve thin lines and sharp corners in *every* orientation.
- **Computationally more expensive** historically:
  - Standard implementation: **O(r² log r)** (r = filter radius)
  - Huang's algorithm: **O(r)**
  - Perreault (2007): **O(1)** — constant time regardless of filter size, a big improvement.
- Despite arguably being a *better* filter in many respects, the lecturer notes it's **not as commonly used** in practice as Gaussian/averaging approaches — no strong reason given beyond convention/tooling.

```cpp
medianBlur(image, smoothed_image, 5);   // 5 = aperture size
```

> **中文详解：中值滤波**
>
> **核心：中值滤波就是"投票"。** 中值（median）的算法是把窗口里的所有像素值**从小到大排序，取正中间那个**。3×3 窗口有 9 个值，排序后取**第 5 个**，例如 `11 18 20 21 [23] 25 25 30 250` → 中值 = 23。
>
> 关键点：**中值一定是窗口里真实存在的某个值**，永远不会算出一个"介于中间"的新值；平均值却会算出原图里根本没有的数。当窗口里的值分成"暗的一群"和"亮的一群"时，排在正中间的那个一定属于**人数更多的一群**。所以中值滤波就像**多数派投票**：窗口里哪一方占多数，中心像素就变成哪一方。
>
> ---
>
> **问题一：为什么中值滤波不会模糊边缘？**
>
> 一维例子：一行像素左边黑、右边白，用宽度 3 的窗口（左邻、自己、右邻）：
>
> | 像素 | 窗口 | 排序 | 中值 | 平均值 |
> |---|---|---|---|---|
> | 第 4 个（0） | 0, 0, 255 | 0, **0**, 255 | **0** | 85 |
> | 第 5 个（255） | 0, 255, 255 | 0, **255**, 255 | **255** | 170 |
>
> ```
> 原图：       0    0    0    0  255  255  255  255
> 中值滤波后： 0    0    0    0  255  255  255  255   ← 和原图一样，边缘仍然清晰
> 平均滤波后： 0    0    0   85  170  255  255  255   ← 出现了 85、170，边缘变模糊
> ```
>
> 二维例子（3×3 窗口），一条竖直的边缘：
> ```
> 0   0  255  255
> 0   0  255  255
> 0   0  255  255
> ```
> - 边缘左侧的像素（0）：窗口里 6 个 0、3 个 255 → `0 0 0 0 [0] 0 255 255 255` → 中值 = **0**，保持黑色。
> - 边缘右侧的像素（255）：窗口里 3 个 0、6 个 255 → `0 0 0 255 [255] 255 255 255 255` → 中值 = **255**，保持白色。
>
> **原因：** 在一条直的边缘上，每个像素"自己这一侧"的邻居总是占多数，所以投票结果总是它原来的值，边缘不会被糊开。平均滤波则会把两边的值**混合**成中间值。
>
> **偶数个值怎么办？（例如 0, 0, 0, 0, 255, 255, 255, 255）** 8 个值没有"正中间"那一个，数学上取中间两个的平均：(0 + 255) / 2 = **127.5**，这就变成模糊的中间值了。但**实际中不会出现**，因为中值滤波的窗口永远是**奇数个像素**（3×3 = 9，5×5 = 25；OpenCV 的 `medianBlur` 也要求奇数）。所以总有唯一的正中间值，两群里总有一方多一个。例如 9 个值里 4 个 0、5 个 255：`0 0 0 0 [255] 255 255 255 255` → 中值 = 255。结果是 0 或 255，**永远不会是 127**。真实图像有噪声（如 `3, 0, 5, 2, 250, 255, 251, 248, 253`），中值仍会从占多数的那一群里挑一个真实的值（这里是 250），边缘依然清晰。
>
> ---
>
> **问题二：为什么 45° 的角会被去掉？**
>
> 还是用"投票"来想：**如果某个结构在窗口里占不到一半，它就会被投掉。**
> - 孤立噪点：只占 9 个里的 1 个 → 被投掉 ✅（好事）
> - 细线、尖角的尖端：也只占窗口里的少数 → 同样被投掉 ❌（坏事）
>
> 中值滤波**分不清"噪点"和"又细又尖的真实结构"**，只看数量。
>
> 用十字形窗口来看（只看自己和上下左右，共 5 个像素，至少 **3 个**是物体才能保留）。下面 `#` 是物体，`.` 是背景：
> ```
> 十字形窗口：
> .  X  .
> X  X  X
> .  X  .
> ```
>
> **情况 A：直角（90°），和窗口方向对齐**
> ```
> .  .  .  .
> .  #  #  #     ← 尖端（第 2 行第 2 列）
> .  #  #  #
> .  #  #  #
> ```
> 以尖端为中心：上 `.`、左 `.`、中 `#`、右 `#`、下 `#` → **3 个 #** → 中值 = `#`，**尖端保留** ✅
>
> **情况 B：45° 的角（尖端朝上的三角形）**
> ```
> .  .  .  .  .
> .  .  #  .  .     ← 尖端（第 2 行第 3 列）
> .  #  #  #  .
> #  #  #  #  #
> ```
> 以尖端为中心：上 `.`、左 `.`、中 `#`、右 `.`、下 `#` → **只有 2 个 #** → 中值 = `.`，**尖端被删掉** ❌
>
> 原因：斜着的角，两条边沿对角线方向走，而十字形窗口**根本不看对角线方向的邻居**，所以尖端只有自己和下面一个"支持者"，输给了背景。角越尖，尖端在窗口里占的比例越小，被削掉的越多，结果就是**角被磨圆**。
>
> **细线也是同样的道理：**
> - 一个像素宽的**水平线**：十字形窗口里中、左、右 = 3/5 → 保留 ✅；3×3 方形窗口里只有 3/9 → **整条线被删掉** ❌
> - 一个像素宽的 **45° 斜线**：十字形窗口只有中心 1/5 → 删掉 ❌；方形窗口 3/9 → 删掉 ❌
>
> **"无论怎么改窗口形状都没用"：** 每种窗口形状都**偏向某些方向**。十字形保得住水平、竖直的线和直角，保不住斜线和 45° 角；换成 X 形能保住斜线，水平、竖直的线又会被删掉；方形窗口对一个像素宽的线，任何方向都保不住。所以**找不到一种窗口形状能保住所有方向的细线和尖角**。
>
> （补充：用 3×3 方形窗口时，直角尖端也只有 4/9 是物体，也会被削掉 1 个像素。讲师说"直角能较好地保留"，最符合十字形窗口的情况。）
>
> ---
>
> **优点：**
> 1. **不太受噪声影响，尤其是孤立的极端值**：一个 250 混在一堆 20 左右的值里，排序后被挤到最后，影响不到中间的值；平均值却会被它拉高很多（上面的例子：中值 23，平均 47）。
> 2. **不模糊边缘**：见问题一。
> 3. **可以反复使用（iteratively）**：对结果再做一次中值滤波，还能继续去掉剩下的噪点。反复做到某一步后，图像就不再变化了。
>
> **缺点：**
> 1. **破坏细线和尖角**：见问题二。
> 2. **计算量比较大**：平均滤波只需要加法；中值滤波每个像素都要**排序**，更慢。r 是窗口半径，窗口大小是 (2r+1)×(2r+1)：
>    - **普通做法 O(r² log r)**：每个像素都把窗口里约 r² 个值排序一遍，窗口越大越慢。
>    - **Huang 算法 O(r)**：窗口向右滑动一格时，只有最左边一列离开、最右边新一列进来。用一个**直方图**记录窗口里的值，每次只更新这两列（约 2r 个像素），不用重新排序。
>    - **Perreault（2007）O(1)**：进一步为每一列预先保存直方图，每次更新的工作量和窗口大小**无关**，窗口再大也一样快。
>
> 术语：median = **中值**；median filter = **中值滤波**；outlier = **离群值 / 极端值**；iterative = **迭代的 / 反复的**；thin line = **细线**；corner = **角**；window = **窗口**；histogram = **直方图**

### 5.3 Choosing a filter for the right noise type
- **Salt-and-pepper noise → median filter** is clearly best; averaging filters leave visible residual noise because the extreme values still drag the local average.
- **Gaussian noise → Gaussian blur or local averaging** tend to work at least as well as, or better than, median filtering — median still helps, but the improvement over averaging methods is less dramatic here since there are no single extreme outliers to reject.

> **中文详解：根据噪声类型选择滤波器**
>
> **先看两种噪声的区别：**
>
> | | **椒盐噪声（salt-and-pepper）** | **高斯噪声（Gaussian）** |
> |---|---|---|
> | 影响多少像素？ | **少数**像素 | **几乎每个**像素 |
> | 误差有多大？ | **极大**：直接变成纯白 255（盐）或纯黑 0（胡椒） | **很小**：在原值附近随机偏一点，比如 ±5 |
> | 看起来像什么？ | 图上撒了一些黑白小点 | 整张图有一层细细的"颗粒感" |
>
> 一句话：**椒盐噪声 = 少数像素错得离谱；高斯噪声 = 所有像素都错一点点。**
>
> 两种滤波器的特长也不同：**中值滤波**擅长**剔除离群值**（投票，把少数派投掉）；**平均 / 高斯滤波**擅长**抵消大量的小误差**（有的偏高、有的偏低，加起来互相抵消）。所以规律是：**什么样的噪声，就配擅长对付它的滤波器。**
>
> ---
>
> **1. 椒盐噪声 → 用中值滤波**
>
> 例子：一块均匀的灰色区域（值都是 50），中间有一个"盐"点（255）：
> ```
> 50  50  50
> 50 255  50
> 50  50  50
> ```
> - **平均滤波：** (8 × 50 + 255) / 9 = 655 / 9 ≈ **73**。噪点没有消失，只是从 255 变成了 73，变成一个**灰色的污点**。而且周围 8 个邻居的窗口里也都包含这个 255，它们也会被拉高一点，结果一个白点变成了一小**片**略亮的斑块。这就是"留下明显的残余噪声"：极端值还在把平均值往上拉。
> - **中值滤波：** 排序 `50 50 50 50 [50] 50 50 50 255` → 中值 = **50**。噪点**完全消失**，周围也不受影响。
>
> 结论：椒盐噪声的每个噪点都是一个"离群值"，正好是中值滤波最擅长剔除的东西。平均滤波只能把它**稀释**，去不掉。
>
> ---
>
> **2. 高斯噪声 → 用高斯滤波或平均滤波**
>
> 例子：某块区域的真实值都是 100，加了高斯噪声后变成：
> ```
>  97  104   99
> 102   95  101
> 103   98  100
> ```
> 每个像素都偏了一点，有的偏高、有的偏低，**没有哪一个特别离谱**。
> - **平均滤波：** 899 / 9 ≈ **99.9**，很接近真实值 100。偏高的和偏低的**互相抵消**了。
> - **中值滤波：** 排序 `95 97 98 99 [100] 101 102 103 104` → 中值 = **100**，这个例子里也不错。
>
> 为什么说平均滤波"至少一样好，甚至更好"？
> - **平均值用上了全部 9 个值的信息**，每个值都参与了抵消误差。
> - **中值只挑了中间那 1 个值**，其他 8 个值只是用来排序，它们的具体大小被浪费了。
> - 这里**没有离群值需要剔除**，中值滤波最大的优势用不上，反而因为"浪费信息"，平均下来去噪效果**略差**。（统计上，对于高斯噪声，平均值是最好的估计方法；中值估计的误差大约是平均值的 1.25 倍。）
>
> 所以：对高斯噪声，中值滤波**也有帮助**，但比起平均类方法，**没有明显优势**。
>
> ---
>
> **3. 总结**
>
> | 噪声类型 | 最好的选择 | 原因 |
> |---|---|---|
> | 椒盐噪声 | **中值滤波** | 噪点是离群值，投票直接把它投掉；平均只能稀释，留下污点 |
> | 高斯噪声 | **高斯 / 平均滤波** | 每个像素都有小误差，平均能让它们互相抵消；中值没有离群值可剔除，还浪费了信息 |
>
> **实际应用中还要考虑：**
> - 如果**边缘很重要**，即使是高斯噪声，也可能选中值滤波，因为它不模糊边缘（见 §5.2）。
> - 真实图像的噪声往往是**混合**的。可以先用中值滤波去掉椒盐点，再用高斯滤波去掉剩下的颗粒感。
>
> 术语：salt-and-pepper noise = **椒盐噪声**；Gaussian noise = **高斯噪声**；outlier = **离群值**；residual noise = **残余噪声**；local averaging = **局部平均**

### 5.4 Effect of mask/filter size
Increasing the size of *any* smoothing filter increases noise suppression, but also increases damage to genuine shape/detail in the image — at large enough sizes, Gaussian noise "disappears," but the underlying shapes in the scene become seriously distorted. Filter size is therefore always a **balance**, tuned to the task.

> **Practical resolution tip:** if you have a high-resolution image and plan to downsample it anyway, it can be better to **apply smoothing *before* downsizing** rather than after. Two reasons:
> 1. **Smoothing first uses all the original pixels.** Downsampling throws most pixels away. If you smooth first, each surviving pixel has already averaged in its neighbours, so their information helps cancel the noise instead of being discarded.
> 2. **Smoothing first prevents aliasing.** Fine detail that is too dense for the smaller image can turn into false patterns when you just keep every 2nd pixel. E.g. stripes `0 255 0 255 ...` subsampled become all black (or all white) instead of mid-grey. This is why image pyramids (§6) always smooth *then* subsample.

> **中文详解：滤波器尺寸的影响**
>
> **核心：窗口越大，去噪越强，但细节也毁得越多。** "掩模 / 滤波器尺寸"就是平滑时用的窗口有多大：3×3、5×5、9×9……（对应 OpenCV 里的 `Size(3,3)`、`Size(5,5)`，或 `medianBlur` 的 `5`）。
>
> **为什么窗口越大，去噪越强？** 参与平均的像素越多，随机误差抵消得越彻底：
>
> | 窗口 | 参与平均的像素 | 高斯噪声大约减小到 |
> |---|---|---|
> | 3×3 | 9 个 | 原来的 1/3 |
> | 5×5 | 25 个 | 原来的 1/5 |
> | 9×9 | 81 个 | 原来的 1/9 |
>
> （规律：N 个独立的随机误差平均后，大约减小到原来的 1/√N。）所以窗口足够大时，噪声几乎"消失"了。
>
> **为什么窗口越大，细节毁得越多？** 平滑分不清噪声和真实细节。窗口越大，每个像素就被越远的邻居影响，于是：
>
> 1. **边缘被糊得更宽**：同一条黑白边缘，用不同宽度的窗口取平均：
>    ```
>    原图：        0    0    0    0  255  255  255  255
>    宽度 3 平均： 0    0    0   85  170  255  255  255    ← 过渡 2 个像素
>    宽度 5 平均： 0    0   51  102  153  204  255  255    ← 过渡 4 个像素，更模糊
>    ```
> 2. **比窗口小的物体被"抹掉"**：例如黑色背景上一条 3 个像素宽的白线（255）。用宽度 3 的窗口，线的中心仍然是 255；用宽度 9 的窗口，线中心 = 3 × 255 / 9 = **85**，变成一条淡淡的灰线，几乎看不见。
> 3. **形状被扭曲**：尖角被磨圆，细小的凸起被削平，相近的两个物体被糊在一起。中值滤波也一样：窗口越大，"投票"时被投掉的结构越大（见 §5.2）。
>
> **所以尺寸要"按任务来调"：** 没有万能的尺寸，要看**你关心的东西有多大**。找**大物体**（整个人、整辆车）可以用大窗口，狠狠去噪；找**小细节**（文字、细线、PCB 线路）只能用小窗口，宁可留一点噪声。**经验法则：窗口要明显小于你想保留的最小细节。**
>
> ---
>
> **实用技巧：先平滑，再缩小图像**
>
> 如果你有一张高分辨率图像，反正要把它缩小（降采样，downsample），**先平滑再缩小**通常比**先缩小再平滑**好。最简单的缩小方法是**每隔一个像素取一个**，其他的直接丢掉（尺寸变成一半，像素变成 1/4）。
>
> **原因 1：先平滑，能用上所有像素的信息。**
> - **先缩小：** 直接丢掉 3/4 的像素。留下来的像素**带着原来的噪声**，丢掉的像素里本来可以帮忙抵消噪声的信息也没了。之后再平滑，只能用剩下 1/4 的数据。
> - **先平滑：** 每个像素先和周围邻居平均，噪声已经被抵消掉一部分。再缩小时，留下的每个像素都**已经吸收了周围像素的信息**，丢掉的像素没有"白丢"。
>
> **原因 2：先平滑，能避免"混叠"（aliasing）。** 这是更重要的原因。一行黑白交替的细条纹：
> ```
> 原图：        0  255    0  255    0  255    0  255
> ```
> 它的真实平均亮度是中灰（约 127）。
> - **直接隔一个取一个：** 取到第 1、3、5、7 个 → `0  0  0  0`，**全黑**！（从第 2 个开始取就是全白。）细条纹变成了一个根本不存在的纯色，完全错了。
> - **先平滑再取：** 平滑后每个像素都接近 127，再隔一个取一个 → `127  127  127  127`，**正确的中灰**。
>
> 这种"细节太密，缩小时变成错误图案"的现象叫**混叠**。用手机拍电脑屏幕或细条纹衬衫时看到的波纹（摩尔纹），就是混叠造成的。这也是为什么 §6 的**图像金字塔**每一层都是"**先平滑，再降采样**"，两步永远一起做。
>
> 术语：mask / filter size = **掩模 / 滤波器尺寸**；noise suppression = **噪声抑制**；distort = **扭曲 / 变形**；downsample = **降采样 / 缩小**；resolution = **分辨率**；aliasing = **混叠**；moiré = **摩尔纹**

---

## 6. Image Pyramids

**Motivation:** we frequently don't know in advance the *scale* at which objects of interest will appear in an image (a face could fill the frame, or be tiny and far away). To handle this, we process the image at **multiple scales simultaneously** — this also has an efficiency benefit: you can often locate something roughly at low resolution first, then refine only that region at high resolution, rather than doing expensive processing over the whole image at full resolution.

**Technique — building a pyramid:**
1. **Smooth** the image (typically with a Gaussian).
2. **Subsample** — usually by a factor of 2 in both rows and columns (so each level has 1/4 the pixels of the level below).
3. Repeat, producing a "pyramid" of progressively smaller/coarser versions of the same image.

> **Why smooth before subsampling?** Subsampling without smoothing first can cause aliasing — but note the transcript doesn't dwell on this explicitly; it's implied by always pairing "smooth" with "subsample" as a single step. Worth keeping in mind for later in the course when sampling theory is covered more formally.

```cpp
pyrDown(image, smaller_image, Size((image.cols+1)/2, (image.rows+1)/2));
```

Image pyramids will reappear repeatedly later in the course wherever multi-scale processing is needed.

> **中文详解：图像金字塔（Image Pyramids）**
>
> **1. 要解决什么问题？**
>
> **我们事先不知道物体在图像里有多大。** 同一张脸，人站在镜头前可能占满整张图（比如 400×400 像素），站在远处可能只有 20×20 像素。但很多检测方法只能找**固定大小**的东西。比如一个人脸检测器是用 24×24 像素的人脸训练的，它就只认得大约 24×24 大小的脸，400×400 的大脸它反而认不出来。
>
> **解决办法：** 物体大小不能改，但**图像大小可以改**。把同一张图缩小成好几个不同的尺寸，总有一个尺寸下，那张脸刚好接近 24×24。在每个尺寸上都跑一遍检测器，就能找到任何大小的脸。这就是**多尺度处理（multi-scale processing）**，把这一组不同尺寸的图像叠起来，就叫**图像金字塔**。
>
> **2. 怎么构建金字塔？**
>
> 每一层做两步，不断重复：**① 平滑**（用高斯滤波模糊一下）→ **② 降采样**（行和列都**隔一个取一个**，尺寸变成一半）。从 640×480 的原图开始：
> ```
> 第 0 层（原图）：  640 × 480   ← 金字塔的底部，最大、最清晰
> 第 1 层：          320 × 240   ← 像素数是上一层的 1/4
> 第 2 层：          160 × 120
> 第 3 层：           80 × 60    ← 金字塔的顶部，最小、最粗糙
> ```
> 从大到小叠起来，形状就像一座**金字塔**。
>
> **存储代价很小：** 每层像素是上一层的 1/4，总像素数 = 1 + 1/4 + 1/16 + 1/64 + … ≈ **4/3**，整座金字塔只比原图多占用大约 **1/3** 的空间。
>
> **3. 为什么一定要"先平滑，再降采样"？**
>
> 这就是 §5.4 讲的**混叠（aliasing）**。不平滑就直接隔一个取一个，太密的细节会变成错误的图案：黑白交替的条纹 `0 255 0 255 …` 会变成全黑或全白，而正确结果应该是中灰。先平滑，把细节"合并"成它们的平均值，再缩小就不会出错。所以在金字塔里，**平滑和降采样永远被当成一步来做**。
>
> **4. 另一个好处：由粗到精，大大加快速度**
>
> 例子：要在 640×480 的图像里找一个物体。
> - **直接在原图上找：** 要检查 640 × 480 = **307,200** 个位置，很慢。
> - **由粗到精（coarse-to-fine）：**
>   1. 先在最小的第 3 层（80×60）上找，只有 **4,800** 个位置，快了 **64 倍**，能**大致**找到物体在哪。
>   2. 回到上一层（160×120），**只在刚才找到的位置附近**仔细找。
>   3. 一层层往下，最后在原图上**只检查很小的一块区域**，得到精确位置。
>
> 就像在地图上找一家店：先看全国地图找到城市，再放大到街区，最后才看街道。
>
> **5. OpenCV 代码**
> - `pyrDown` 一次完成**两步**：先用 5×5 的高斯核平滑，再隔一个取一个。
> - `Size(...)` 是输出尺寸：宽和高各减半。
> - **为什么要 +1？** 为了处理**奇数**尺寸。比如宽度是 641：不加 1，641 / 2 = 320（整数除法舍去小数），丢掉了一列；加 1，(641 + 1) / 2 = 321，向上取整，不会丢掉边缘。
> - 反复调用 `pyrDown`，就能一层一层建出整座金字塔。反方向的函数是 `pyrUp`：把图像放大一倍。
>
> **6. 总结**
> - **为什么需要：** 不知道物体会以多大的尺寸出现。
> - **怎么做：** 平滑 → 降采样（每次缩小一半）→ 重复。
> - **为什么先平滑：** 避免混叠。
> - **额外好处：** 由粗到精搜索，先在小图上快速定位，再到大图上精确定位。
> - **代价：** 只多占大约 1/3 的存储空间。
> - 后面课程里凡是需要**多尺度处理**的地方，都会用到金字塔。
>
> 术语：image pyramid = **图像金字塔**；scale = **尺度**；multi-scale = **多尺度**；subsample / downsample = **降采样**；coarse-to-fine = **由粗到精**；aliasing = **混叠**；level = **层**

---

## Quick recap — key things to remember
- **Pinhole camera model**: 3D world point → 2D image point via focal length + optical centre; used (post distortion-correction) as OpenCV's underlying model.
- Digital images require **sampling** (where to measure) + **quantisation** (what values are allowed) — both introduce information loss/artefacts (gaps/blended edge pixels; false contours at low bit-depth).
- **RGB** is the default working colour space; OpenCV stores it as **BGR**. Know the greyscale conversion formula and the Bayer pattern/interpolation story.
- **HLS** separates luminance from hue/saturation and is the most human-intuitive space — but watch out for **circular hue**, and unreliable hue/saturation at extreme luminance or low saturation.
- Two core noise types: **Gaussian** (per-pixel random perturbation) and **salt-and-pepper** (impulse, max/min values).
- **Averaging/Gaussian filters** (linear) vs **median filter** (non-linear): median is best for salt-and-pepper and preserves edges better, but damages corners/thin lines and (historically) was more expensive to compute.
- **Image pyramids** = smooth + subsample repeatedly, for efficient multi-scale processing.
