# Computer Vision: Region Segmentation
*Notes based on the lecture slides (*Region Segmentation*, from Dawson-Howe, *A Practical Introduction to Computer Vision with OpenCV*, 2014) and three transcripts: Connectivity & Connected Component Analysis, Region Features, and kMeans Clustering.*

## ✅ 必背清单
> 只列考试相关内容，都来自课件。🔍 框是帮助理解的，不用背。📝 框 = 往年试卷怎么考这部分。
> **考试没有计算题**（2021–2025 五份试卷都没有），只有应用题、比较题、解释题，而且**不能写代码或 OpenCV 函数**，只能讲理论。所以公式理解即可，重点是每个技术的**输入、输出、参数、问题**，以及技术之间的**异同**。

1. **分割 (segmentation)** 分两大类：基于区域 (region-based) 和基于边缘 (edge-based)。实际按"颜色/灰度一致的区域"切，不是直接按物体切。
2. **连通性悖论 (connectivity paradox)：** 只用 4 邻接或只用 8 邻接都会出矛盾，所以交替使用：背景 4、物体 8、孔洞 4……
3. **CCA：** 二值图 → 标签图。两遍扫描：第一遍记录等价 (equivalences)，第二遍合并。它是阈值化之后的标准下一步。问题：相互接触的物体会被合成一个区域。
4. **区域特征 (region features)：** 面积、MBR（长宽比、矩形度）、细长度、凸包、矩、凹陷和孔洞、周长和圆形度。作用是把每个区域变成一组数字，用来**筛选和识别**；比值类特征与大小无关。
5. **矩：** 只记不变性阶梯：原始矩 → 中心矩（平移不变）→ η（缩放不变）→ Hu（旋转也不变）。
6. **k-means：** 在颜色空间聚类；要事先给 k；**不用位置信息**；随机初始化，每次结果可能不同；用 DB 指数选 k。
7. **Watershed：** 把图像当地形，从最低点注水，水域相遇处就是边界；用位置信息；缺点是过分割，用 markers 解决。
8. **Mean shift：** 每个像素往"密度更高"的地方移动，停在同一处的归为一类；用空间核 + 颜色核；不需要 k；缺点是带宽难选、慢。

### 📝 往年试卷里的这一章
| 试卷 | 题目 | 用到这一章的什么 |
|---|---|---|
| 2022 Q2(b) 比较题 | CCA vs Watershed vs Mean shift | 整道题都是这一章（见 §7 比较题素材） |
| 2023 Q1(b) 比较题 | Canny vs CCA vs Mean shift | CCA、Mean shift（Canny 在 Edges 那一章） |
| 2023 Q3(b) 比较题 | Hough 找圆 vs Chamfer 找圆 vs Mean shift + 圆形度 | Mean shift、圆形度 |
| 很多应用题 | 找出口标志、自行车标志、蓝色路牌、车牌等 | 分割 → CCA → 用区域特征筛选候选区域 |

**应用题里这一章的标准套路：** 原图 →（阈值化，或用 k-means / mean shift 做颜色分割）→ 二值图 → **CCA** → 一个个候选区域 → 算**区域特征** → 按特征范围去掉不对的区域 → 识别。每一步都要写清楚**输入、输出、参数**。

**目录 (Topics)**
0. What segmentation is
1. Connectivity: 4- and 8-adjacency and the paradox
2. Connected Components Analysis (CCA)
3. Region features: area, bounding rectangle, elongatedness, convex hull, moments, concavities and holes, perimeter and circularity
4. k-means clustering
5. *Watershed segmentation (slides only; lectured later)*
6. *Mean shift segmentation (slides only; lectured later)*
7. Summary, exam tips and practice questions

---

## 0. Segmentation: the big picture

**Segmentation** splits an image into smaller parts, preferably corresponding to (parts of) **objects**.

- **Two main approaches:**
  - **Region-based:** find areas that are **homogeneous** or self-similar (consistent colour, for example).
  - **Edge-based:** find the **boundaries** between regions, where the image changes.
  - These are two views of the same thing (a region is enclosed by edges), but the techniques are usually quite different.
- **Why not segment straight into objects?** You would need to know what the objects are first. A person's T-shirt and trousers are different colours, so without knowing it's a person, nothing says they belong together. The same goes for a plaid shirt with many colours. So in practice we segment into **homogeneous regions**, not objects.
- **Video segmentation** has two meanings:
  1. Breaking a video into **clips** (shots).
  2. Segmenting objects or regions **consistently over time**, despite rotation, deformation, or taking off a jacket. This is hard and **not covered** in the course; video is handled frame by frame plus **tracking**.

> **中文解释：** 分割 = 把图像切成更小的部分。理想是按"物体"切，但做不到（不知道物体是什么，就无法知道 T 恤和裤子属于同一个人），所以实际上是按**颜色/灰度一致的区域**来切。两大类方法：**基于区域**（找内部相似的区域）和**基于边缘**（找区域之间的边界）。

---

## 1. Connectivity (binary images)

### 1.1 The problem
To group pixels into regions, we must decide **which pixels are neighbours**. This seems trivial, but it leads to a **paradox**.

Picture a ring of object pixels drawn diagonally, with gaps only at corners:
- The ring looks **continuous**, since the pixels touch diagonally.
- But the **background inside and outside** also touches diagonally through the same corners.
- **Both can't be connected.** If the ring is closed, the inside and outside must be separate regions.

### 1.2 4-adjacency vs 8-adjacency
Label the 3×3 neighbourhood with the centre pixel **8** and its neighbours **0–7** going around it (0, 2, 4, 6 = the edge-sharing neighbours; 1, 3, 5, 7 = the diagonals):

```
 1 2 3            . X .              X X X
 0 8 4    4-adj:  X 8 X     8-adj:   X 8 X
 7 6 5            . X .              X X X
```

| | Neighbours of the centre | Effect on the ring example |
|---|---|---|
| **4-adjacency** | Up, down, left, right only (**4**) | The ring breaks into **many small regions**, which is too many to analyse. The circle is no longer visible. |
| **8-adjacency** | All 8, including diagonals | The ring is **one region**, but the **background leaks through the diagonals**, so inside and outside are joined. That's the paradox. |

### 1.3 The practical solution: alternate
Assume the outermost area is **background**, then alternate the adjacency type at each level of nesting:

| Level | Adjacency |
|---|---|
| **Background** (outside) | **4-adjacency** |
| **Objects** in the background | **8-adjacency** |
| **Holes** in objects | **4-adjacency** |
| **Objects in holes** | **8-adjacency** |
| … | keep alternating |

- This is **arbitrary** and sometimes makes the wrong connection. But computer vision aims for **a single interpretation** of a scene, and this guarantees one.
- It only applies to **binary images** (object vs background).

> **中文解释：** **4 邻接**只看上下左右，**8 邻接**还看对角线。只用一种会有悖论：用 4 邻接，圆环会碎成很多块；用 8 邻接，圆环连起来了，但背景也通过对角线"漏"过去了，内外背景变成同一块。**解决办法：交替使用**，背景用 4 邻接，物体用 8 邻接，洞用 4 邻接，洞里的物体用 8 邻接……虽然有点武断，但能保证场景只有**唯一一种解释**。

---

## 2. Connected Components Analysis (CCA)

### 2.1 Why
After **thresholding**, a binary image is just black and white dots. To analyse it, you need to **join the dots**: give every pixel of a contiguous region the **same label**. This is almost always the step after thresholding:

**greyscale → (choose a threshold, e.g. Otsu) → threshold → CCA → region features → recognition**

The output is a **label image**: the values are **region IDs**, not intensities.

### 2.2 Algorithm (two passes)
```
Pass 1: search the image row by row, left to right
  for each non-zero (object) pixel:
      look at its PREVIOUS (already visited) neighbours
          8-adjacency: left, upper-left, up, upper-right
          4-adjacency: left, up
      if all previous neighbours are background:
          assign a NEW label
      else:
          pick any label from the previous neighbours
          if other previous neighbours have a DIFFERENT label:
              note the EQUIVALENCE (e.g. red ≡ brown)
Pass 2: relabel every pixel, replacing each label with its equivalence class
```

**Lecture example: the letter "W".** Scanning the top row starts **three labels** (red, brown, yellow), one per arm of the W. Lower down, a pixel has red on its left and brown at its upper-right, so **red ≡ brown** is noted. Later **yellow ≡ red** is noted. Pass 2 merges them all into **one region**.

### 2.3 My own worked example (8-adjacency)
```
     col: 0 1 2 3
row 0:    1 0 0 1
row 1:    1 0 0 1
row 2:    1 1 1 1          (a "U" shape)
```
| Pixel | Previous neighbours (L, UL, U, UR) | Action |
|---|---|---|
| (0,0) | none | new label **A** |
| (0,3) | none | new label **B** |
| (1,0) | U = A | A |
| (1,3) | U = B | B |
| (2,0) | U = A | A |
| (2,1) | L = A, UL = A | A |
| (2,2) | L = A, UR = (1,3) = **B** | A, note **A ≡ B** |
| (2,3) | L = A, U = B | A (A ≡ B already noted) |

Pass 2 relabels B → A, giving **one region**. Without the equivalence step, the U would be counted as two objects.

> **中文解释：** 连通域分析 (CCA) = 给二值图中**连在一起的像素贴上同一个标签**。逐行扫描，每个前景像素只看"之前已经扫描过"的邻居：都没有标签就新建标签；有标签就沿用；如果邻居有**两个不同标签**，就记下"等价"。最后第二遍扫描，把等价的标签合并。这是阈值化之后几乎**必做的一步**。

