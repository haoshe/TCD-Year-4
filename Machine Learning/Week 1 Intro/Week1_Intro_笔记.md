# 机器学习 Week 1：Introduction to Machine Learning 中文笔记

> 对应课件：`intro.pdf`（共 18 页），讲师 Doug Leith
> 专业术语保留英文，方便你和课件、作业、考试对照。

---

## 0. 这一讲在讲什么？（一句话总结）

用一个**电影评论情感分析**（判断一条影评是好评还是差评）的完整例子，把机器学习的整个流程走一遍：

**原始数据（文字）→ 转成数字（特征）→ 选一个模型（线性模型）→ 用数据训练出参数 → 评估效果好不好**

后面整个学期基本都是在把这几个步骤逐一深入展开。所以这一讲虽然简单，却是全课的"地图"。

---

## 1. 课程管理信息（Slide 2–4）

### 1.1 基本信息
- 讲师：Doug Leith
- 课程资料：Blackboard
- 编程语言：**Python**，主要用 **scikit-learn（sklearn）** 库
- 先修要求：会写 Python
- 推荐的补充学习（Coursera）：
  - Applied Machine Learning in Python
  - Neural Networks and Deep Learning
- 讲师提醒：网上资料很多，但**博客之类的质量参差不齐，要小心**

### 1.2 评分方式 ⚠️
| 部分 | 占比 |
|---|---|
| 每周作业（weekly assignments） | **40%** |
| 期末大作业（final assignment） | **60%** |

注意：**没有写"期末考试"，而是期末 assignment**，所以平时动手写代码 + 写报告的能力非常关键。

### 1.3 学术诚信规定（Honour Code）——非常重要
**每周作业：**
- ✅ 可以（甚至鼓励）和同学**讨论**
- ❌ 但答案必须**自己用自己的话写**（明确说了**不能用 ChatGPT 等 AI**）
- ❌ 代码必须**自己写**，也**不能把代码分享给别人**
- ⚠️ **必须解释/论证你的答案是怎么得来的**：
  - 写了代码 → 要解释代码在干什么
  - 给出了数字结果或图 → 要讨论它们的含义
  - 只写"see code"、只贴数字或图而不解释 → **低分**

**期末作业：**
- 完全个人完成，**不能和任何同学讨论**，不能用 AI

**检查方式：** 所有提交都过反抄袭软件，每周还会随机抽查人工检查。

> 💡 给你的建议：这门课的分数很大程度取决于"**解释**"。以后每次作业，养成习惯：每一段代码、每一张图、每一个数字后面都写 2–3 句话说明"这是什么、为什么这样、说明了什么"。

### 1.4 课程结构（Slide 4）
- **前 3 周：节奏很快、很密集**，目标是让你尽快能把机器学习用到真实数据上
- 之后逐个深入：
  - **模型（Models）**：kNN、决策树（decision trees）、神经网络（neural nets）、核方法（kernel methods）→ 一直到 mid-term break
  - Mid-term 之后：**特征工程（feature engineering）** 和 **深度学习（deep learning）**
- 每周作业和课堂内容紧密相关，**千万别落下**

---

## 2. 监督学习 Supervised Learning（Slide 5）

### 2.1 什么是监督学习？
**用"带标签的数据"（labelled data）来学习，然后预测新数据的输出/目标值（output / target）。**

"监督"的意思是：训练时有"标准答案"（label）在旁边"监督"模型。就像做练习题时有答案可以对。

与之相对的是 **无监督学习（unsupervised learning）**：数据没有标签，模型自己找结构（比如聚类）。本课主要讲监督学习，期末前会简单涉及无监督学习。

### 2.2 两大类任务

| | 分类 Classification | 回归 Regression |
|---|---|---|
| 目标值类型 | **离散的类别**（discrete classes） | **连续的数值**（continuous values） |
| 课件例子 | 判断信用卡交易是否欺诈（欺诈 / 不欺诈） | 根据蓝牙信标的接收信号强度（如 -60dB）预测两人之间的距离 |
| 其他例子 | 垃圾邮件识别、图片里是猫还是狗 | 预测房价、预测明天气温 |

