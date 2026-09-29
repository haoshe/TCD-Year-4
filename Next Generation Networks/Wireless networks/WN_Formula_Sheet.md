# 无线网络 — 公式速查 & 总览

## 各章笔记

- [01 Channel Impairments and Mitigation](<01 Channel Impairments and Mitigation/WN_Channel_Impairments_Chapter_Notes.md>)
- [02 Overview of Wireless Networks](<02 Overview of Wireless Networks/WN_Overview_Chapter_Notes.md>)
- [03 LTE LTE-A 5G NR](<03 LTE LTE-A 5G NR/WN_LTE_5GNR_Chapter_Notes.md>)
- [04 WiFi IEEE 802.11](<04 WiFi IEEE 802.11/WN_WiFi_Chapter_Notes.md>)
- [05 HetNet mmWave THz](<05 HetNet mmWave THz/WN_HetNet_mmWave_THz_Chapter_Notes.md>)

---

## 全课程公式速查

| 公式 | 含义 |
|---|---|
| $\lambda = c/f$，$c = 3\times10^8$ m/s | 波长；空间分集天线间距 ≥ λ/2 |
| $P_r \propto 1/d^2$（自由空间），一般 $1/d^n$ | 路径损耗 |
| $s = t_b / t_c$ | DSSS 扩频因子 |
| $C = B\log_2\left(1+\dfrac{S}{I+N}\right)$，$N = N_0B$ | 香农容量 |
| $B\to\infty$：$C \to 1.44\,S/N_0$ | 带宽无限时容量有上限 |
| $f_d = vf/c$，$T_c \approx 0.423/f_d$ | 多普勒频移 / 相干时间 |
| RB = 12 × 15 kHz = 180 kHz × 0.5 ms；1 帧 = 10 ms = 20 时隙；TTI = 1 ms | LTE 资源网格 |
| 每符号比特数 = $\log_2 M$ | BPSK 1 / QPSK 2 / 16QAM 4 / 64QAM 6 / 256QAM 8 / 4096QAM 12 |
| 频谱效率 = 流数 × $\log_2 M$ × $R_c$ | bps/Hz |
| Max C/I：$\arg\max d_n(t)$；PF：$\arg\max d_n(t)/r_n(t)$ | 调度 |
| $\text{ASE} = \dfrac{\text{Throughput}}{\text{Area}\cdot\text{Bandwidth}}$ | 面积频谱效率 |
| 802.11a：$f_c = 5000 + 5n$ MHz；802.11b：$f_c = 2407 + 5n$ MHz (n=1–13) | Wi-Fi 信道中心频率 |
| SCS (5G) $= 15\times2^\mu$ kHz | 5G numerology |

**一句话记忆链**
- 2G TDMA → 3G CDMA → 4G OFDMA → 5G 灵活 numerology 的 CP-OFDM
- 频率 ↑ → 路径损耗 ↑、覆盖 ↓、易阻挡 ↑、带宽 ↑、天线 ↓
- 分集 = 求稳；复用 = 求快；波束成形 = 求准
- RR 公平、Max C/I 吞吐、PF 折中