### 2.4 Things to watch
- **Touching characters merge.** In the slide's letters example, the Ws are connected (even with 4-adjacency), and the Bs and As touch diagonally so they merge with 8-adjacency. The Os and Js stay separate.
- To label the **background** regions instead, **invert the image** (or tweak the algorithm).

> 📝 **应用题里怎么写 CCA**
> 1. **输入：** 二值图（前一步阈值化或颜色分割的结果）。
> 2. **输出：** 标签图，每个连通区域一个编号，也就是一组候选区域 (candidate regions)。
> 3. **参数：** 物体用 8 邻接（背景 4 邻接，见 §1.3）。
> 4. **在题目里的作用：** 把"一堆前景像素"变成"一个个可以单独测量的区域"，下一步才能对每个区域算特征。
> 5. **要主动指出的问题：** 相互接触的物体会合成一个区域（比如车牌上粘在一起的字符），可以先做开运算 (opening) 把它们分开（Binary Vision 那一章）；噪声会产生很多很小的区域，可以按面积去掉。

### 2.5 OpenCV
OpenCV does CCA via **`findContours`**. This is **confusing terminology**: in vision, "contour" means an **edge or boundary**, but OpenCV represents each **region** by the contour around it, which is more efficient.

```cpp
vector<vector<Point>> contours;
vector<Vec4i> hierarchy;
findContours(binary_image, contours, hierarchy,
             CV_RETR_TREE, CV_CHAIN_APPROX_NONE);   // TREE = full hierarchy (regions → holes → regions in holes)

// draw each region filled, in a random colour
for (int contour = 0; contour < contours.size(); contour++) {
    Scalar colour(rand()&0xFF, rand()&0xFF, rand()&0xFF);
    drawContours(contours_image, contours, contour, colour, CV_FILLED, 8, hierarchy);
}
```
- `RETR_TREE` returns the **hierarchy**: objects, holes inside them, objects inside holes, and so on. It matches the alternating idea in §1.3.
- `CHAIN_APPROX_NONE` keeps **every** boundary point, with no compression.
- Python: `contours, hierarchy = cv2.findContours(binary, cv2.RETR_TREE, cv2.CHAIN_APPROX_NONE)`. Modern OpenCV also has `cv2.connectedComponentsWithStats(binary)`, which returns a true label image plus the area, bounding box and centroid of each region.

---

## 3. Region Features

Once a region has been segmented, describe it with **features**, so that you can **recognise or classify** it later (for example with a **linear classifier**, which is also the last stage of a deep network).

### 3.1 Area
- **Area = number of pixels** in the region.
- It is **too simplistic** on its own, because it depends on the **distance to the camera** (scale). It is useful when **normalised** or combined with other features.

### 3.2 Minimum Bounding Rectangle (MBR，最小外接矩形)
The **tightest-fitting rectangle** around the region, at any orientation.

**How:** rotate the rectangle through **discrete angle steps** (for example 1°), fit it tightly at each angle, and keep the smallest. You only need to search **one quadrant (0–90°)**, because a rectangle rotated by 90° is the same rectangle.

| Metric | Formula | Meaning |
|---|---|---|
| **Length-to-width ratio** | Length / Width | Rough shape (square ≈ 1) |
| **Rectangularity** | Area / (Length × Width) | **1** = a perfect rectangle; close to 0 for thin shapes such as an "X" |
| **Convex hull / MBR ratio** | Area inside convex hull / (Length × Width) | How much of the rectangle the hull fills |

Sanity check: a circle's rectangularity is πr² / (2r)² = π/4 ≈ **0.79**, which matches the slide's table.

```cpp
RotatedRect min_bounding_rectangle = minAreaRect(contours[contour_number]);
```

> **中文解释：** MBR（最小外接矩形）= **转着找一个面积最小、能刚好框住区域的矩形**。用它的长和宽可以算出三个形状特征：长宽比（大概形状）、矩形度（有多像矩形）、凸包比（填平凹陷后占多少）。

#### 中文详解：MBR 在做什么、怎么找

**1. 为什么要"任意角度"？**
水平竖直的框遇到**倾斜的物体**就不准。比如一根斜放 45° 的铅笔：水平框接近正方形，里面大部分是空白，长宽比 ≈ 1，好像是个"方块"。把框也转 45°，就能紧贴铅笔，得到又长又窄的矩形，这才反映真实形状。

**2. 第一步：0° 的框（水平竖直）**
遍历区域的所有像素（CCA 标签相同的像素），记下四个值：
- `min_x`（最左）、`max_x`（最右）、`min_y`（最上）、`max_y`（最下）
- 宽 = `max_x − min_x + 1`，高 = `max_y − min_y + 1`（+1 因为每个像素本身占一格）

四条边都刚好碰到区域最外面的像素，所以这是 0° 方向上最紧的框。

**3. 其他角度 θ：先转坐标轴，再找最小/最大值**
对每个像素 (x, y) 计算它在旋转后坐标轴上的坐标：
- u = x·cosθ + y·sinθ
- v = −x·sinθ + y·cosθ

然后：长 = `max_u − min_u`，宽 = `max_v − min_v`，面积 = 长 × 宽。
可以理解为"沿一个斜的方向量尺子"，**每个角度都是在那个方向上找最远的两个点**。

**4. 小例子：45° 斜线**，像素 (0,0)、(1,1)、(2,2)、(3,3)
- **θ = 0°：** x 和 y 都是 0~3，框为 4 × 4，面积 **16**，大部分是空的。
- **θ = 45°**（cos = sin ≈ 0.707）：

| 像素 | u = 0.707(x+y) | v = 0.707(y−x) |
|---|---|---|
| (0,0) | 0 | 0 |
| (1,1) | 1.41 | 0 |
| (2,2) | 2.83 | 0 |
| (3,3) | 4.24 | 0 |

长 ≈ 4.24，宽 ≈ 0（实际约一个像素厚，≈ 1），面积约 4.24，**远小于 16**。所以 MBR 在 45° 附近。

**5. 完整流程**
```
best_area = 无穷大
for θ = 0°, 1°, 2°, ..., 89°:      # 只需 0~90°：转 90° 只是长宽互换，是同一个矩形
    对区域每个像素算 u, v
    长 = max_u − min_u,  宽 = max_v − min_v
    if 长 × 宽 < best_area:
        best_area = 长 × 宽，记下 θ、长、宽
```
最后记下的就是 MBR。

**6. 三个特征的直观理解**
- **长宽比：** 正方形 ≈ 1，细长形状数值大。只是粗略描述，**不能当作细长度**（见 3.3，弯曲的细长形状外接矩形可能接近正方形）。
- **矩形度 = 区域面积 / (长 × 宽)：** 区域占外接矩形的比例。矩形 = 1，圆 = π/4 ≈ 0.79（四个角空了），"X" 接近 0（两条细线只占很小一部分）。
- **凸包 / MBR 比：** 凸包 = 用橡皮筋套住区域，把凹进去的部分填平（见 3.4）。"X" 的矩形度很低，但凸包比较高；两者一对比，就能看出形状有多少凹陷。

**7. 小技巧 & OpenCV**
- 只用区域的**边界像素（轮廓）**来算就够了，因为最远的点一定在边界上。所以 `minAreaRect` 的输入是 contour。
- `minAreaRect` 返回 `RotatedRect`：**中心点、长宽 (size)、旋转角度 (angle)**。
- OpenCV 实际用更快的 rotating calipers（旋转卡壳）算法，不是逐度尝试，但原理一样。考试按"逐个角度尝试"来解释即可。

### 3.3 Elongatedness
- It **CANNOT be** the length/width ratio of the MBR. A thin, bent shape (like the green "[" on slide 11) can have a squarish MBR but be very elongated.
- **Definition:** region area ÷ (thickness)².

$$\text{elongatedness} = \frac{\text{area}}{(2d)^2}$$

- **d = the number of erosions needed to make the region disappear completely.**
  - Erosion eats away from **both sides**, so the thickness is ≈ **2d**.
- **Sanity checks (my own):**
  - **Square** of side s: d = s/2, so s² / s² = **1.0**.
  - **Circle** of radius r: d = r, so πr² / (2r)² = **0.785**.
  - **a × b rectangle** (b thin): d = b/2, so ab / b² = **a/b**.
  - These match the slide's reference values (0.78, 1.00, 1.85).
- **Lecturer's warning:** in the slide's table, the thin green shape gets **1.82**, the **same** as the thick rectangles. The lecturer said this is **wrong**: a thin shape should be much **more** elongated. The lesson is to **always sanity-check your feature values**.

> **中文解释：** 细长度**不能**用最小外接矩形的长宽比（弯曲的细长形状外接矩形可能接近正方形）。正确做法：面积 ÷ (厚度)²，厚度 = 2d，**d = 腐蚀多少次后区域完全消失**（腐蚀从两边同时进行，所以乘 2）。老师特别提到幻灯片中绿色细长形状的数值是**算错的**，提醒我们一定要做**合理性检查**。

#### 中文详解：细长度在做什么、怎么算

**1. 为什么不能用 MBR 的长宽比？**
MBR 只看外框大小，不管区域本身有多"粗"。例子：一个 "[" 形状，线条粗 2 像素，竖线长 10，上下横臂各长 6。
- MBR ≈ 6 × 10，长宽比 ≈ **1.67**，看起来不细长。
- 但把 "[" **拉直**，它是一根长约 18、粗 2 的细条，非常细长。

