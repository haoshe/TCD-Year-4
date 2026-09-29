# Computer Vision — Chapter: Binary Vision
*Notes compiled from lecture slides + transcripts (Binary, Cleaning Binary Images)*

---

## 0. What is binary vision, and why bother?

**Binary image processing** = taking an image (usually greyscale) and converting it to **black and white**, so every pixel is labelled either:
- **1 — object of interest** (foreground), or
- **0 — not of interest** (background).

Lecture examples:
- **Licence plate:** crop the plate → greyscale → binary. The characters come out black and the plate white, so each black region is a candidate character. Grouping the pixels of each character into a "region" is region processing, which comes later in the course.
- **Road signs:** first label each pixel "red / not red". Then, *inside* the red regions, label pixels "black / not black". This combination picks out the sign's contents.

**When does it work?** Only in **constrained** situations, where objects of interest can be reliably separated from the background. Notice both examples are man-made objects (signs, plates), even though they're photographed in the real world.

**Why it was historically so important:** much early (especially **industrial**) vision was done entirely with binary images, because it is **very fast**: no colour, and every decision is a simple yes/no. That mattered on old hardware. Today we're less constrained, but the ideas are still used everywhere as a building block.

> **Big idea to keep in mind for the whole chapter:** binary vision throws away almost all the information in the image and keeps a single yes/no per pixel. That's great when foreground vs background really *is* a yes/no question, and terrible when it isn't.

Chapter structure:
1. **Thresholding**: greyscale → binary
2. **Threshold detection**: choosing the threshold automatically (Otsu)
3. **Variations**: adaptive thresholding, multi-spectral thresholding
4. **Cleaning binary images**: mathematical morphology (erosion, dilation, opening, closing), plus greyscale/colour morphology and finding local maxima

---

## 1. Thresholding

### 1.1 Binary thresholding
For every pixel, compare its grey value $f(i,j)$ against a threshold $T$:

$$
g(i,j) = \begin{cases} 1 & \text{if } f(i,j) \geq T \\ 0 & \text{if } f(i,j) < T \end{cases}
$$

- $f(i,j)$ is the input greyscale image and $g(i,j)$ the output binary image.
- In practice the "1" is usually stored as **255**, so the binary image is visible when displayed (a pixel value of 1 out of 255 would look black).
- Which side counts as "object" depends on the scene. For dark text on a light plate, the object is the *below-threshold* part. OpenCV has an inverted mode (`THRESH_BINARY_INV`) for that.

⚠️ **Only works for simple scenes**: the foreground and background must be **distinct** in grey level. The lecturer stresses this repeatedly: check that thresholding is appropriate *before* you use it.

### 1.2 Look-Up Tables (LUTs), a speed trick
Rather than doing a comparison for every pixel, precompute the answer for every *possible* grey level once:

$$
\text{LUT}(k) = \begin{cases} 1 & k \geq T \\ 0 & k < T \end{cases} \quad \text{for } k = 0 \ldots 255
$$

then, for every pixel, $g(i,j) = \text{LUT}(f(i,j))$.

> **Why this helps:** the LUT has only 256 entries, but an image might have millions of pixels. Each pixel now needs one array lookup instead of a comparison and branch. It mattered a lot in early vision. Today it matters less, and efficiency concerns have moved to things like video, very large images, and multi-scale (pyramid) processing.

### 1.3 OpenCV
```cpp
threshold(gray_image, binary_image, threshold, 255, THRESH_BINARY);
```
- Input must be a **single-channel** (greyscale) image.
- `threshold` is $T$; `255` is the output value for pixels **≥ T** (in fact OpenCV uses strictly **> T**, a detail that rarely matters).
- `THRESH_BINARY` selects the operation. There are other modes (inverted, truncate, to-zero…).