> 💡 判断方法：问自己"答案是**选一个选项**，还是**给一个数**？" 选选项 → 分类；给数字 → 回归。
> 本讲的影评例子（好评/差评）是**分类**问题。

---

## 3. 分类的基本框架（Slide 6）—— 核心概念，一定要懂

课件用水果图片做例子：给一张图，判断是苹果、橙子……

### 3.1 关键术语
- **训练集（training set）**：一堆已经标好答案的样本（图片 + "这是苹果"）
- **输入 x**：必须是一个**数字向量**，例如图片的所有像素值。也叫 **特征向量（feature vector）**。
  - x = (x₁, x₂, ..., xₙ)，每个 xᵢ 叫一个**特征（feature）**
- **输出 y**：也是一个数字，例如 1 = 苹果，2 = 橙子
- **分类器（classifier）**：就是一个**函数 h**，把输入 x 映射成对 y 的预测：

$$\hat{y} = h(x)$$

  - 读作 "y hat"，**ŷ 表示"预测值"**，y 表示"真实值"。这个区分以后会一直用到。
  - h 常被称为 **hypothesis（假设函数）**

### 3.2 机器学习要做的两件事
1. **把真实世界的输入（图片、文字、声音……）转成数字特征向量 x** → 这就是**特征工程（feature engineering）**
2. **学习函数 h(x)** → 这就是**训练模型**

训练完成后，给一个新的、从没见过的 x，分类器就能预测它的标签。

> 💡 为什么输入必须是数字？因为模型本质上是数学函数，只会做加减乘除。计算机"看不懂"文字和图片，必须先变成数字。

---

## 4. 训练数据从哪里来？（Slide 7–8）

### 4.1 获取标签（labels）才是难点
- 原始数据（图片、文本）通常**很容易**拿到
- 但**标签很难拿到**——需要有人告诉你"这张图是苹果"

**常见获取标签的方法：**
1. **雇人标注**：Amazon Mechanical Turk（众包平台）、验证码 Captcha（你点"选出所有红绿灯"其实就是在帮别人标数据）、直接雇人
   - 缺点：枯燥、容易出错
   - 讲师的一个批判性观点：看似"高大上"的智能机器，背后是**大量低薪的人工劳动**
2. **利用过去人类已经做过的工作**：
   - Wikipedia 文章及其人工分类
   - 学术论文及作者给的关键词/分类
3. **在线服务中记录结果（logging outcomes）**：
   - 例如：系统标记一笔交易可疑 → 后来调查确认是否真的欺诈 → 这个结果就成了标签
4. **用间接结果（indirect outcomes）代替**：
   - 真正关心的是"看了广告后有没有**购买**"，但这很难追踪（购买可能发生在几天后）
   - 于是用"有没有**点击**广告"代替 → 但**点击 ≠ 购买**，要小心

### 4.2 数据可能出什么问题？
1. **数据不具代表性（unrepresentative）**
   - 例：只从学生或只从男性收集数据 → 模型对其他人群不准
   - 例：数据是很久以前收集的，现在情况变了
2. **数据太"嘈杂"或不可靠（noisy / unreliable）**
   - 例：点击广告和最终购买之间只有很弱的关系
3. **数据没有捕捉到有用的关系**
   - **相关性 ≠ 因果性（Correlation vs Causation）**
   - 例：冰淇淋销量和溺水人数正相关，但冰淇淋并不导致溺水——两者都是因为"夏天"。模型只能学到相关性，学不到因果。

> 💡 核心思想：**Garbage in, garbage out**。模型再好，数据有问题，结果也不会好。

---

## 5. 机器学习工作流程（Slide 9）

```
Data Preparation（数据准备）  ← 通常占了大部分工作量！
        ↓
Choose Features（选择特征）
        ↓
Select Model（选择模型）
        ↓
Train Model（训练模型）
        ↓
Test Model（测试模型）
        ↓
Business Application（实际应用）  ← 这才是真正重要的
```