所以要换思路：**不看外框，看区域本身有多厚。**

**2. 为什么定义成 面积 ÷ 厚度²？**
把形状想成一根拉直的棍子：面积 ≈ 长 × 厚，所以 面积 / 厚² ≈ **长 / 厚**。不管形状弯不弯，这个值都约等于"拉直之后的长宽比"。

**3. 腐蚀 (erosion) 和 d**
- **腐蚀** = 把区域最外面一层像素剥掉（前景像素只要旁边有背景像素，就被删掉）。
- **d** = 反复腐蚀，直到区域**完全消失**，一共腐蚀了几次。
- **厚度 = 2d**：每次腐蚀两边同时各剥掉一层。

```
腐蚀前：  ████████   ← 厚度 4
1 次后：   ██████    ← 上下各少一层，厚度 2
2 次后：   (消失)     → d = 2，厚度 = 2d = 4
```

**4. 例子**

| 形状 | 面积 | d | 厚度 2d | 细长度 |
|---|---|---|---|---|
| 8 × 2 长条 | 16 | 1 | 2 | 16 / 4 = **4**（= 8/2 ✓） |
| 4 × 4 正方形 | 16 | 2 | 4 | 16 / 16 = **1** ✓ |
| "[" 形状（线粗 2） | 10×2 + 2×(4×2) = 36 | 1 | 2 | 36 / 4 = **9** ✓ |

MBR 长宽比对 "[" 只给出 1.67，这个方法给出 9，正确反映它是一根弯曲的细线。

**5. 面积用的是区域面积，不是 MBR 面积**
- 公式里的面积 = **区域本身的像素个数**（3.1 节），不是外接矩形面积。
- 用 "[" 举例：区域面积 36 → 细长度 9 ✓；MBR 面积 6 × 10 = 60 → 60 / 4 = 15 ✗（把开口处的空白也算进去了）。
- 实际上不会"只有 MBR 没有区域"：流程是 阈值化 → CCA → 区域像素 → 特征。MBR 本身就是从区域算出来的。**面积（数像素）和 d（对区域腐蚀）都要用区域本身来算**，MBR 提供不了。
- 如果题目给了矩形度，可以反推：**区域面积 = 矩形度 × 长 × 宽**。但仍需要 d 才能算细长度。

**6. 区域有宽有窄时，d 怎么算？**
**d 由最厚的部分决定。** 窄的部分先被腐蚀掉，但只要最厚的部分还剩像素，就要继续腐蚀。

例子：6 × 6 方块 + 伸出的 20 × 2 细棍。

```
██████
██████
██████████████████████████
██████████████████████████
██████
██████
```

| 腐蚀次数 | 细棍（厚 2） | 方块（厚 6） |
|---|---|---|
| 1 次 | **消失** | 剩 4 × 4 |
| 2 次 | – | 剩 2 × 2 |
| 3 次 | – | **消失** |

- d = 3，厚度 = 6，面积 = 36 + 40 = 76，细长度 = 76 / 36 ≈ **2.1**。
- 但细棍单独算是 40 / 4 = 10。结果**被方块"拖低"了**。
- **结论：** d 反映的是**最大厚度**（约等于区域内能放下的最大圆的半径）。粗细均匀的形状（直棍、弯曲细线、正方形、圆）很准；**有粗有细时，细长度会被低估**。
- 课外补充（一般不考）：可以用**距离变换 (distance transform)** 算局部厚度，或者求**骨架 (skeleton)** 得到拉直后的长度。

**7. 怎么做合理性检查 (sanity check)？**
核心问题：**"这个数字对这个形状来说合理吗？"**

- **用已知答案的形状测试代码：** 正方形 ≈ 1.0，圆 ≈ 0.785，40 × 4 矩形 ≈ 10。如果自己画的 40 × 4 长条算出 1.8，是**代码**的问题（多半是 d 或面积算错）。
- **计算前先用眼睛估：** 细长度 ≈ 拉直后的长度 ÷ 厚度。比如长约 60、厚约 4 的弯线，结果应该在 **15** 左右。算出 2 或 200 就肯定不对。**数量级对就行。**
- **比较形状之间的大小顺序：** 细的形状细长度应该**更高**；矩形的矩形度应该比 "X" 高。**老师的例子：** 细细的绿色形状算出 1.82，和粗矩形差不多，顺序不对，所以数字是错的。
- **检查取值范围：** 矩形度一定在 0~1 之间（超过 1 一定是 bug）；细长度不应明显小于 0.785（圆）；d 不可能超过 MBR 宽度的一半。
- **看中间结果：** 把 MBR 画在图上看是否贴紧；显示每一步腐蚀结果看 d 对不对（细形状应很快消失）；打印面积、d、MBR 长宽，错误通常能追溯到其中一个。

**一句话总结：** 细长度 = 面积 ÷ (2d)²，d = 腐蚀几次后区域消失，衡量的是"拉直后的长宽比"。面积和 d 都来自区域本身。算完要做合理性检查：记住参考值、先估算、互相比较、不对就查中间步骤。

### 3.4 Convex hull (凸包)
- **The smallest convex region** that contains the shape (think of an elastic band stretched around it). It is generally **smaller than the MBR**.
- **Algorithm** (gift-wrapping):
  1. Start at a point **known to be on the hull** (for example the **top-left** point, since nothing is above it), with a starting direction (for example horizontal).
  2. Search **all other boundary points** and find the one with the **least angle** to the previous vector.
  3. Move to that point, and make the new vector the current one.
  4. Repeat from step 2 until you return to the **start point**.

```cpp
vector<vector<Point>> hulls(contours.size());
for (int contour = 0; contour < contours.size(); contour++)
    convexHull(contours[contour], hulls[contour]);
```

> **中文解释：** 凸包 = **橡皮筋套住区域后的形状**，把凹陷"填平"。用 gift-wrapping 算：从最左上角出发，每次选**转角最小**的点，绕一圈回到起点。它比 MBR 更贴合区域，主要用来衡量和找出形状的**凹陷**。

#### 中文详解：凸包是什么、怎么算

**1. 什么是"凸"？**
区域是**凸 (convex)** 的 = 区域里**任意两点连直线，整条线都在区域内部**。
- 凸：圆、正方形、三角形。
- 不凸（凹）："U"、"C"、"X"、星形。比如 "U" 两个顶端连线，会穿过中间开口跑到区域外面。

**2. 凸包 = 能包住整个区域的最小凸形状**
**橡皮筋比喻：** 把区域想成木板上的钉子，撑大一根橡皮筋套住所有钉子再松手，橡皮筋收紧后围成的形状就是凸包。
- 凸的地方：橡皮筋紧贴边界。
- 凹进去的地方：橡皮筋直接**跨过去**，把凹陷填平。
- 例子："U" 的凸包是矩形（开口被包进去）；"X" 的凸包大致是菱形/正方形；圆、正方形本身是凸的，凸包就是自己。

**3. 为什么凸包一般比 MBR 小？**
MBR 是矩形，**矩形本身也是凸的**，也包住了区域；而凸包是**所有**包住区域的凸形状里**最小**的。所以 **凸包面积 ≤ MBR 面积**，只有区域本身是矩形时两者相等。例：圆的凸包 = 圆本身（πr²），MBR 是正方形（4r²）。

**4. 礼品包装算法 (Gift-wrapping)：像包礼物一样用绳子沿外面绕一圈**
1. **找一个一定在凸包上的起点**，比如**最左上角的点**（它上面没有任何点，一定在最外面）。起始方向设为水平向右。
2. 从当前点出发，**检查所有其他边界点**：算"当前点 → 候选点"的方向和当前前进方向的夹角，**夹角最小的点**就是下一个凸包顶点。
3. 走到这个点，把"上一个点 → 这个点"作为新的前进方向。
4. 重复 2、3，**直到回到起点**。

**直觉：** 绳子绕着钉子走，每次都**尽量少拐弯**，就总是贴着最外面的点走，不会漏掉外侧的点，也不会拐进凹陷里。

**5. 例子："U" 形**（图像坐标，y 向下）
```
(0,0) (1,0)       (3,0) (4,0)
  █     █           █     █
  █     █           █     █
  █     █ (1,3)(3,3)█     █
  █     █████████████     █
(0,4)                   (4,4)
```

| 步骤 | 当前点 | 前进方向 | 候选点及转角 | 选中 |
|---|---|---|---|---|
| 1 | (0,0) | 向右 → | (1,0)、(3,0)、(4,0) 都是 0°（共线） | **(4,0)**：共线时取最远的点 |
| 2 | (4,0) | 向右 → | (4,4) 转 90°；(3,3) 约 108°；(0,4) 135° | **(4,4)** |
| 3 | (4,4) | 向下 ↓ | (0,4) 转 90°；(3,3)、(1,3) 转角更大 | **(0,4)** |
| 4 | (0,4) | 向左 ← | (0,0) 转 90° | **(0,0)**，回到起点，结束 |

结果：凸包 = (0,0) → (4,0) → (4,4) → (0,4)，一个正方形。凹进去的点 (1,3)、(3,3) **都被跳过**，因为走到它们要拐更大的弯。这就是橡皮筋"跨过凹陷"。

**6. 凸包有什么用？**
凸包本身不是最终特征，而是用来**算其他特征**：
- **凸包 / MBR 比**（3.2）：凸包占外接矩形多少。
- **凹陷 (concavities)**（3.6）：**凸包 − 区域本身** = 形状凹进去的地方（"U" 的开口、"C" 的缺口）。数凹陷的个数和大小可以区分字母或物体。
- **区域面积 vs 凸包面积：** 接近 → 形状很"实"，几乎没凹陷；差很多 → 凹陷多（"X"、星形）。