### 1.4 Why choosing T is hard
The lecture video sweeps the threshold on a **PCB (printed circuit board)** image:
- **T too low** → too many pixels are "on", and neighbouring tracks merge together.
- **T too high** → too few pixels, and tracks break apart (false gaps in connectivity).
- Very high → almost nothing left.

On a hard image there may be **no single perfect T**. That leads to the next question: how do we pick T?

---

## 2. Threshold Detection

### 2.1 Manual setting (and why it fails)
You could just drag a slider until the result looks good. The problem is that in a real system **lighting changes**:
- Different times of day, different environments.
- Even in a sealed industrial inspection box with controlled lighting, **bulbs degrade over time**, so a fixed threshold slowly drifts out and the system eventually fails.

➡️ We need to determine T **automatically** for each image.

### 2.2 Histograms: the tool for this
Notation used for the techniques that follow:
- **Image:** $f(i,j)$
- **Histogram:** $h(g)$ = number of pixels with grey value $g$, for $g = 0 \ldots 255$
- **Normalised probability distribution:**
$$ p(g) = \frac{h(g)}{\sum_g h(g)} $$
i.e. divide each count by the total number of pixels, so the values sum to 1. $p(g)$ is "the fraction of the image that has grey value $g$".

> **Intuition:** if the image has dark objects on a light background, the histogram typically has **two humps** (it's *bimodal*): one for the object pixels and one for the background. A good threshold sits in the **valley between the humps**. Otsu's method is a principled way of finding that valley.

### 2.3 Otsu thresholding (the method OpenCV uses)
**Goal:** split the pixels into two classes, **foreground (F)** and **background (B)**, so that each class is as **tight (low spread)** as possible.

**Within-class variance** for a threshold $T$:
$$ \sigma_W^2(T) = w_f(T)\,\sigma_f^2(T) + w_b(T)\,\sigma_b^2(T) $$

- **Weights** give the fraction of pixels in each class:
$$ w_f(T) = \sum_{g=T}^{255} p(g) \qquad w_b(T) = \sum_{g=0}^{T-1} p(g) $$
- **Class means** give the average grey value of each class:
$$ \mu_f(T) = \frac{\sum_{g=T}^{255} g\,p(g)}{w_f(T)} \qquad \mu_b(T) = \frac{\sum_{g=0}^{T-1} g\,p(g)}{w_b(T)} $$
- **Class variances** give the spread of each class around its own mean:
$$ \sigma_f^2(T) = \frac{\sum_{g=T}^{255} (g-\mu_f)^2\,p(g)}{w_f(T)} \qquad \sigma_b^2(T) = \frac{\sum_{g=0}^{T-1} (g-\mu_b)^2\,p(g)}{w_b(T)} $$

*(The formula slide is an image in the PDF, so these are the standard Otsu definitions, written to match the lecturer's description. Check them against slide 6.)*

**Algorithm:**
1. Compute the histogram and $p(g)$ **once**.
2. For **every** possible $T$ (only 256 of them), compute $\sigma_W^2(T)$.
3. Pick the $T$ with the **smallest within-class variance**.

This is very fast: you never go back to the image, just do 256 cheap calculations over the histogram.

**Between-class variance (the implementation trick):**
Minimising within-class variance is **equivalent** to maximising the **between-class variance**:
$$ \sigma_B^2(T) = w_f(T)\,w_b(T)\,\big(\mu_f(T) - \mu_b(T)\big)^2 $$
This version is cheaper because it needs no per-class variance sums, so implementations normally maximise it instead.

> **Why are they equivalent?** Total variance of the image = within-class + between-class, and total variance doesn't depend on $T$. So making one smaller necessarily makes the other bigger. Intuitively, you want each class tight (small within) and the two class means far apart (large between).

> **Tip for understanding:** the lecturer admits the maths "needs sitting down with". Draw a histogram, pick a T, shade the left part (background) and right part (foreground), then ask: what is each side's weight (area), mean (centre), and spread? That is literally all the formulas compute.

**OpenCV:** add the `THRESH_OTSU` flag. The `threshold` argument you pass is then **ignored** and Otsu picks it (the function returns the chosen value).
```cpp
threshold(gray_image, binary_image, threshold, 255, THRESH_BINARY | THRESH_OTSU);
```

On a PCB image, Otsu gives a very clean result. The remaining imperfections come mainly from **poor lighting**. In industrial vision, **getting the lighting right** often makes the image processing much simpler.

---

## 3. Variations on Thresholding

### 3.1 Adaptive thresholding: the motivation
A **single global threshold** fails when illumination varies across the image. Example: a scanned apartment floor plan that is **dark in the bottom-right and bright in the top-left**. Any one T will wipe out one side or the other.

Second example, a **train station** (converted to greyscale): global Otsu loses the yellow sign and the big platform "2" sign.

➡️ Idea: use **different thresholds in different parts of the image**.

### 3.2 Classic adaptive thresholding (sub-images + interpolation)
1. **Divide** the image into sub-images (the lecture uses **8 × 8 = 64** blocks).
2. **Compute a threshold for each sub-image**, e.g. using Otsu on that block's histogram.
3. **Interpolate** the thresholds for every pixel using **bilinear interpolation**, based on the pixel's distance to the centres of the neighbouring blocks. This way the threshold changes smoothly instead of jumping at block boundaries.

> **What's bilinear interpolation?** For a pixel lying between four block centres, take a weighted average of those four thresholds, weighting closer centres more heavily. First interpolate horizontally, then vertically (hence "bi-linear").

**Failure mode:** blocks that are **essentially uniform** (e.g. blank paper at the top of the floor plan). Otsu *always* returns some threshold, even when the block has nothing to separate, so it ends up splitting noise or a gentle gradient, which produces garbage. **Hint for detection:** such a block's threshold is very different from its neighbours'.

### 3.3 OpenCV's adaptive thresholding (local mean)
OpenCV's version works differently: it compares **each pixel to the mean of its own neighbourhood**:

$$
g(i,j) = \begin{cases} 255 & \text{if } \left( f(i,j) - \dfrac{\sum_{a=-m}^{m}\sum_{b=-m}^{m} f(i+a,\,j+b)}{(2m+1)^2} \right) > \text{offset} \\[8pt] 0 & \text{otherwise} \end{cases}
$$

- The neighbourhood is a $(2m+1)\times(2m+1)$ square centred on the pixel. In the lecture $m = 40$, giving an **81 × 81** block.
- The **offset** (5 in the lecture) avoids reacting to **noise**. In a flat region every pixel is roughly equal to its local mean, so comparing against 0 would flip pixels randomly with tiny noise fluctuations.
- In words: a pixel is foreground if it is noticeably brighter than its surroundings.

```cpp
adaptiveThreshold(gray_image, binary_image, output_value,
                  ADAPTIVE_THRESH_MEAN_C, THRESH_BINARY,
                  block_size, offset);   // lecture: block_size = 81, offset = 5
```
> ⚠️ Sign convention: OpenCV's actual formula is `f(i,j) > mean − C`, i.e. it *subtracts* the offset from the mean. So the sign of the offset you pass is flipped relative to the slide's formula. Worth checking the docs when you use it in labs.

**Result on the train station:** much better. The "2" is clear and the yellow sign's text starts to appear. The same weakness remains, though: a **region of constant colour** (to the left of the "2") becomes a white block, because there's no real structure there to threshold.

**Tuning:** the block size must suit the scale of the things you're looking for. Too small and it can't "see" the background around an object; too large and it behaves like a global threshold.

### 3.4 Multi-spectral / multi-level thresholding
Using **several thresholds** to split an image into **more than two classes**. Example from an MRI brain scan (Univ. of North Carolina, published work): separating **white matter, grey matter and subcortical structures** using thresholds found in the greyscale histogram, then displaying them as red/green/blue.

**The lecturer's critique (important!):** the three class distributions in the histogram **overlap substantially**. Every pixel in the overlapping tails is **misclassified**, and there are a lot of them. The lecturer finds it worrying that clinical decisions could rest on this.

> **Lesson:** this goes back to the chapter's golden rule. **Only threshold when the classes are genuinely distinct in the feature you're thresholding.** If the histograms overlap, no threshold can separate them, and a more powerful technique is needed.

---

## 4. Cleaning Binary Images: Mathematical Morphology

### 4.1 Why we need special cleaning operations
Binary images straight out of thresholding are rarely clean:
- **Isolated noise points**: single "on" pixels with nothing around them.
- **Rough, jagged region boundaries**, which make later shape analysis hard.
- **Holes** inside objects and **breaks** in objects that should be connected.

We **can't use normal smoothing** (averaging, Gaussian, etc. from the Images chapter). Averaging 0s and 255s gives values like 113, and then the image is no longer binary. Instead we reason about **shape**.

**Mathematical morphology** treats a binary image as a **set** of foreground pixel coordinates and defines operations on those sets. It's a large field in its own right. This course covers only the four most-used operations: **dilation, erosion, opening, closing**.

### 4.2 The structuring element
A **structuring element** $B$ is a small shape (typically a **3×3** square) with a designated **origin**, usually its centre. It defines "which neighbours count" when we process each pixel. Think of it as the binary analogue of a filter kernel.

- **Isotropic** structuring element: a solid, "continuous" shape with no gaps, like a full 3×3 or 5×5 square (or a disc). It behaves roughly the same in all directions.
- Larger elements (5×5, 7×7…) give a stronger effect (more smoothing, bigger holes filled, bigger noise removed).

### 4.3 Dilation (⊕): grow regions
**Minkowski set addition:**
$$ X \oplus B = \{\, p \in \mathbb{Z}^2 \;:\; p = x + b,\; x \in X,\; b \in B \,\} $$

- $X$ = the set of foreground pixels in the original image, $x$ = one of those pixels, $B$ = the structuring element.
- **In words:** place a copy of $B$ on every foreground pixel. Every pixel covered by any copy is foreground in the output.
- With a 3×3 $B$, each single pixel becomes a 3×3 blob, so every region **grows by one pixel** all around its border.

**Effects:**
- Makes regions **bigger**.
- **Fills small holes.**
- **Joins close regions** (bridges small gaps).

**PCB application:** dilate the tracks. If two tracks nearly touch after dilation, they were laid down **too close together**, a potential short circuit. Dilation can also reconnect regions that thresholding accidentally broke apart.

### 4.4 Erosion (⊖): shrink regions
**Minkowski set subtraction:**
$$ X \ominus B = \{\, p \in \mathbb{Z}^2 \;:\; p + b \in X \;\text{ for every } b \in B \,\} $$

- **In words:** place $B$ at pixel $p$. Pixel $p$ survives **only if every pixel under $B$ is foreground**. If any neighbour (per $B$) is background, $p$ is removed.
- The effect is that border pixels are stripped away. The lecturer notes this is the *effect*, but the formal definition is the "does B fit entirely inside the object?" test.

**Effects:**
- Makes regions **smaller**.
- **Removes noise**: isolated points and tiny blobs vanish completely.
- **Removes narrow bridges**: thin connections between two regions are cut.

**PCB application:** erode the tracks. If a track **breaks**, it was too thin at that point and might not carry the signal properly, which flags a manufacturing defect.

> ⚠️ **Erosion and dilation are NOT inverses.** If erosion deletes a small feature entirely (like the little L-shape in the slide example), dilation has nothing left to grow back from. Information is lost.

> **Handy duality to remember:** eroding the foreground is the same as dilating the background (and vice versa).

### 4.5 Opening (∘): erosion then dilation
$$ X \circ D = (X \ominus D) \oplus D $$

**Motivation:** erosion cleans up but shrinks objects. Dilating afterwards restores the size.
- **Removes noise** and **removes narrow bridges** (erosion's benefits, which the dilation can't undo because those bits are gone).
- **Roughly maintains region size.**
- **Smooths the shape**: sharp protrusions and jagged outer edges are trimmed.

Mnemonic: opening **opens up gaps**, breaking thin connections and deleting specks.

### 4.6 Closing (•): dilation then erosion
$$ X \bullet D = (X \oplus D) \ominus D $$

- **Fills small holes** and **joins close regions** (dilation's benefits). Once a hole or gap is filled, the following erosion doesn't reopen it.
- **Roughly maintains region size.**

Mnemonic: closing **closes holes and gaps**.

> **Quick comparison:**
>
> | Operation | Order | Removes | Fills / joins | Size |
> |---|---|---|---|---|
> | Dilation | ⊕ | — | holes, gaps | grows |
> | Erosion | ⊖ | noise, bridges | — | shrinks |
> | Opening | ⊖ then ⊕ | noise, bridges | — | ≈ kept |
> | Closing | ⊕ then ⊖ | — | holes, gaps | ≈ kept |

### 4.7 Properties
- An isotropic structuring element **eliminates small image details** (anything smaller than the element).
- **Idempotence:** applying opening (or closing) a second time with the *same* structuring element changes nothing:
$$ X \circ D = (X \circ D) \circ D \qquad X \bullet D = (X \bullet D) \bullet D $$
  This is **not** true for plain erosion or dilation (each repeat keeps shrinking or growing). It also doesn't hold if you use a different-sized structuring element the second time.

### 4.8 Worked application: people leaving a library
A realistic pipeline that combines everything so far:
1. Build a **background model**: what the scene looks like with nobody in it (from video).
2. **Subtract** the background from the current frame to get a **difference image**.
3. **Threshold** the difference, giving a binary image of "things that changed".
4. The raw binary is messy: people have holes and breaks where their clothes match the background (a trouser leg the same grey as the stair handrail, a person passing a display cabinet of similar tone).
5. **Closing** fills those breaks. It looks a bit blocky, but the people become solid.
6. **Opening** removes remaining noise, e.g. a thin vertical stripe from a **door slowly closing**.
7. The result is two clean person-shaped blobs of roughly the right size. Shape analysis (tall and narrow) could then confirm they're people.

The same close-then-open approach finds **breaks in PCB tracks**.

> **Takeaway:** in practice you **chain** these operations, choosing the order and structuring-element size to match the kind of mess in your image.

### 4.9 OpenCV code
```cpp
// Dilation with the default 3x3 isotropic structuring element (pass an empty Mat)
dilate(binary_image, dilated_image, Mat());

// Dilation with an explicit 5x5 isotropic structuring element (all ones)
Mat structuring_element(5, 5, CV_8U, Scalar(1));
dilate(binary_image, dilated_image, structuring_element);

// Erosion: identical usage, just call erode
erode(binary_image, eroded_image, Mat());
erode(binary_image, eroded_image, structuring_element);

// Opening and closing
Mat five_by_five_element(5, 5, CV_8U, Scalar(1));
morphologyEx(binary_image, opened_image, MORPH_OPEN,  five_by_five_element);
morphologyEx(binary_image, closed_image, MORPH_CLOSE, five_by_five_element);
```
*(`getStructuringElement(MORPH_ELLIPSE, Size(5,5))` is a handy way to make non-square shapes such as discs.)*

---

## 5. Morphology Beyond Binary

### 5.1 Greyscale and colour morphology
Morphology isn't only for binary images. To extend it to greyscale:
- For **every grey level $g$**, form a set containing **all pixels with value ≥ $g$**. So a pixel of value 255 belongs to every set, while a pixel of value 3 belongs only to the sets for 1, 2 and 3.
- Apply erosion/dilation to **each set separately**, then reassemble the image.

> **The practical shortcut (equivalent result):** greyscale **dilation = local maximum** over the structuring element, and greyscale **erosion = local minimum**. This is how you'd actually compute it, and it's a much easier way to picture it.

- **Colour:** do the same **per level per channel**, e.g. an RGB image has 255 × 3 sets.
- **Effect:** dilation spreads bright areas outward, erosion spreads dark areas. It gives interesting "painterly" effects and can make further processing easier. The lecturer thinks it's under-used.

### 5.2 Finding local maxima (very useful later!)
In recognition, we often compute a **"degree of fit" at every pixel** (how well does the image around here match what I'm looking for?). We then want the **peaks**: the local maxima.

**Morphological recipe:**
1. **Dilate** the image. Each pixel becomes the maximum of its neighbourhood.
2. **Compare** the dilated image to the original. Where they're **equal**, the pixel *was* the maximum of its neighbourhood, so it's a local maximum.
3. **Threshold** the original, to keep only points that are high enough to matter.
4. **Logical AND** the two results, giving only the **"high" local maxima**. Small bumps below the threshold are rejected.

(Local minima work the same way using erosion.)

```cpp
Mat dilated, thresholded_input, local_maxima, thresholded_8bit;
dilate(input, dilated, Mat());
compare(input, dilated, local_maxima, CMP_EQ);                        // equal => local max
threshold(input, thresholded_input, threshold, 255, THRESH_BINARY);  // high enough?
thresholded_input.convertTo(thresholded_8bit, CV_8U);                // match types for the AND
bitwise_and(local_maxima, thresholded_8bit, local_maxima);            // both conditions
```

> **Why this matters:** this works on *any* 2D (or n-D) array of values, not just photos: match scores, heat maps, accumulator arrays. You'll likely see it again for detecting features and matching templates.

---

## Quick recap: key things to remember
- **Thresholding** turns greyscale into binary: $g = 1$ if $f \geq T$. It is fast and simple, but **only valid when foreground and background are genuinely distinct**. LUTs make it faster.
- **Manual thresholds break** as lighting changes, so detect T automatically from the **histogram**.
- **Otsu**: try all 256 thresholds and pick the one with the **minimum within-class variance**, equivalently the **maximum between-class variance**. In OpenCV this is `THRESH_OTSU`.
- **Adaptive thresholding** handles uneven lighting:
  - Classic: per-block Otsu plus bilinear interpolation.
  - OpenCV: compare each pixel to its local mean minus an offset (`adaptiveThreshold`, e.g. block size 81, offset 5).
  - Both fail in **uniform regions**.
- **Multi-level thresholding** with overlapping class histograms guarantees misclassification (the MRI example).
- You can't smooth a binary image the normal way, so use **morphology**:
  - **Dilation** grows regions, fills holes, joins regions.
  - **Erosion** shrinks regions, removes noise, cuts bridges.
  - They are **not inverses**.
  - **Opening** (erode→dilate) removes noise and bridges while keeping size.
  - **Closing** (dilate→erode) fills holes and gaps while keeping size.
  - Opening and closing are **idempotent**.
- Morphology extends to **greyscale/colour** (dilate = local max, erode = local min) and gives a neat way to find **local maxima** (dilate, compare-equal, threshold, AND).

## Extra material (links from the module folder)
- Thresholding (HIPR2): https://homepages.inf.ed.ac.uk/rbf/HIPR2/threshld.htm
- Adaptive thresholding (HIPR2): https://homepages.inf.ed.ac.uk/rbf/HIPR2/adpthrsh.htm
- Morphology (HIPR2): https://homepages.inf.ed.ac.uk/rbf/HIPR2/morops.htm
- Thresholding in OpenCV: https://docs.opencv.org/4.4.0/d7/dd0/tutorial_js_thresholding.html
- Morphological operations in OpenCV: https://docs.opencv.org/4.4.0/d4/d76/tutorial_js_morphological_ops.html
