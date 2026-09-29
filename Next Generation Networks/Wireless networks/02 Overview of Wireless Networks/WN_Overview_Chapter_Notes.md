# Lecture 2 — 无线网络概览

> 课程：Next Generation Networks (EEU44C04 / CS4031 / CS7NS3 / EEP55C27) · 讲师：Nicola Marchetti
> 对应课件：`NM 1_02_Overview_of_Wireless_Networks.pdf`
>
> 标记：🔍 难点精讲 · 📝 课堂例题 · ⚠️ 易错点 · 公式汇总见上级目录 `WN_Formula_Sheet.md`

---

## ✅ 必背清单（只含课件内容，考试要背的就这些）

> 下面的 🔍 方框是帮你**理解**的，不用背。复习时先把这个清单过熟。

1. **Mobile vs Nomadic**：Mobile = **无缝漫游**（边走边用，蜂窝/LTE/5G、802.16、卫星）；Nomadic = 换接入点但**不需要无缝漫游**（WLAN 802.11、Bluetooth、热点）。
2. **OSI 各层在无线中的职责（关键词）**
   - PHY：对抗损伤（coding、diversity、power control、waveform selection）；调制、选频、信号检测；加密
   - Data link：介质访问控制 (MAC)、帧同步、差错控制
   - Network：寻址 + 为移动性重定向、**handover**、ad-hoc 多跳路由、设备定位
   - Transport：可靠端到端传输、流量/拥塞控制
   - Application：适配移动设备限制（小屏、低功耗）、位置服务、无线 web
3. **频段量级**：电视/广播 **< 1 GHz**；5G 3.4–3.8 GHz；WLAN 2.4 GHz (b/g) 和 5 GHz (a)；Bluetooth 2.45 GHz。
4. **为什么电视用 < 1 GHz**：低频随距离衰减更小 → 适合远距离传输。
5. **监管机构**：ITU-R（国际，标准化 + 频率规划，定期开 **World Radio Conference**）；FCC（美国）；ComReg（爱尔兰）。
6. **Harmonization**：难（遗留系统多），但必要 → ① 避免干扰 / 互操作；② 规模经济。
7. **Dynamic spectrum access**：频谱利用率低，但很难从持牌者手中收回；反例 = 模拟转数字电视后的 **TV 频段 re-farming**；只要不干扰持牌者就可用授权频段；靠 Cognitive radio、SON、Zero Touch Networks 实现。
8. **有线 vs 无线互补**：光纤到不了所有地方但带宽极大；无线几乎哪里都能到但带宽受限、易受损伤。
9. **高速用户（1000 km/h）的功率分配**：选**平均分配**，因为信道相干时间 < CSI 反馈 + 传输所需时间，CSI 过时了。

## 目录

1. 2.1 无线网络类型
2. 2.2 OSI 模型在无线中的职责
3. 2.3 频谱分配
4. 2.4 监管机构与 Harmonization
5. 2.5 动态频谱接入
6. 📝 课堂例题
7. 🧪 自测题

---

### 2.1 无线网络类型

| | **Mobile（移动）** | **Nomadic（游牧）** |
|---|---|---|
| 特点 | **无缝漫游**（移动中不断线） | 用户会更换接入点，但**不需要无缝漫游** |
| 应用 | 无线电话、移动数据（邮件、网页、SMS、视频、社交）、车联网、位置服务 | WLAN、热点、无线 ISP |
| 技术 | 蜂窝/PCS（GSM、GPRS、UMTS、LTE、5G、B5G/6G）、IEEE 802.16、陆地移动无线电、卫星 | IEEE 802.11、Bluetooth |

> ⚠️ Nomadic ≠ Mobile：Nomadic 是"在 A 点用、关掉、到 B 点再用"；Mobile 是"边走边用，连接不断"。

### 2.2 OSI 模型在无线中的职责

| 层 | 无线中的主要功能 |
|---|---|
| Application | 针对移动设备限制（小屏、低功耗）调优；位置感知服务；无线 web 访问 |
| Transport | 可靠的端到端传输；流量与拥塞控制 |
| Network | 全网寻址、为支持移动性而重定向；**handover（切换）**；ad-hoc 多跳路由；设备定位 |
| Data link | **介质访问控制 (MAC)**；帧同步；可靠的点到点/点到多点连接；差错控制 |
| Physical | 对抗损伤（编码、分集、功率控制、波形选择）；调制、选频、信号检测；加密 |