**7. OpenCV**
- 每个区域的轮廓 (contour) 算一个凸包，存在 `hulls` 里。
- 输入是**轮廓**而不是所有像素，因为凸包顶点一定在边界上，内部像素不可能是凸包顶点。
- OpenCV 实际用更快的算法（如 Sklansky），结果一样。考试按 gift-wrapping 解释即可。

### 3.5 Moments and moment invariants (矩与矩不变量)
Moments measure the **distribution of the shape** (and optionally its grey levels) about its position.

**Raw moments:**
$$M_{xy} = \sum_i \sum_j i^x j^y f(i,j)$$
- f(i,j) = 1 or 0 for a binary region, or the grey level if you want intensity included.
- $M_{00}$ = **area**. The **centroid** is $(\bar i, \bar j) = \left(\frac{M_{10}}{M_{00}}, \frac{M_{01}}{M_{00}}\right)$.

**Central moments $\mu_{xy}$** are moments about the **centroid**, so they are **translation invariant**:
- $\mu_{00} = M_{00}$, $\mu_{10} = \mu_{01} = 0$
- $\mu_{11} = M_{11} - \dfrac{M_{10}M_{01}}{M_{00}}$, $\mu_{20} = M_{20} - \dfrac{M_{10}^2}{M_{00}}$, $\mu_{02} = M_{02} - \dfrac{M_{01}^2}{M_{00}}$
- There are third-order ones too ($\mu_{21}, \mu_{12}, \mu_{30}, \mu_{03}$). You don't need to memorise them. (Note: the slide writes "$\mu_{12} = M_{21} - …$". That should be $M_{12}$, a typo.)
- Intuition: $\mu_{20}$ and $\mu_{02}$ measure the spread in each direction; $\mu_{11}$ measures the tilt.

**Scale-invariant (normalised) central moments:**
$$\eta_{xy} = \frac{\mu_{xy}}{\mu_{00}^{\,1+\frac{x+y}{2}}}$$

> 🔍 **怎么读 η 和 μ（课件没讲/了解即可）**
> 1. **η** 是希腊字母 **eta**。英国和爱尔兰读 "EE-tuh"，美国常读 "AY-tuh"；中文念"伊塔"。
> 2. **μ** 是希腊字母 **mu**，读 "mew"；中文念"缪"。
> 3. 带下标时逐个读数字：$\eta_{20}$ = "eta two-zero"，$\mu_{11}$ = "mu one-one"。
> 4. 两者关系：μ 是**中心矩**（相对质心算），η 是 μ 除以面积的某个次方后得到的**归一化**版本。

Example: a big "9" and a small "9" give the same value.

**Hu moment invariants $I_1 … I_7$** are combinations of the $\eta$ values that are **invariant to translation, scale and rotation**:
- $I_1 = \eta_{20} + \eta_{02}$ (the overall spread about the centroid)
- $I_2 = (\eta_{20} - \eta_{02})^2 + 4\eta_{11}^2$ (how elongated or asymmetric the shape is)
- $I_3 … I_7$ use third-order moments and have **no intuitive meaning**. **Don't memorise these formulas.**

**Digits example (slide 16):** the **first Hu moment** splits the digits 0–9 into two groups: about five at ≈ 0.40–0.47 (1, 2, 3, 5, 7) and about five at ≈ 0.17–0.23 (0, 4, 6, 8, 9). One feature won't identify every digit, but it **helps**. Combining several features does the rest.

```cpp
Moments contour_moments = moments(contours[contour]);
double hu_moments[7];
HuMoments(contour_moments, hu_moments);
```

> **中文解释：** 矩描述形状的"分布"。原始矩 $M_{00}$ = 面积，$M_{10}/M_{00}$、$M_{01}/M_{00}$ = 质心。**中心矩**相对质心计算，所以与平移无关；**归一化中心矩 η** 与尺度无关；**Hu 不变矩（7 个）**同时与平移、尺度、旋转无关。除了前一两个，Hu 矩没有直观含义，不用背公式，知道它们的"不变性"即可。

#### 中文详解：矩在做什么、为什么能做到不变

**0. 物理直觉**
把每个前景像素想成一个**质量为 1 的小球**：矩就是物理里的**质量（面积）、质心、转动惯量（散开程度）**。矩描述"**像素是怎么分布的**"：中心在哪、往哪个方向散开、有没有倾斜。

**1. 原始矩 $M_{xy} = \sum_i \sum_j i^x j^y f(i,j)$**
- (i, j) 是像素坐标（i 看作横坐标，j 看作纵坐标）；f = 1（前景）或 0（背景），或用灰度值。x、y 是**阶数**。
- $M_{00} = \sum 1$ = **面积**；$M_{10} = \sum i$；$M_{01} = \sum j$。
- **质心** = 坐标平均值：$\bar i = M_{10}/M_{00}$，$\bar j = M_{01}/M_{00}$。
- **问题：** 依赖**位置**。形状平移后，原始矩全都变了。

**2. 中心矩：解决平移**
不从原点量，而是**从质心量**：$\mu_{xy} = \sum_i \sum_j (i-\bar i)^x (j-\bar j)^y f(i,j)$。
形状平移时质心跟着平移，像素**相对质心**的位置不变，所以与平移无关。
- 笔记里 $\mu_{20} = M_{20} - M_{10}^2/M_{00}$ 等公式只是**用原始矩快速计算的捷径**，结果和定义一样。
- $\mu_{00}$ = 面积；$\mu_{10} = \mu_{01} = 0$（相对质心，正负偏差抵消）。
- 二阶中心矩 = 统计里的**方差/协方差**：

| 矩 | 含义 |
|---|---|
| $\mu_{20} = \sum (i-\bar i)^2$ | **横向**散开程度 |
| $\mu_{02} = \sum (j-\bar j)^2$ | **纵向**散开程度 |
| $\mu_{11} = \sum (i-\bar i)(j-\bar j)$ | **倾斜**：0 = 不斜，正负 = 往哪边斜 |

**例 1：3 × 2 矩形**（宽 3、高 2，共 6 个像素）

先把点画出来（i 向右，j 向下，和图像坐标一样；● = 前景像素，✚ = 质心）：

```
         i=0   i=1   i=2
  j=0     ●     ●     ●
                ✚            ← 质心 (1, 0.5)：中间那一列，两行正中间
  j=1     ●     ●     ●
```

1. **面积：** 数点，$M_{00} = 6$。
2. **质心：**
   - $M_{10} = \sum i$：每行是 0 + 1 + 2 = 3，两行共 6，所以 $\bar i = 6/6 = 1$。
   - $M_{01} = \sum j$：上面一行 3 个点 j = 0，贡献 0；下面一行 3 个点 j = 1，贡献 3。所以 $\bar j = 3/6 = 0.5$。
3. **每个点相对质心的偏差，逐个算：**

| 像素 (i, j) | $i-\bar i$ | $j-\bar j$ | $(i-\bar i)^2$ | $(j-\bar j)^2$ | $(i-\bar i)(j-\bar j)$ |
|---|---|---|---|---|---|
| (0, 0) 左上 | −1 | −0.5 | 1 | 0.25 | +0.5 |
| (1, 0) 中上 | 0 | −0.5 | 0 | 0.25 | 0 |
| (2, 0) 右上 | +1 | −0.5 | 1 | 0.25 | −0.5 |
| (0, 1) 左下 | −1 | +0.5 | 1 | 0.25 | −0.5 |
| (1, 1) 中下 | 0 | +0.5 | 0 | 0.25 | 0 |
| (2, 1) 右下 | +1 | +0.5 | 1 | 0.25 | +0.5 |
| **合计** | | | **$\mu_{20} = 4$** | **$\mu_{02} = 1.5$** | **$\mu_{11} = 0$** |

4. **解读：**
   - $\mu_{20} = 4 > \mu_{02} = 1.5$：点在横向离质心更远，形状**横向更宽** ✓
   - $\mu_{11} = 0$：左上、右下的 +0.5 和右上、左下的 −0.5 正好抵消，所以**不斜** ✓
5. **平移验证：** 整体右移 10 格，i 变成 10、11、12，$\bar i = 11$，但 $i - \bar i$ 仍是 −1、0、+1，表里每一列都不变，$\mu_{20}$ 仍是 4 ✓

**例 2：斜线** (0,0) (1,1) (2,2)

```
         i=0   i=1   i=2
  j=0     ●     ·     ·
  j=1     ·     ●✚    ·      ← 质心 (1, 1)，正好在中间那个点上
  j=2     ·     ·     ●
```

| 像素 (i, j) | $i-\bar i$ | $j-\bar j$ | $(i-\bar i)^2$ | $(j-\bar j)^2$ | $(i-\bar i)(j-\bar j)$ |
|---|---|---|---|---|---|
| (0, 0) | −1 | −1 | 1 | 1 | +1 |
| (1, 1) | 0 | 0 | 0 | 0 | 0 |
| (2, 2) | +1 | +1 | 1 | 1 | +1 |
| **合计** | | | **$\mu_{20} = 2$** | **$\mu_{02} = 2$** | **$\mu_{11} = 2$** |

- $\mu_{20} = \mu_{02}$：只看横向、纵向散开程度，看不出它是斜线。
- $\mu_{11} = 2 \neq 0$：两端的点偏差**同号**（都是负负或正正），乘积都为正，不会抵消，所以**有倾斜** ✓（j 向下时，$\mu_{11} > 0$ 表示从左上斜到右下，像反斜杠 \ 。）

