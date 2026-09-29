# Lec4 课堂练习答案：Propositional Logic

> Formal Verification CSU44004/CSU55004。每题标注了对应的 slide 页码。

## 目录
- [Slide 2 · Q1：间接证明法 (p→(q∨r)), (q→r), p ⊨ (¬r→s)](#q1)
- [Slide 2 · Q3 Bonus：(p → ¬q) → (q ∨ ¬p) 的性质](#q3)
- [待做题](#todo)

---

## 基础回顾：间接证明法 (indirect proof / proof by contradiction)

检查 A1, …, An ⊨ B 的步骤：
1. 假设**所有前提为 T**，并且**结论为 F**。
2. 推出**矛盾**：说明这样的赋值不存在，所以 ⊨ **成立** ✔。
3. **没有矛盾**：找到的赋值就是反例 (counterexample)，所以 ⊨ **不成立** ✘。

> 💡 技巧：**先从结论下手**。结论为 F 通常能直接锁定几个变量的值，然后顺着前提往下推。

---

<a id="q1"></a>
## Q1（Slide 2，课件解答在 Slide 3）：(p → (q ∨ r)), (q → r), p ⊨ (¬r → s)

### 解法：间接证明（反证法）
1. **假设结论为 F**：¬r → s = F。只有"T → F"才为 F，所以 ¬r = T，s = F，得到 **r = F，s = F**。
2. **前提 p = T**：得到 **p = T**。
3. **前提 q → r = T**：r = F，如果 q = T 就会变成 T → F = F，所以 **q = F**。
4. **检查前提 p → (q ∨ r)**：q ∨ r = F ∨ F = F，所以 p → (q ∨ r) = T → F = **F**。
5. **矛盾**：这个前提假设为 T，算出来却是 F。
6. **结论**：不存在"前提全真、结论为假"的赋值，所以 **蕴含成立 ✔**。

### 🔍 课件的写法（Slide 3）：从前提正向推
1. 假设三个前提都为 T。p = T，所以 q ∨ r = T。
2. 分情况：q = T 时，由 q → r 得 r = T；q = F 时，由 q ∨ r = T 得 r = T。两种情况都得到 **r = T**。
3. 于是 ¬r = F，结论 ¬r → s = F → s = **T**（前件为假，蕴含自动为真），和 s 取什么无关。
4. 所以蕴含成立 ✔。

> 📝 两种写法结论一样。题目要求 "indirect proof" 时，**写反证法那一版**（假设结论为假，推出矛盾）更贴合题意。

---

<a id="q3"></a>
## Q3 Bonus（Slide 2）：(p → ¬q) → (q ∨ ¬p) 有哪些性质？

选项：A) Satisfiable　B) Insatisfiable　C) Valid　D) Invalid

### 解法：真值表 (truth table)
只有 2 个变量，一共 4 行，用真值表最快。和 [Lec3 Q1](Lec3-answers.md#q1) 是同一个公式。

| p | q | ¬q | p→¬q | ¬p | q∨¬p | 整体 |
|---|---|---|---|---|---|---|
| T | T | F | F | F | T | **T** |
| T | F | T | T | F | F | **F** |
| F | T | F | T | T | T | **T** |
| F | F | T | T | T | T | **T** |

- **A) Satisfiable ✔**：第 1 行为 T
- **B) Insatisfiable ✘**：有为 T 的行
- **C) Valid ✘**：第 2 行 (p=T, q=F) 为 F
- **D) Invalid**：看 "invalid" 怎么定义
  - 按 Lec3 slide 3 的定义（invalid = contradiction，每一行都为 F）：**✘**
  - 按常见的定义（invalid = not valid，至少一行为 F）：**✔**

> ⚠ 考试写 D 时要说明用的是哪个定义，最稳妥的写法："Not valid, since p=T, q=F makes it false; but not a contradiction, since p=T, q=T makes it true."

> 💡 捷径：判断这四个选项，只需要找到**一行 T 和一行 F**，不用把整张表算完。

---

<a id="todo"></a>
## 待做题
- [ ] Q2（Slide 2，解答在 Slide 4）：用真值表判断 (p→q), (q→r), p ⊨ r 是否成立