两个重点：
- **数据准备往往是最花力气的部分**（清洗、处理缺失值、格式转换……），不是调模型
- **最终真正重要的是实际应用效果**，而不是模型在纸面上的某个分数

---

## 6. 完整例子：电影评论情感分析（Slide 10–13）

### 6.1 问题
- 数据：IMDb 电影评论（Cornell 大学整理的数据集）
- 任务：给一段影评文字，判断是**正面（positive）** 还是**负面（negative）**
- 已有训练数据：一批已标好正/负的评论

### 6.2 直觉思路
- 有些词明显是正面的（wonderful, great），有些是负面的（terrible, awful）
- 数一下正面词和负面词各多少，哪个多就预测哪个
- **问题：怎么自动化？** 我们不想手动列词表 → **用带标签的训练数据自动学出哪些词是正面/负面的**

### 6.3 Bag of Words（词袋模型）—— 把文字变成数字

这是第一个**特征工程**的例子。步骤：

1. **去掉停用词（stop words）**：像 and, of, the, a 这种到处都有、不带情感的词
2. **词干提取（stemming）**：把词尾砍掉，统一成词根
   - happening, happened, happens → **happen**
   - 目的：让同一个意思的不同形式算作同一个词
3. **建立词典（dictionary）**：把所有评论中出现过的处理后的词（叫 **tokens**）收集起来，得到一个有 **N 个词**的大列表，每个词有一个编号
4. **把每条评论变成一个长度为 N 的数组**：第 i 个位置 = 第 i 个词在这条评论里出现的**次数**

**课件例子：** "a terrible mess of a movie starring a terrible mess of a man, mr. hugh grant"

| 词 | 在词典中的编号 | 出现次数 |
|---|---|---|
| grant | 10287 | 1 |
| hugh | 11485 | 1 |
| mess | 14967 | **2** |
| mr | 15553 | 1 |
| starring | 22491 | 1 |
| terrible | 23718 | **2** |

其他两万多个位置全是 0。（"movie"和"man"没出现在表里，可能因为太常见被过滤了。）

> 💡 为什么叫"词袋"？因为就像把所有词扔进一个袋子里摇一摇——**只记录每个词出现了几次，完全丢掉了词的顺序**。所以 "not good" 和 "good not" 在词袋模型看来是一样的。这是它的一个明显缺点。
>
> 💡 这样得到的向量非常**稀疏（sparse）**：长度几万，但大部分是 0。

### 6.4 线性模型 Linear Model

**思路**：给词典里每个词 i 分配一个**权重（weight）θᵢ**（θ 读作 theta）。
- 正面词 → θ 为正数（如 θ_wonderful = +2）
- 负面词 → θ 为负数（如 θ_terrible = -3）
- 中性词 → θ 接近 0

一共有 N 个权重，每个词一个。

**计算一个得分 z：**

$$z = \theta_1 x_1 + \theta_2 x_2 + \cdots + \theta_N x_N = \sum_{i=1}^{N} \theta_i x_i = \theta^T x$$

用上面的例子：

$$z = \underbrace{\theta_{10287} \times 1}_{\text{grant}} + \underbrace{\theta_{11485} \times 1}_{\text{hugh}} + \cdots + \underbrace{\theta_{23718} \times 2}_{\text{terrible}}$$

没出现的词 xᵢ = 0，乘出来也是 0，所以不影响。

**预测规则：**
- z > 0 → 预测为正面
- z < 0 → 预测为负面

写成公式：

$$\hat{y} = \text{sign}(\theta^T x)$$

其中 sign 函数（符号函数）：z > 0 时为 +1，z < 0 时为 -1，z = 0 时为 0。

> 💡 为什么叫"线性"？因为得分 z 只是各特征乘以权重再相加，没有平方、没有相乘等复杂运算。几何上，θᵀx = 0 定义了一个"超平面"，把空间一分为二：一边是正面，一边是负面。这个超平面叫**决策边界（decision boundary）**，以后会细讲。
>
> 💡 注意这个模型本质上就是在"加权数词"：这正是 6.2 那个"数正面词和负面词"直觉的数学化版本！