**3. 归一化中心矩 η：解决缩放**
放大后像素更多、离质心更远，$\mu$ 变大。所以除以面积的某个次方：$\eta_{xy} = \mu_{xy} / \mu_{00}^{\,1+\frac{x+y}{2}}$。
**为什么是这个次方？** 形状放大 s 倍：
- 面积 $\mu_{00}$ 变 **s²** 倍。
- $\mu_{xy}$：像素数 × s²，每项 × $s^{x+y}$，合计 **$s^{2+x+y}$** 倍。
- 分母 $(s^2)^{1+\frac{x+y}{2}} = s^{2+x+y}$，**正好抵消**。

例：a × b 矩形（连续情况）$\eta_{20} = a/(12b)$，**只和长宽比有关**，与大小无关 ✓（像素图因离散化有一点误差）。

**4. Hu 不变矩：再解决旋转**
η 仍依赖方向（矩形转 90°，$\eta_{20}$ 和 $\eta_{02}$ 互换）。Hu 把 η 组合成 7 个对**平移、缩放、旋转**都不变的数。
- **$I_1 = \eta_{20} + \eta_{02}$：整体散开程度。** 横向 + 纵向，转了之后总和不变。越紧凑越小，越分散/细长越大。

| 形状 | $I_1$ |
|---|---|
| 圆 | $1/(2\pi) \approx$ **0.159**（最小，最紧凑） |
| 正方形 | 1/12 + 1/12 ≈ **0.167** |
| 10 × 1 长条 | 10/12 + 1/120 ≈ **0.84** |

- **$I_2 = (\eta_{20} - \eta_{02})^2 + 4\eta_{11}^2$：两个方向差多少**，即细长/不对称程度。正方形、圆 $I_2 = 0$；长条很大。$4\eta_{11}^2$ 让**斜放**的长条也得到同样的值。
- **$I_3 \sim I_7$：** 三阶矩，没有直观含义，**不用背**，知道它们也不变即可。

**5. 层层递进**

| 矩 | 平移不变 | 缩放不变 | 旋转不变 |
|---|---|---|---|
| 原始矩 $M$ | ✗ | ✗ | ✗ |
| 中心矩 $\mu$ | ✓ | ✗ | ✗ |
| 归一化中心矩 $\eta$ | ✓ | ✓ | ✗ |
| Hu 不变矩 $I$ | ✓ | ✓ | ✓ |

**减去质心** → 平移；**除以面积的次方** → 缩放；**组合成 Hu 矩** → 旋转。

**6. 数字例子（第 16 页）**
$I_1$ 把 0~9 分成两组：约 0.40~0.47（1、2、3、5、7，开放线条，像素较"散"）和约 0.17~0.23（0、4、6、8、9，有封闭圈，较"紧凑"）。一个特征分不开所有数字，但能分组；再结合孔洞数、凹陷（3.6）等特征就能进一步区分。

**7. OpenCV**
- `moments()`：从轮廓算出所有原始矩、中心矩、归一化中心矩（m00、mu20、nu20 等）。
- `HuMoments()`：从这些矩算出 7 个 Hu 不变矩，放进 `hu_moments` 数组。

**一句话总结：** 矩把区域当作一堆有质量的点：$M_{00}$ = 面积，一阶矩给质心，二阶中心矩描述散开和倾斜。依次**减去质心、除以面积的次方、组合成 Hu 矩**，就得到不受位置、大小、旋转影响的特征，适合做识别。

### 3.6 Concavities and holes (凹陷和孔洞)
- **Concavities** are the gaps between the shape and its **convex hull**. OpenCV calls them **convexity defects**.
- **Holes** are background regions **inside** the shape. They are easy to get from the `findContours` hierarchy.
- **Licence-plate digit example** (slide 17):
  - The **6**'s biggest concavity is at **its own top right**: the gap under the top hook, to the right of the stem (left table, row 6: area 3.1 at 317°).
  - The **9**'s biggest concavity is at **its own bottom left**: the gap above the bottom tail, to the left of the stem (left table, row 9: area 3.0 at 143°).
  - These positions describe **where the gap is on the digit**, not where the digit is on the slide. The numbers come from the **left table** (training digits 0–9), not the right table (the plate).
  - Holes: 0, 4, 6 and 9 have **1**; 8 has **2**; the others have **0**.
- **Pixelisation creates tiny false concavities.** A "7" shows **9 concavities** because of the staircase on its diagonal. Only **large** concavities matter, but it's unclear what counts as "large", so don't throw information away too early.
- **Recognition result:** the plate **95-D-37825** was read as **95037825**, because the **D** was classified as **0**. The system was only trained on digits.

```cpp
convexHull(contours[contour], hull_indices[contour]);          // hull as indices
convexityDefects(contours[contour], hull_indices[contour], convexity_defects[contour]);
```

> **中文解释：** **凹陷 = 凸包减去区域**，描述"哪里缺了一块"；**孔洞 = 区域里包住的背景**，描述"中间有几个洞"。用 `convexityDefects` 找凹陷，用 `findContours` 的层级找孔洞。它们能区分 6 和 9 这种旋转后相同的形状。要注意像素化造成的小假凹陷，也要注意分类器只能识别训练过的类别。

#### 中文详解：凹陷和孔洞在做什么、怎么读第 17 页的表

**1. 为什么需要这两个特征？**
很多形状整体轮廓很像，真正的区别在于**哪里凹进去了**（6 vs 9）和**中间有没有洞**（0 vs 1，8 vs 3）。这两个特征对识别字母、数字特别有用。

**2. 凹陷 (Concavities) = 凸包 − 区域本身**
橡皮筋跨过去、但区域没填满的空白（凸包见 3.4）。例："U" 顶部开口、"C" 右边缺口、"6" 钩子下面右上方的缺口；"O"、正方形没有凹陷。

OpenCV 叫 **convexity defects（凸缺陷）**：
- `convexHull(..., hull_indices)`：凸包存成**轮廓点的下标**而不是坐标，因为 `convexityDefects` 需要知道凸包顶点是轮廓上的**第几个点**，才能找出两个顶点之间凹进去的那段轮廓。
- `convexityDefects(...)`：每个凹陷返回 4 个数：

| 值 | 含义 |
|---|---|
| start | 凹陷**开始**的凸包顶点（橡皮筋离开区域处） |
| end | 凹陷**结束**的凸包顶点（橡皮筋重新碰到区域处） |
| farthest | 凹陷里**离橡皮筋最远**的轮廓点（最深处） |
| depth | 最深处到橡皮筋的**距离**（OpenCV 存的是 ×256 的整数） |

想象：橡皮筋从 start 跨到 end，下面的轮廓凹下去，最深处是 farthest，深度是 depth。

**3. 孔洞 (Holes) = 区域内部被完全包围的背景**
- 0、4、6、9：1 个洞；8：2 个洞；1、2、3、5、7：没有洞。
- 按 1.3 节规则，孔洞是背景，用 **4 邻接**。
- **用 `findContours` 的层级 (hierarchy) 找：** 外轮廓是"父"，孔洞的边界（内轮廓）是它的"子"。
  - **孔洞个数** = 外轮廓有几个子轮廓；**孔洞面积** = 对每个子轮廓算面积（如 `contourArea`）。
  - 每个轮廓的层级是 4 个数：[下一个, 上一个, 第一个子轮廓, 父轮廓]。从"第一个子轮廓"开始沿"下一个"走，就能数完所有洞。

**4. 看懂第 17 页的表**

| 列 | 含义 |
|---|---|
| Shape | 这个字符（被识别成什么） |
| Height / Width | MBR 高宽比（3.2） |
| Hull / Box | 凸包面积 / MBR 面积（3.2、3.4） |
| # holes | 孔洞个数 |
| Hole Areas | 每个孔洞的面积 |
| # concavities | 凹陷个数 |
| Concavity Details | 最大的两个凹陷，写成 **面积@角度** |

- **"面积@角度"：** 如 6 的 **3.1@317**：3.1 = 凹陷面积（课件没说明单位，看起来是缩放过的值，不是像素数）；317° = 凹陷相对字符中心的**方向**。
- 课件没说明角度怎么量，但按**图像坐标（y 向下）**解读都说得通：6 的 317° → **右上** ✓；9 的 143° → **左下** ✓；5 的 140°（左下）和 302°（右上）正好是 5 的两个缺口 ✓。
- **左表** = 训练用的标准数字 0~9；**右表** = 车牌 95-D-37825 上每个字符的实际测量值。

**5. 6 和 9：凹陷方向是关键**

| | 孔洞 | 最大凹陷 |
|---|---|---|
| **6** | 1 个 | 3.1@317，**右上** |
| **9** | 1 个 | 3.0@143，**左下** |

6 和 9 是同一形状转 180°。3.5 的 **Hu 矩对旋转不变**，所以**分不开** 6 和 9；高宽比、凸包比、孔洞数也几乎一样。只有**凹陷的方向**相反，所以能区分。**"不变性"不一定总是好事**，有时方向本身就是区分的关键。

**6. 像素化造成的"假凹陷"**
左表里 **7 有 9 个凹陷**，但 7 只有一个大缺口。原因：斜线在像素图上是**锯齿状 (staircase)**：
```
██████
    ██
   ██     ← 每一级台阶和橡皮筋之间
  ██        留下一个小三角形空白
 ██         → 被算成一个"凹陷"
```
只看**大的**凹陷，但"多大算大"没有标准答案。所以**不要太早扔掉信息**：保留所有凹陷和面积，让后面的分类器决定哪些重要。