> 🔍 **Network 行里的 "ad-hoc 多跳路由" 是什么？**
> **Ad-hoc** = **自组织网络 / 无中心网络**（拉丁语，"为特定目的临时搭建的"）。
> 1. **普通 WiFi（infrastructure mode，基础设施模式）**：手机 → 路由器 (Access Point) → 电脑，所有通信都要经过这个"中心"。蜂窝网络同理，经过基站。
> 2. **Ad-hoc 没有中心**：没有路由器或基站，设备之间直接连接，凑到一起就能组网，不需要事先搭建设施。
> 3. **多跳路由 (multi-hop routing)**：A 想发给 C，但信号到不了；B 在中间，就走 A → B → C。每转发一次叫一**跳 (hop)**，这里是 2 跳。没有路由器决定走哪条路，所以**每台设备既是终端又是路由器**。这就是课件第 8 页 "Routing through multiple hops in ad-hoc networks"。
> 4. **联系**：2.1 节 Mobile 里的 **Vehicular ad hoc networks（车联网，VANET）** 就是车和车之间直接通信的 ad-hoc 网络。其他例子：战场、救灾现场、蓝牙直连（了解即可）。
>
> **一句话**：ad-hoc = 没有中心设备，设备之间直接组网，数据靠其他设备一跳一跳转发。

### 2.3 频谱分配

**频谱分配表（美国）**：非常乱——历史遗留系统太多。

> 🔍 **"数字 3" 是怎么回事？（3 kHz、3 MHz、3 GHz …）**
> 因为 $\lambda = c/f$，而 $c = 3\times10^8$ m/s。频段边界选在 $3\times10^n$ Hz，对应的**波长正好是 10 的整数次幂**：
> - 3 MHz → 100 m；30 MHz → 10 m；300 MHz → 1 m；3 GHz → 10 cm；30 GHz → 1 cm；300 GHz → 1 mm
>
> 所以 ITU 的频段（HF、VHF、UHF、SHF、EHF…）按**波长的十倍关系**划分。这也解释了"毫米波 (mmWave)" = 30–300 GHz（波长 1–10 mm）。

> 🔍 **HF、VHF、UHF、SHF、EHF 要背吗？**
> 课件文字里只出现 **VHF / UHF**（第 12 页 Broadcast TV），其余三个**课件没讲，了解即可**。
> 1. **VHF 和 UHF：认得出来就行**
>    - VHF = Very High Frequency（甚高频），30–300 MHz → 电视用 54–88、174–216 MHz
>    - UHF = Ultra High Frequency（特高频），300 MHz–3 GHz → 电视用 470–806 MHz
>    - 两段都 < 1 GHz，对应 Q1 的答案"电视用 < 1 GHz"
> 2. **其余三个看一眼就好**
>    - HF = High Frequency（高频），3–30 MHz
>    - SHF = Super High Frequency（超高频），3–30 GHz
>    - EHF = Extremely High Frequency（极高频），30–300 GHz = 毫米波 (mmWave)
>
> **记规律比背名字有用**：每个频段边界都是 3×10ⁿ Hz，往上一档频率 ×10，名字也升一级：High → Very → Ultra → Super → Extremely。

**典型频段（记住量级）**
| 业务 | 频段 |
|---|---|
| FM 广播 | 88–108 MHz |
| 广播电视 VHF / UHF | 54–88、174–216 MHz / 470–806 MHz |
| 2G | 800–900 MHz |
| PCS | 1.85–1.99 GHz |
| 3G | 746–794 MHz、1.7–1.85 GHz、2.5–2.7 GHz |
| 4G | 800 MHz、1.8 GHz |
| 5G | 3.4–3.8 GHz |
| WLAN 802.11b/g | 2.4 GHz |
| WLAN 802.11a | 5 GHz |
| Bluetooth | 2.45 GHz |
| LMDS | 27.5–31.3 GHz |

### 2.4 监管机构与 Harmonization

**监管机构**：ITU-R（国际，负责标准化和频率规划，定期召开 **World Radio Conference**）；FCC（美国）；**ComReg**（爱尔兰）。

**Harmonization（全球频谱协调）**：很难（遗留系统多），但必要——① 避免干扰/互操作；② 规模经济。

### 2.5 动态频谱接入（Dynamic spectrum access）

- 频谱利用率其实很低，但很难从持牌者手中收回频谱。
- **反例**：模拟电视转数字电视后，TV 频段被 **re-farming（频谱重耕）**——数字电视效率高，腾出的频段被收回并重新分配给 4G 等移动通信。能成功是因为技术进步让原用户用更少频谱做同样的事，是"以旧换新"而不是硬抢。
- 动态频谱接入：只要不干扰持牌者，就可以在授权频段中使用。**Cognitive radio（认知无线电）**、**SON（自组织网络）**、**Zero Touch Networks** 等自治网络在尝试实现这一点。

### 📝 课堂例题（Lecture 2）

**Q1：哪个频段最适合（实际也用于）电视广播？** → **< 1 GHz**。低频随距离衰减更小，适合远距离广播。

**Q2：评论有线和无线系统的互补性。**
→ 光纤不是哪里都有，但有的地方带宽极大；无线接入几乎哪里都能到，但带宽非常受限，而且容易受各种损伤影响。二者互补。