---

## 7. 线性代数记号（Slide 14）

课程里线性代数**只用来当记号**，不用怕。

| 概念 | 写法 | 例子 |
|---|---|---|
| 向量 vector | 默认是**列向量**（竖着的） | x = [230.1, 37.8]ᵀ（竖排），x₁ = 230.1 |
| 矩阵 matrix | 二维数组 | A = [[1, 2], [3, 4]]，A₁₁ = 1（第1行第1列） |
| 转置 transpose | xᵀ：把列向量变成行向量 | xᵀ = [230.1, 37.8] |
| 内积 inner product | xᵀy = Σ xᵢyᵢ | 对应位置相乘再加起来 |

**内积小例子：** x = [1, 2, 3]，y = [4, 5, 6]
xᵀy = 1×4 + 2×5 + 3×6 = 4 + 10 + 18 = **32**

所以 θᵀx 就是"每个权重乘以对应特征，再全部加起来"——跟 6.4 的求和是完全一样的东西，只是写法更简洁。

在 Python（numpy）里：`theta @ x` 或 `np.dot(theta, x)`

复习资源：Coursera 的 Matrices and Vectors 讲座、Khan Academy 线性代数。

---

## 8. 如何确定权重 θ？方法一：Naive Bayes（朴素贝叶斯）（Slide 15）

模型形式定好了（θᵀx），但 θ 取什么值？有很多方法，第一种很直观：**根据词频**。

### 8.1 步骤
1. 对每个词 j，算它在**正面评论**中的出现频率：

$$f_{pos,j} = \frac{\text{词 } j \text{ 在所有正面评论中出现的次数}}{\text{所有正面评论的总词数}}$$

2. 同理算在**负面评论**中的频率 f_neg,j
3. 权重先取两者之比：θⱼ = f_pos,j / f_neg,j
4. **取对数（log）**：

$$\theta_j = \log \frac{f_{pos,j}}{f_{neg,j}}$$

5. 再加一个**偏置项（bias / intercept）**：

$$\theta_0 = \log \frac{\text{正面评论数}}{\text{负面评论数}}$$

### 8.2 为什么要取 log？（重点理解）
- **符号变得有意义**：
  - 词在正面评论中更常见 → 比值 > 1 → log > 0 → **正权重** ✅
  - 词在负面评论中更常见 → 比值 < 1 → log < 0 → **负权重** ✅
  - 两边一样常见 → 比值 = 1 → log = 0 → **不影响**（中性词）✅
  - 这正好配合"z > 0 预测正面"的规则！没取 log 之前，比值永远是正数，不能直接这样用。
- **防止极端值"淹没"其他词**（课件原话的意思）：某个罕见词的比值可能是 1000，直接相加会压倒其他所有词；取 log 后变成约 6.9，影响力被压缩了。
- **正负对称**：比值 10 和 1/10 取 log 后是 +2.3 和 -2.3，正负词地位对等。

### 8.3 θ₀ 的含义
θ₀ 反映**先验（prior）**：在还没看评论内容之前，正面评论本身有多常见。如果训练集里正面评论多，θ₀ > 0，模型会稍微偏向预测正面。

### 8.4 【拓展】为什么叫"朴素贝叶斯"？（课件没展开，帮你理解）
根据贝叶斯定理，判断正/负就是比较：

$$\frac{P(\text{pos} \mid \text{评论})}{P(\text{neg} \mid \text{评论})} = \frac{P(\text{pos})}{P(\text{neg})} \times \frac{P(\text{评论} \mid \text{pos})}{P(\text{评论} \mid \text{neg})}$$

**"朴素"（naive）的假设**：假设评论里每个词的出现是**相互独立的**（显然不真实，比如 "hugh" 和 "grant" 经常一起出现，但这个假设让计算变得极简单）。在这个假设下：

$$P(\text{评论} \mid \text{pos}) = \prod_j f_{pos,j}^{\,x_j}$$

两边取 log，乘法变加法：