**7. D 被认成 0**
车牌 95-D-37825 被识别成 95037825。D 和 0 的特征几乎一样：

| | 高宽比 | 凸包/MBR | 孔洞 | 孔洞面积 | 凹陷 |
|---|---|---|---|---|---|
| **0**（左表） | 2.46 | 0.85 | 1 | 4.1 | 没有 |
| **D**（右表） | 2.54 | 0.87 | 1 | 4.5 | 只有很小的（0.4、0.1） |

更根本的原因：系统**只用数字 0~9 训练**，没见过字母，只能从 0~9 里选最像的。**教训：** 分类器只能认出**训练时见过的类别**；遇到没见过的东西，它不会说"不认识"，而是硬塞进最接近的类别。

### 3.7 Perimeter length and circularity (周长与圆形度)
- **Perimeter** ≈ the **number of boundary points**: `contours[contour].size()`.
  - Strictly, you should weight **diagonal steps by √2** rather than 1.
- **Circularity:**

$$\text{Circularity} = \frac{4\pi \cdot \text{Area}}{(\text{Perimeter})^2}$$

- A **circle gives 1** (4πr² / (2πr)² = 1). A square gives π/4 ≈ 0.785. The less compact the shape, the closer to 0.

> **中文解释：** 圆形度 = 4π·面积 / 周长²，**圆 = 1**，越不"圆"越接近 0。周长近似为边界点数，严格来说斜向步长应乘 √2。

#### 中文详解：周长和圆形度在做什么、怎么算

**1. 为什么需要这两个特征？**
面积只说明"有多大"，不说明"是什么形状"。三个面积都是 100 的形状：10×10 正方形、20×5 长方形、40×2.5 细条，面积一样，但一个很"团"，一个很"扁"。周长能补上这一点：**面积相同时，形状越不紧凑，边界越长。** 圆形度把面积和周长合成一个数，衡量形状有多"圆"（多紧凑）。

**2. 周长：数边界点**
OpenCV 用轮廓表示区域（2.5），轮廓就是边界点的列表。所以最简单的周长 = 边界点个数 = `contours[contour].size()`。
- 前提：`findContours` 要用 `CHAIN_APPROX_NONE`（保留每个点）。如果用 `CHAIN_APPROX_SIMPLE`，直线段只留两个端点，`.size()` 就不再是周长了。

例子：一个 6 像素的阶梯形（坐标 (x, y)，y 向下）：
```
█ . .
█ █ .
█ █ █
```
沿边界走一圈：(0,0) → (1,1) → (2,2) → (1,2) → (0,2) → (0,1) → 回到 (0,0)
- 边界点 **6 个**，所以粗略周长 = 6。
- 但这 6 步里有 **2 步是斜着走的**（(0,0)→(1,1)、(1,1)→(2,2)），每步实际长度是 √2 ≈ 1.414，不是 1。

**3. 为什么斜向步长要乘 √2？**
横/竖一步走 1 个像素；斜着一步同时横走 1、竖走 1，由勾股定理，长度 = √(1² + 1²) = √2。
- 上面的例子：精确周长 = 4 × 1 + 2 × √2 ≈ 4 + 2.83 = **6.83**。只数点得 6，**少算了约 12%**。
- 斜边越多，误差越大。极端情况：转了 45° 的正方形（菱形），边全是斜的，只数点会**少算约 29%**（1/√2 ≈ 0.707）。
- OpenCV 的 `arcLength(contour, true)` 把相邻点之间的真实距离加起来，**自动处理了 √2**（`true` 表示轮廓是闭合的）。

**4. 圆形度公式从哪来？**
$$\text{Circularity} = \frac{4\pi \cdot A}{P^2}$$
- 代入半径 r 的圆：A = πr²，P = 2πr，4π·πr² / (2πr)² = 4π²r² / 4π²r² = **1**。前面的 4π 就是为了让**圆正好等于 1**。
- 为什么其他形状都 < 1？几何上有个结论（等周定理）：**周长相同时，圆围出的面积最大。** 所以非圆形状的 A/P² 都比圆小，圆形度 ≤ 1。
- 为什么用 P²？面积的单位是"长度²"，周长是"长度"，P² 和 A 单位相同，相除后**没有单位**，所以和大小无关（见第 6 点）。

**5. 例子：几种形状的圆形度**

| 形状 | 面积 A | 周长 P | 4πA / P² |
|---|---|---|---|
| 圆，r = 10 | 314.2 | 62.8 | **1** |
| 正方形 10 × 10 | 100 | 40 | 400π / 1600 ≈ **0.785** |
| 等边三角形，边长 a | (√3/4)·a² | 3a | π√3 / 9 ≈ **0.605** |
| 长方形 20 × 5 | 100 | 50 | 400π / 2500 ≈ **0.503** |
| 细条 40 × 2.5 | 100 | 85 | 400π / 7225 ≈ **0.174** |

正方形、长方形、细条面积都是 100，但圆形度从 0.785 降到 0.174：**越扁、越细，越接近 0。**

**6. 尺度不变：放大不改变圆形度**
正方形边长从 10 变 20：面积 100 → 400（×4），周长 40 → 80，P² 1600 → 6400（×4）。分子分母同时 ×4，圆形度还是 **0.785**。
- 所以圆形度和 3.8 表里说的一样：**对尺度不变**，平移也不影响，旋转在理想情况下也不影响（但见第 7 点）。
- 周长本身**不是**尺度不变的（放大 2 倍，周长 ×2），所以做识别时通常用圆形度，不直接用周长。

**7. 为什么 √2 对圆形度很重要：转 45° 的正方形**
同一个正方形转 45°，圆形度应该还是 0.785。设一个菱形，四个顶点离中心 10 像素，边全是斜线：用 `contourArea` 算面积 = 200，边界一共 40 个斜向步。

| 周长算法 | P | 4π·200 / P² |
|---|---|---|
| 只数点 | 40 | 800π / 1600 ≈ **1.571** ✗（比圆还"圆"，不可能） |
| 斜步 × √2 | 40√2 ≈ 56.57 | 800π / 3200 ≈ **0.785** ✓ |

只数点时，同一个形状转一下，圆形度就从 0.785 变成 1.571，特征失去意义。这就是讲义说"严格来说斜向步长要乘 √2"的原因。

**8. 要注意的地方**
- **对边界噪声敏感：** 边界上的锯齿、毛刺会让周长变长，面积却几乎不变，所以圆形度被拉低。和 3.6 里像素化产生的假凹陷一样，小形状尤其明显。
- **面积和周长要用同一种表示：** `contourArea` 配 `arcLength` 最一致。如果面积用像素个数、周长用边界点中心的连线，小形状可能算出 > 1。例如 3×3 方块：像素面积 9，边界点中心连线周长 8，4π·9 / 64 ≈ 1.77。
- **单靠圆形度分不开所有形状：** 比如 0 和 D 都比较"团"。和其他特征一样，要组合使用（3.8）。

**9. OpenCV**
```cpp
double perimeter   = arcLength(contours[contour], true);   // 闭合轮廓，自动处理斜向 √2
double area        = contourArea(contours[contour]);
double circularity = 4 * CV_PI * area / (perimeter * perimeter);
```
Python：`P = cv2.arcLength(cnt, True)`，`A = cv2.contourArea(cnt)`。

**一句话总结：** 周长 ≈ 边界点数（斜步要算 √2，`arcLength` 自动处理）；圆形度 = 4πA / P² 把面积和周长合成一个与大小无关的数：**圆 = 1，正方形 ≈ 0.785，越扁越细越接近 0**。

### 3.8 Feature summary
| Feature | Invariant to scale? | Notes |
|---|---|---|
| Area | ✗ | Too simple alone |
| MBR: length/width, rectangularity | ✓ (ratios) | Rectangularity: 1 = a rectangle |
| Elongatedness | ✓ | area / (2d)², d = number of erosions |
| Convex hull (and hull/MBR ratio) | ✓ (ratio) | Gift-wrapping algorithm |
| Moments / Hu invariants | ✓ (η, Hu) | Hu: translation, scale and rotation invariant |
| Concavities and holes | counts ✓ | Pixel noise gives false small concavities |
| Perimeter and circularity | circularity ✓ | Circle = 1 |

**The hard problem:** there are many features, and choosing **which ones** to use is difficult. This is addressed later with classifiers.

> 📝 **应用题里怎么用区域特征**
> 1. **说清楚用哪几个特征、为什么选它。** 优先选与大小无关的比值类特征（矩形度、长宽比、圆形度、孔洞数），因为物体离相机远近不同，大小会变。
> 2. **阈值怎么定：** 不要凭空给一个数，要说"用一组已知样本算出每个特征的合理范围"。
> 3. **例子（我自己举的，课件没讲）：**
>    - 出口标志、车牌：接近矩形，所以矩形度接近 1，长宽比在某个范围内。
>    - 自行车标志：两个轮子，孔洞数大约是 2。
>    - 圆形物体：圆形度接近 1（2023 Q3(b) 就考了"mean shift + 圆形度"找圆）。
> 4. **一个特征通常不够**（就像第一个 Hu 矩只能把数字分成两组），要组合几个特征。

---

## 4. k-means Clustering

### 4.1 Purpose
- Find the **significant colours** in an image or region:
  - **Concise descriptions**, for example "boy in a **green T-shirt** and **tan trousers**" from an image reduced to 4 colours.
  - **Object tracking**: follow a person by their clothing colours, even through **occlusions** when people cross paths.