**Q3：用户以 1000 km/h 移动（飞机），基站多天线，哪种下行功率分配最合理？**
(i) 平均分给各天线；(ii) 按手机反馈的 CSI 分配；(iii) 信道差时不给功率
→ **(i)**。用户移动太快时，**信道相干时间比"反馈 + 传输"所需时间还短**，基站拿不到最新的 CSI，所以依赖 CSI 的方案反而没用。

> 🔍 **算一下就明白了**
> $v = 1000$ km/h ≈ 278 m/s，设 $f = 2$ GHz：
> 多普勒频移 $f_d = \dfrac{v f}{c} = \dfrac{278 \times 2\times10^9}{3\times10^8} \approx 1850$ Hz
> 相干时间 $T_c \approx \dfrac{0.423}{f_d} \approx 0.23$ ms
> 而 LTE 的一个 TTI 就是 1 ms，CSI 反馈回路通常要好几 ms → 等基站收到反馈，信道早就变了。
> 在这种情况下，"不知道信道就平均分配"是最稳妥的；选项 (iii) 同样需要知道信道好坏，也不可行。

> 🔍 **上面 $T_c \approx 0.423/f_d$ 里的 0.423 从哪来？**（课件没讲，了解即可，不用背这个数）
> 1. **符号**：$f_d$ = 多普勒频移 (Doppler shift)，描述信道变化有多快，越快越大；$T_c$ = 相干时间 (coherence time)，信道基本不变的时长。
> 2. **核心关系**：$T_c \propto 1/f_d$，信道变得越快，保持不变的时间越短。最粗略的估算是 $T_c \approx 1/f_d$。
> 3. **系数取决于"不变"的定义**：要求前后相关性 ≥ 0.5 时 $T_c \approx \dfrac{9}{16\pi f_d} \approx \dfrac{0.179}{f_d}$（偏保守）。Rappaport 教材取 0.179 和 1 的**几何平均 (geometric mean)**：$\sqrt{9/(16\pi)} \approx 0.423$。这是工程折中取值，不是物理常数。
> 4. **代入 $f_d \approx 1850$ Hz**：$1/f_d \approx 0.54$ ms；$0.423/f_d \approx 0.23$ ms；$0.179/f_d \approx 0.10$ ms。三个结果都远小于 1 ms TTI，更小于 CSI 反馈的几 ms。
>
> **结论**：选哪个系数都不影响答案，结论都是"CSI 过时，只能平均分配功率"。只需记住 **$T_c \propto 1/f_d$**，速度越快，相干时间越短。

---

## 🧪 自测题

> 先自己答，再点开看答案。

**1.** Mobile 和 Nomadic 最关键的区别是什么？各举一个技术例子。
<details><summary>答案</summary>Mobile 需要无缝漫游（如 LTE/5G）；Nomadic 换接入点但不需要无缝漫游（如 WiFi 802.11、Bluetooth）。</details>

**2.** Handover 属于 OSI 哪一层？介质访问控制 (MAC) 呢？加密呢？
<details><summary>答案</summary>Handover → Network 层；MAC → Data link 层；加密 → 课件把它放在 Physical 层。</details>

**3.** PHY 层用哪些手段对抗无线信道的损伤？
<details><summary>答案</summary>Coding、diversity、power control、waveform selection。</details>

**4.** 电视广播为什么用 < 1 GHz？
<details><summary>答案</summary>低频随距离衰减更小，适合远距离传输。</details>

**5.** WiFi 802.11b/g、802.11a、Bluetooth、5G 分别大约在哪个频段？
<details><summary>答案</summary>2.4 GHz；5 GHz；2.45 GHz；3.4–3.8 GHz。</details>

**6.** 全球频谱 Harmonization 为什么难？为什么又必须做？
<details><summary>答案</summary>难：遗留系统太多。必须：① 避免干扰 / 互操作；② 规模经济。</details>

**7.** 什么是 dynamic spectrum access？课件举的"成功收回频谱"的反例是什么？
<details><summary>答案</summary>只要不干扰持牌者，就可以在授权频段里使用。反例：模拟电视转数字电视后 TV 频段被 re-farming。实现手段：Cognitive radio、SON、Zero Touch Networks。</details>

**8.** 为什么说有线和无线是互补的？
<details><summary>答案</summary>光纤带宽极大但到不了所有地方；无线几乎处处可达，但带宽受限、易受损伤。</details>

**9.** 用户在飞机上（1000 km/h），基站多天线，下行功率该怎么分？为什么？
<details><summary>答案</summary>平均分给各天线。信道相干时间比 CSI 反馈 + 传输的时间还短，基站拿到的 CSI 已经过时，依赖 CSI 的方案都没用。</details>