$$\log\frac{P(\text{pos}\mid\text{评论})}{P(\text{neg}\mid\text{评论})} = \underbrace{\log\frac{\#pos}{\#neg}}_{\theta_0} + \sum_j x_j \underbrace{\log\frac{f_{pos,j}}{f_{neg,j}}}_{\theta_j}$$

这恰好就是 θ₀ + θᵀx！所以课件里"取 log"的做法不是凭空拍脑袋，而是从概率论推出来的。z > 0 就等价于"是正面的概率 > 是负面的概率"。

### 8.5 结果与讨论
- 在**训练数据上**准确率 **99.4%**
- 讲师问："**Too good to be true?**（好得不太真实？）"
  - 是的！因为这是在**训练数据上**测的——模型已经"见过"这些评论了，相当于**考试考原题**。
  - 真正要看的是在**没见过的新数据**上的表现（见第 9 节）。
- 实际中 Naive Bayes 更常被用作**基线（baseline）**：一个简单的参照模型，用来和更复杂的模型比较。如果你的复杂模型连 Naive Bayes 都打不过，那就说明有问题。

---

## 9. 如何确定权重 θ？方法二：优化（Optimisation）（Slide 16）

这是**更通用、整门课会一直用的方法**，也是机器学习"训练"的标准含义。

### 9.1 三个要素
1. **训练数据**：选择 θ，让模型在训练数据上的预测尽可能准确 → 这个过程就叫 **训练（training）**
2. **代价函数 / 损失函数（cost function / loss function）**：一个衡量"预测错得有多离谱"的数学公式。预测越错，cost 越大。
3. **优化算法（optimisation algorithm）**：自动调整 θ，找到让 cost **最小**的那组值（例如以后会学的梯度下降 gradient descent）

> 💡 比喻：cost function 是"考试扣分规则"，优化算法是"根据扣分情况不断调整答题策略的过程"。

### 9.2 结果与讨论
- 用 **Logistic Regression（逻辑回归）** 的 cost function（后面课程会详细讲），训练数据上准确率 **100%**
- 讲师问了两个问题：
  1. **"Is that what we expect?"（这在意料之中吗？）**
     - 是意料之中的。词典有**几万个词 = 几万个参数 θ**，但评论只有大约 2000 条。**参数比数据点还多**，模型自由度极大，几乎总能找到一组 θ 把训练数据全部分对——甚至可以"记住"每条评论的个别怪词。
  2. **"Is this a good test of prediction performance?"（这是衡量预测性能的好方法吗？）**
     - **不是！** 这就是**过拟合（overfitting）** 的风险：模型把训练数据"背"下来了，但对新数据不一定好。
     - 正确做法：把数据分成**训练集（training set）** 和 **测试集（test set）**，用训练集训练，用**模型从未见过的测试集**评估。下一节代码就是这么做的。

> 💡 这是本讲最重要的"坑"之一，考试/作业里经常会考到：**永远不要只用训练数据的准确率来评估模型。**

---

## 10. 动手试试：代码逐行讲解（Slide 17）

### 10.1 准备数据
1. 下载：`http://www.cs.cornell.edu/people/pabo/movie-review-data/review_polarity.tar.gz`
2. 解压后得到文件夹 `txt_sentoken`，里面有 `pos`（正面）和 `neg`（负面）两个子文件夹
3. 安装库：`pip install scikit-learn`

### 10.2 代码（已修正课件中的 PDF 排版引号问题，可直接运行）

```python
# ① 读取数据
from sklearn.datasets import load_files
d = load_files("txt_sentoken", shuffle=False)
x = d.data     # 每条评论的原始文本（列表）
y = d.target   # 每条评论的标签：子文件夹 neg → 0，pos → 1（按字母序）

# ② 特征提取器：把文本变成数字向量
from sklearn.feature_extraction.text import TfidfVectorizer
vectorizer = TfidfVectorizer(stop_words='english', max_df=0.2)

# ③ 划分训练集 / 测试集
from sklearn.model_selection import train_test_split
xtrain, xtest, ytrain, ytest = train_test_split(x, y, test_size=0.2)

# ④ 把文本转成特征矩阵
Xtrain = vectorizer.fit_transform(xtrain)
Xtest  = vectorizer.transform(xtest)

# ⑤ 训练逻辑回归模型
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(Xtrain, ytrain)

# ⑥ 在测试集上预测并评估
preds = model.predict(Xtest)
from sklearn.metrics import classification_report
print(classification_report(ytest, preds))
```

> ⚠️ 课件 PDF 里的引号是弯引号 `”` `’`，还有 `max\_df` 带反斜杠，直接复制会报错。上面已改为正常的 `"`、`'`、`max_df`。

### 10.3 逐步解释

**① `load_files`**
自动读取文件夹结构：每个子文件夹名就是一个类别。`d.data` 是文本列表，`d.target` 是对应的数字标签。`shuffle=False` 表示不打乱顺序。

**② `TfidfVectorizer`** —— Bag of Words 的升级版
- 普通词袋只数次数；**TF-IDF** 会给每个词的次数再乘一个"稀有度"权重：
  - **TF（Term Frequency，词频）**：这个词在这条评论里出现多少次
  - **IDF（Inverse Document Frequency，逆文档频率）**：这个词在**多少条评论里**出现过——出现在越多评论里，IDF 越小，说明它越"普通"、越没区分度
  - TF-IDF = TF × IDF：**在这条评论里常出现、但在其他评论里少见的词**，得分最高
- `stop_words='english'`：去掉 sklearn 内置的英文停用词（the, and, of……），就是 6.3 的第 1 步
- `max_df=0.2`：如果一个词出现在**超过 20% 的评论**里，就忽略它（太常见，比如 "movie"、"film"，对区分好坏没帮助）
- 注意：sklearn 默认**不做 stemming**

**③ `train_test_split(x, y, test_size=0.2)`**
随机把数据分成两部分：**80% 训练，20% 测试**。这正是第 9 节说的"要在没见过的数据上评估"。
（每次运行划分是随机的，所以结果会稍有不同；想要可复现可以加 `random_state=0`。）

**④ `fit_transform` vs `transform` —— 非常重要的细节！**
- `vectorizer.fit_transform(xtrain)`：
  - **fit**：从**训练集**里学习词典（有哪些词）和每个词的 IDF 值
  - **transform**：把训练文本转成数字矩阵
- `vectorizer.transform(xtest)`：只 transform，**用训练集学到的词典**来转换测试集，**不重新 fit**
- **为什么测试集不能 fit？** 因为测试集要模拟"未来的新数据"，你在建模时是不应该知道的。如果用测试集建词典，测试集的信息就"泄露"进了模型，评估结果会偏乐观。这叫 **数据泄露（data leakage）**。
- 结果 `Xtrain` 是一个大矩阵：每行一条评论，每列一个词。（是稀疏矩阵，节省内存）

**⑤ `LogisticRegression().fit(Xtrain, ytrain)`**
这就是第 9 节的"优化"方法：定义逻辑回归的 cost function，并用优化算法找到最优的 θ。训练完后，θ 存在 `model.coef_` 里（每个词一个权重），θ₀ 存在 `model.intercept_` 里。
> 注意：虽然名字叫"回归"，**Logistic Regression 其实是一个分类模型**！这是一个经典的命名陷阱。

**⑥ `classification_report`**
输出每个类别的几个指标：
| 指标 | 含义 | 通俗理解 |
|---|---|---|
| **precision（精确率）** | 预测为正面的里，真正是正面的比例 | "我说是好评的，有多少真的是好评？" |
| **recall（召回率）** | 真正是正面的里，被成功预测出来的比例 | "所有真好评里，我找出了多少？" |
| **f1-score** | precision 和 recall 的调和平均 | 两者的综合 |
| **support** | 该类别在测试集中的样本数 | |
| **accuracy** | 总体预测正确的比例 | |

测试集上的准确率一般在 **80%–85% 左右**——明显低于训练集的 100%，这就印证了第 9 节的讨论。

> 💡 小练习：自己跑一下，然后再加一行 `print(classification_report(ytrain, model.predict(Xtrain)))` 看看训练集上的结果，对比两者差距。这种"对比 + 解释"正是作业想要你写的东西。

---

## 11. 本讲核心思想总结（Slide 18）

整门课都围绕这五个环节展开：

| 环节 | 在本例中是什么 | 以后会学的其他选择 |
|---|---|---|
| **1. 特征工程 Feature engineering** | Bag of Words / TF-IDF，把文字变成数字数组 | 很多其他方法（如 word embeddings） |
| **2. 模型选择 Model selection** | 线性模型 ŷ = sign(θᵀx) | kNN、决策树、神经网络、核方法…… |
| **3. 代价函数选择 Cost function selection** | Logistic cost function | 平方误差、hinge loss…… |
| **4. 优化 Optimisation** | 选 θ 使 cost 最小 | 梯度下降等 |
| **5. 性能评估 Evaluating performance** | 训练/测试集划分 + classification report | 交叉验证（cross-validation）等 |

---

## 12. 术语表（中英对照）

| English | 中文 |
|---|---|
| supervised / unsupervised learning | 监督 / 无监督学习 |
| classification / regression | 分类 / 回归 |
| label, target | 标签，目标值 |
| feature, feature vector | 特征，特征向量 |
| training set / test set | 训练集 / 测试集 |
| classifier, hypothesis h(x) | 分类器，假设函数 |
| prediction ŷ | 预测值 |
| stop words | 停用词 |
| stemming | 词干提取 |
| token | 词元 |
| bag of words | 词袋模型 |
| TF-IDF | 词频-逆文档频率 |
| weight / parameter θ | 权重 / 参数 |
| bias / intercept θ₀ | 偏置 / 截距 |
| linear model | 线性模型 |
| decision boundary | 决策边界 |
| transpose, inner product | 转置，内积 |
| Naive Bayes | 朴素贝叶斯 |
| prior | 先验 |
| baseline | 基线模型 |
| cost / loss function | 代价 / 损失函数 |
| optimisation | 优化 |
| logistic regression | 逻辑回归（是分类模型！） |
| overfitting | 过拟合 |
| data leakage | 数据泄露 |
| precision / recall / F1 | 精确率 / 召回率 / F1 分数 |
| correlation vs causation | 相关性 vs 因果性 |

---

## 13. 自测问题（检查自己是否理解）

1. 预测明天的降雨量（毫米）是分类还是回归？预测明天是否下雨呢？
2. 为什么机器学习模型的输入必须是数字向量？文字是怎么变成数字的？
3. Bag of Words 丢掉了什么信息？举一个会因此判断错误的句子。
4. 在 Naive Bayes 中，如果某个词在正面和负面评论中出现频率完全一样，它的权重 θ 是多少？这合理吗？
5. 为什么训练集上 100% 准确率不值得高兴？
6. 为什么测试集只能用 `transform`，不能用 `fit_transform`？
7. 如果你只从大学生那里收集影评来训练模型，可能会出什么问题？

<details>
<summary>点击查看参考答案</summary>

1. 降雨量是连续数值 → 回归；是否下雨是两个类别 → 分类。
2. 模型是数学函数，只能处理数字。本讲用 Bag of Words / TF-IDF：建立词典，统计每个词在评论中出现的次数（或 TF-IDF 值）。
3. 丢掉了词的顺序。例如 "not good, actually terrible" 和 "not terrible, actually good" 词袋几乎一样，但意思相反。
4. θ = log(1) = 0，不影响得分。合理，因为这个词对区分正负没有帮助。
5. 因为模型可能只是"记住"了训练数据（过拟合）；参数数量远多于样本数时尤其容易。要看它在没见过的测试数据上的表现。
6. 测试集模拟未来的未知数据，如果用它来 fit（建词典、算 IDF），测试集信息泄露进模型，评估结果会过于乐观（data leakage）。
7. 数据不具代表性：学生的用词习惯、喜欢的电影类型和其他人群不同，模型用到其他人群上效果可能变差。
</details>