- **Reduce the number of colours**, which can also be used for **compression**.
- **It is not spatial.** It clusters in **colour space** only. Shuffling the pixels gives the **same result**, so positions are ignored. That makes it unlike CCA.
- It is an **unsupervised learning** technique.

> **中文解释：** k-means 在**颜色空间**里把像素分成 k 类，从而找出图像中最主要的 k 种颜色（例如"绿色 T 恤 + 棕褐色裤子"）。注意：它**完全不考虑像素位置**，打乱像素结果不变。属于**无监督学习**。

### 4.2 Algorithm (the basic, original version)
```
Choose k (known in advance, or try several and pick the most confident)
Initialise k exemplars (cluster centres):
    randomly, OR the first k patterns, OR k random pixels from the image
1st pass: for each pattern (pixel colour):
    allocate it to the CLOSEST exemplar
    recompute that exemplar as the centre of gravity (mean) of its patterns
2nd pass: using the final exemplars from pass 1,
    reallocate ALL patterns to their closest exemplar
```
- A **pattern** is a pixel's value vector, for example (R, G, B).
- With **random** initialisation the algorithm is **non-deterministic**: each run gives different clusters (slide 26, three runs with k = 30).
- **Not all clusters end up with patterns.** With k = 10 on the snooker image, only **7** colours survived; exemplars that attract no patterns are discarded. With k = 20, only 16 survived.
- **More exemplars** generally give a **more faithful** image. Too few colours cause **false contours**, the same effect as quantisation. At k = 10 the blue ball and the white ball came out wrong.

### 4.3 My own 1D worked example
Patterns (grey levels) in order: **12, 2, 20, 4, 22, 10**. Initial exemplars: e₁ = 0, e₂ = 11, e₃ = 30.

**Pass 1** (each exemplar is updated to the mean of its patterns after every allocation):

| Pattern | Closest exemplar | Updated exemplars (e₁, e₂, e₃) |
|---|---|---|
| 12 | e₂ (distance 1) | 0, **12**, 30 |
| 2 | e₁ (2) | **2**, 12, 30 |
| 20 | e₂ (8) | 2, **16**, 30 |
| 4 | e₁ (2) | **3**, 16, 30 |
| 22 | e₂ (6 vs 8) | 3, **18**, 30 |
| 10 | e₁ (7 vs 8) | **5.33**, 18, 30 |

**Pass 2:** reallocate using e₁ = 5.33, e₂ = 18 → {2, 4, 10} and {12, 20, 22}. **e₃ received no patterns**, so it is discarded, just like on the slide. Note that 12 was allocated early, when e₂ was nearby, so the outcome **depends on the order and the initialisation**.

### 4.4 Choosing k: the Davies–Bouldin index
$$DB = \frac{1}{k}\sum_{i=1}^{k} \max_{j \neq i} \frac{\sigma_i + \sigma_j}{\delta_{i,j}}$$
- $\sigma_i$ = the average distance of the patterns in cluster i from its centre (the cluster's **spread**).
- $\delta_{i,j}$ = the **distance between the centres** of clusters i and j.
- For each cluster, find its **most confusable** neighbour (the max). Then average over all clusters.
- **Lower = better separation.**
- **Weakness:** it works poorly when the clusters have **very different sizes**. The snooker image has a huge green cluster and tiny white-ball and skin clusters.
- **Alternative:** check whether each cluster's distribution is **normal**. **Split** a cluster that isn't (it probably holds two colours), and **merge** two clusters that together form one normal distribution.

> **中文解释：** DB 指数衡量聚类分离程度：每个簇找"最容易混淆的另一个簇"（两簇内部离散度之和 ÷ 簇中心距离，取最大），再取平均，**越小越好**。缺点：簇大小差别很大时效果差。另一种方法：检查每个簇是否服从**正态分布**，不是就拆分，两个簇合起来是正态就合并。

### 4.5 OpenCV
`kmeans` isn't image-specific, so the pixels must be copied into a **1D array of samples** (rows × cols samples, 3 columns, float):
```cpp
Mat samples(image.rows*image.cols, 3, CV_32F);
for (int row = 0; row < image.rows; row++)
  for (int col = 0; col < image.cols; col++)
    for (int channel = 0; channel < 3; channel++)
      samples.at<float>(row*image.cols+col, channel) = (uchar) image.at<Vec3b>(row,col)[channel];

Mat labels, centres;
kmeans(samples, k, labels,
       TermCriteria(CV_TERMCRIT_ITER|CV_TERMCRIT_EPS, 0.0001, 10000),
       iterations, KMEANS_PP_CENTERS, centres);

// rebuild the image: every pixel gets the colour of its cluster centre
for (row...) for (col...) for (channel...)
  result_image.at<Vec3b>(row,col)[channel] =
      (uchar) centres.at<float>(*(labels.ptr<int>(row*image.cols+col)), channel);
```
Python:
```python
Z = img.reshape(-1, 3).astype(np.float32)
_, labels, centres = cv2.kmeans(Z, k, None,
        (cv2.TERM_CRITERIA_EPS + cv2.TERM_CRITERIA_MAX_ITER, 10000, 1e-4), 3, cv2.KMEANS_PP_CENTERS)
out = centres.astype(np.uint8)[labels.flatten()].reshape(img.shape)
```
(`KMEANS_PP_CENTERS` = k-means++ initialisation, a smarter alternative to purely random exemplars.)

---

## 5. Watershed Segmentation *(slides only; the lecturer said this is covered later in the course)*

- **The analogy from geology:** watersheds are the ridges that separate **catchment basins**, the areas where rain collects.
- **In vision:**
  1. Treat the image as a **landscape** and **identify all minima**.
  2. Give each minimum a **different region label**.
  3. **Flood** from the minima, growing the regions.
  4. Where two regions **meet**, you get **watershed lines**. These are the region boundaries.
- **Minimum of what?** The **greyscale** itself, the **gradient** (so boundaries sit on edges), or the **inverse of the chamfer distance** (to split touching blobs).
- **Problem:** you generally get **too many regions** (over-segmentation), because every little minimum becomes a region.
- **Fix: watershed with markers.** Use **a priori labels** (markers) to identify the "objects", and flood from these **instead of from every minimum**.
- **Unlike k-means**, it **does use spatial information**.
- OpenCV: `cv2.watershed(img, markers)`.

> **中文解释：** 分水岭算法：把图像当作地形，从每个**局部最低点**开始"注水"，不同水域相遇的地方就是**分水岭线**（区域边界）。缺点是**过分割**（区域太多），改进方法是用**标记 (markers)** 只从指定的物体位置开始注水。

> 📝 **考试提醒：** 课件说这部分后面才讲，但往年比较题考过（2022 Q2(b)），要和 CCA、Mean shift 一起掌握。比较要点见 §7 比较题素材。

---

## 6. Mean Shift Segmentation *(slides only; covered later in the course)*

### 6.1 Motivation: fixing k-means' weaknesses
| k-means | Mean shift (Comaniciu & Meer, 2002) |
|---|---|
| Needs the **number of clusters k** in advance | **No need** to know the number of clusters |
| **No spatial** information | Can do **spatial and colour** segmentation |

**Goal:** associate each pixel with a **high-density cluster (mode)** in colour space by moving "particles" **in the direction of increasing local density**.

### 6.2 Kernel density estimation (KDE)
To estimate density from sparse samples, **smooth each sample with a kernel and add them all together**:
$$\hat f_h(x) = \frac{1}{nh}\sum_{i=1}^{n} K\!\left(\frac{x - x_i}{h}\right) \qquad \text{(d dimensions: } \tfrac{1}{nh^d}\text{)}$$
- K = the kernel function (it **must integrate to 1**), n = the number of samples, **h = the bandwidth** (width).
- Typical kernels are **uniform** (a flat box; the slide also shows the Epanechnikov-type profile $k_E(x) = 1 - x$ for $0 \le x \le 1$) and **Gaussian**: $K_N(\mathbf{x}) = (2\pi)^{-d/2}\exp(-\tfrac12\|\mathbf{x}\|^2)$.

### 6.3 The mean shift vector
For a radially symmetric kernel $K(\mathbf x) = c\,k(\|\mathbf x\|^2)$ with $g(x) = -k'(x)$, the density gradient points along:
$$\mathbf m_{h}(\mathbf x) = \frac{\sum_i \mathbf x_i\, g\!\left(\left\|\frac{\mathbf x - \mathbf x_i}{h}\right\|^2\right)}{\sum_i g\!\left(\left\|\frac{\mathbf x - \mathbf x_i}{h}\right\|^2\right)} - \mathbf x$$
In words: **(the weighted mean of the nearby points) − (the current position)**. Moving by this vector goes **uphill in density**.

### 6.4 Algorithm
```
for each particle (pixel):
    repeat:
        estimate the local kernel density → compute the mean shift vector
        shift the particle to the new (weighted) mean
    until the location stabilises (it has reached a mode)
pixels that end up at the same location = the same cluster
```
- **Two kernels are used together, and both can be Gaussian:**
  - a **spatial kernel**, which limits and weights **which nearby pixels** are considered;
  - a **colour (range) kernel**, which limits and weights **how similar in colour** a pixel must be to count.
- **Varying the spatial radius or the colour radius** changes the result a lot (slide 40).
- OpenCV: `cv2.pyrMeanShiftFiltering(img, sp, sr)`, where sp = spatial radius and sr = colour radius.

### 6.5 Pros and cons
| + | − |
|---|---|
| No need to know the number of clusters in advance | **Choosing the kernel widths** (bandwidths) can be very hard |
| Gives **spatial and colour** segmentation | **Slow**, especially with many clusters |

> **中文解释：** 均值漂移：对每个像素，计算其邻域（同时考虑**空间距离**和**颜色相似度**两个核）的加权平均，并移动过去，重复直到收敛到"密度峰值"；收敛到同一点的像素归为同一类。优点：**不需要事先知道 k**、同时利用空间和颜色；缺点：**带宽难选**、**速度慢**。

> 📝 **考试提醒：** Mean shift 是这一章往年考得最多的：2022 Q2(b)、2023 Q1(b)、2023 Q3(b) 三道比较题都有它。要能讲清楚：
> 1. **输入：** 彩色图（不用先阈值化）。
> 2. **输出：** 颜色一致、位置相连的区域。
> 3. **两个参数：** 空间带宽（多远的像素算邻居）和颜色带宽（颜色差多少以内算相似）。带宽越大，合并得越多，区域越少越大。
> 4. **和 k-means 的区别：** 不需要 k、用位置信息、给定参数后结果确定；代价是慢、带宽难选。

---

## 7. Summary

| Technique | Input → Output | Spatial? | Key points |
|---|---|---|---|
| **Connectivity** | — | ✓ | 4 vs 8 adjacency; **alternate** (background 4, objects 8, holes 4, …) |
| **CCA** | binary → label image | ✓ | 2 passes; **equivalences**; OpenCV `findContours` |
| **Region features** | region → feature vector | — | Area, MBR, elongatedness, hull, moments, concavities/holes, circularity |
| **k-means** | colour image → k colours | ✗ | Needs k; random init → non-deterministic; DB index |
| **Watershed** | image → regions | ✓ | Flood from minima; over-segments; use **markers** |
| **Mean shift** | image → regions | ✓ | No k needed; spatial + colour kernels; slow, bandwidths are hard to choose |

### 📝 比较题素材：CCA vs Watershed vs Mean shift（2022 Q2(b)）
> 比较题**只按异同点给分**，分开描述每个技术不得分。按下面的"维度"逐条比，每一条都同时说到三个技术。

**相同点**
1. 三者都是**分割**技术，输出都是一组**带标签的区域**，后面都可以接区域特征做识别。
2. 三者都**用位置信息**（相邻的像素才会归到一起），这一点和 k-means 不同。
3. 三者都**不需要事先给出区域数量 k**，这一点也和 k-means 不同；区域数量由数据和参数决定。
4. 给定输入和参数，三者的结果都是**确定的**，不像随机初始化的 k-means。
5. 结果好坏都依赖**前处理或参数**：CCA 依赖阈值，Watershed 依赖从哪些最低点或 markers 开始，Mean shift 依赖两个带宽。

**不同点**

| 维度 | CCA | Watershed | Mean shift |
|---|---|---|---|
| 输入 | **二值图**（要先阈值化） | **单通道图**：灰度本身、梯度图、或距离变换的反相 | **彩色（或灰度）图**，直接用 |
| 原理 | 相邻前景像素贴同一标签；两遍扫描 + 等价合并 | 把图当地形，从最低点注水，水域相遇处是边界 | 每个像素向局部密度最高处移动，停在同一处的归为一类 |
| 用不用颜色 | 不用（只看 0/1） | 通常不用（单通道） | 用，颜色核直接比较颜色 |
| 参数 | 4 / 8 邻接 | 用什么做"地形"；是否用 markers | 空间带宽、颜色带宽 |
| 区域数量由什么决定 | 二值图里有几块连通区域 | 最低点（或 markers）的个数 | 密度峰值 (modes) 的个数，由带宽决定 |
| 主要问题 | 接触的物体会合并；噪声产生小区域；完全依赖阈值效果 | **过分割** | **慢**；**带宽难选** |
| 速度 | 很快（两遍扫描） | 一次注水过程，不迭代 | 慢（每个像素反复迭代） |
| 典型用途 | 阈值化后把物体一个个分开、计数 | 分开相互接触的物体（用距离变换）；沿边缘分割（用梯度） | 自然图像的颜色分割；也用于跟踪 |

**可以写的亮点：** CCA 的一个主要问题是接触的物体会合并，而 Watershed 用距离变换的反相做地形，正好能把接触的物体分开。两者可以配合使用。

### 📝 比较题素材：Canny vs CCA vs Mean shift（2023 Q1(b)）
> Canny 在 Edges 那一章，这里只列和本章技术比较时要用的要点。

1. **最大的区别：** Canny 是**基于边缘**的，输出细的**边缘像素**（二值边缘图），边缘不一定闭合，所以**不直接给出区域**。CCA 和 Mean shift 是**基于区域**的，输出区域。
2. **输入：** Canny 用灰度图；CCA 用二值图；Mean shift 用彩色图。
3. **参数：** Canny 是高斯平滑的 σ 和两个滞后阈值 (hysteresis thresholds)；CCA 是 4/8 邻接；Mean shift 是两个带宽。
4. **相同点：** Canny 的滞后阈值其实也用了**连通性**：弱边缘像素只有和强边缘**连在一起**才保留，这和 CCA 的思路一样。三者都用位置信息。
5. **可以组合：** Canny 的边缘图可以作为 CCA 的输入，比如对非边缘像素做 CCA，得到被边缘围起来的区域。

### Exam tips（按往年试卷调整）
- **没有计算题。** 不用手算 CCA、矩、DB 指数；笔记里的手算例子是帮你理解的。
- **比较题：** 这一章最可能考 CCA / Watershed / Mean shift / k-means 之间的比较。按维度（输入、原理、参数、输出、问题、用途）逐条比异同，见上面的比较题素材。
- **应用题：** 用"阈值化或颜色分割 → CCA → 区域特征筛选"这条链，每一步写清楚输入、输出、参数。**不能写 OpenCV 函数名**（写"连通域分析"，不写 `findContours`）。
- **这些概念要能讲清楚：** 连通性悖论和交替规则；细长度为什么不能用 MBR 长宽比；矩的不变性阶梯；k-means 不用位置、结果不确定、需要 k；Watershed 过分割和 markers；Mean shift 的两个核和缺点。

---

## 8. Practice Questions

1. Explain the connectivity paradox with a diagram. How is it resolved in practice?
2. Run CCA with 8-adjacency on this image, listing the labels and equivalences:
   ```
   1 0 1
   0 1 0
   1 0 1
   ```
   How many regions are there with 4-adjacency instead?
3. Why can't elongatedness be the length/width ratio of the minimum bounding rectangle? Give the correct formula.
4. Compute the rectangularity and circularity of a 10 × 10 square (take its perimeter to be 40).
5. What does each step (central → normalised → Hu) add in terms of invariance?
6. Why does a "7" in a pixel image show many concavities? How should you deal with this?
7. Give two reasons why k-means might give different results on the same image.
8. What does the Davies–Bouldin index measure, and when does it work badly?
9. Give two advantages and two disadvantages of mean shift compared with k-means.
10. Why does basic watershed over-segment, and how do markers help?

<details>
<summary><b>Answers</b></summary>

1. A diagonal ring: with 8-adjacency, the ring **and** the background both connect through the same diagonal corners, which is contradictory. With 4-adjacency, the ring breaks into many pieces. The fix is to alternate: background uses 4-adjacency, objects 8, holes 4, objects in holes 8, and so on.
2. 8-adjacency, scanning row by row:
   - (0,0) gets new label A; (0,2) gets new label B.
   - (1,1) has previous neighbours UL = A and UR = B, so it takes A and **A ≡ B** is noted.
   - (2,0) has UR = (1,1) = A, so it takes A.
   - (2,2) has UL = (1,1) = A, so it takes A.
   - Result: **1 region**. With 4-adjacency, no pixels share an edge, so there are **5 regions**.
3. A thin, bent shape (like "[" or "C") can have a nearly square MBR while being very elongated. The correct formula is **area / (2d)²**, where d = the number of erosions until the region vanishes.
4. Rectangularity = 100 / (10 × 10) = **1**. Circularity = 4π·100 / 40² = 400π / 1600 = **π/4 ≈ 0.785**.
5. Central moments add **translation** invariance (they are measured about the centroid). Normalised η adds **scale** invariance. Hu moments add **rotation** invariance.
6. The diagonal is a **staircase** of pixels, and each step creates a tiny concavity. Keep only the **large** concavities by thresholding on area, but be careful not to throw away real ones too early.
7. **Random initial exemplars** (it is non-deterministic), and the **order** in which patterns are allocated in the first pass. A different k also changes the result.
8. It measures **cluster separation**: for each cluster, (σᵢ + σⱼ) / δᵢⱼ with its most confusable neighbour, averaged over all clusters. **Lower is better.** It works badly when the clusters are **very different in size**.
9. Advantages: no need for **k**, and it uses **spatial + colour** information. Disadvantages: the **bandwidths are hard to choose**, and it is **slow**.
10. Every local minimum (many are caused by noise) becomes its own region. **Markers** give a priori seed labels for the objects, and the flooding grows only from those markers.
</details>

---

*Note: watershed (§5) and mean shift (§6) are in the slides, but the transcripts say they are "left for a little bit later in the course", so those sections are based on the slides only.*
*Related notes: thresholding and Otsu (Binary Vision), and erosion, used for elongatedness (Binary Vision: morphology). Histogram comparison with the Bhattacharyya metric is used in mean shift **tracking** (Histograms).*
