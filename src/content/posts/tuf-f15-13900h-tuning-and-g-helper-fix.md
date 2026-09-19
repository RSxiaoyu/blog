---
title: 华硕天选4 (FX507VV) i9-13900H 调优定案与 G-Helper 功耗墙 Bug 根因复盘
published: 2026-09-20
description: 从 G-Helper #6039 源码级误判排查，到 AC/DC Loadline 110→135 六点拓扑实测定案，终结 i9-13900H 性能调优的玄学猜测。
tags: [Hardware, Intel, Undervolt, ASUS, OpenSource]
category: Hardware
draft: false
---

## 1. G-Helper 功耗滑块锁定 90W 根因与修复 (Issue #6039)

### 现象
华硕天选4 酷睿版（FX507VV，搭 Intel Core i9-13900H）在 G-Helper 的“风扇+功耗”控制台中，CPU 功耗滑块（PL1/PL2/Total）上限被强行锁定在 **90W**，无法使用 Intel 默认的 115W~150W 调节区间。

### 源码追查
在 G-Helper 源码 `app/AsusACPI.cs:363`：

```csharp
if (AppConfig.IsCPULight())
{
    MaxTotal = 90; // 判定为低功耗机型时强制截断为 90W
}
```

追查 `app/AppConfig.cs:652` 中的判定逻辑：

```csharp
public static bool IsCPULight()
{
    return ... || ContainsModel("FA507X") || ...;
}
```

- 华硕天选4 酷睿版（FX507VV）因共用模具，主板 WMI 模型字符串为：`ASUS TUF Gaming F15 FX507VV_FA507XV`。
- `ContainsModel("FA507X")` 命中了字符串尾部的 `_FA507XV`，将 Intel 13900H 误判为 AMD 锐龙机型（FA507X），功耗上限被直接锁死在 90W。

### 修复与上游合并
向官方提交排查报告（[Issue #6039](https://github.com/seerge/g-helper/issues/6039)），作者采纳后将匹配规则精确化，排除酷睿版本后缀。代码已合并进官方 Release（v0.284+），原生恢复 90W~150W 自由调节。

---

## 2. 硬件功耗仲裁机制：MSR vs MMIO

在华硕主板上调功耗需注意底层双重寄存器约束：

$$\text{实际执行功耗} = \min(\text{MSR}, \text{MMIO})$$

- **MSR**：BIOS 或调节软件下发的静态 TDP。
- **MMIO**：主板 EC 芯片硬件控制。华硕在非奥创托管状态下，默认将 EC MMIO 静态焊死在 90.0W。
- **结论**：若不解除 MMIO 限制或保持动态同步，BIOS 中设置 100W+ 也会在长时稳态被 EC 强行压制在 90W。

---

## 3. AC/DC Loadline 六点拓扑实测定案

在 FX507VV（BIOS 332 解锁版，PTM7950 导热，尾部垫高）实机环境，绘制 110 至 135 完整连续负载测试曲线，寻找物理甜点：

| AC/DC 设定 | Cinebench R23 多核 | 大核交付率 | 小核交付率 | 稳态功耗 / 温度 | 状态判定 |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **110** | 15,863 pts | 91.2% | 85.0% | 86.4W / 73.5°C | **严重时钟拉伸**：IA CEP 介入降效 |
| **115** | 16,137 pts | 93.8% | 87.2% | 87.1W / 74.2°C | **时钟拉伸区** |
| **120** | 17,253 pts | 98.1% | 92.5% | 88.5W / 75.8°C | 爬坡过渡区 |
| **125** | 17,345 pts | 99.1% | 97.5% | 89.2W / 76.3°C | 接近满血 |
| **130** | **17,982 pts** | **99.7%** | **99.2%** | **90.2W / 77.0°C** | **【全局绝对峰值】** 时钟拉伸归零 |
| **135** | 17,718 pts | 99.7% | 99.2% | 90.0W / 78.4°C | **进入下降沿**：撞 90W 功耗墙降频 20MHz |

### 物理机理结论
1. **AC 必须等于 DC**：两者不匹配会导致 CPU VID 读数紊乱与功耗估算失准。
2. **低于 120 属于假性降压**：电压虽低，但触发了 IA CEP 时钟拉伸，有效频率大幅受损。
3. **130 为硬件体质极限**：130 档时钟拉伸完全消除；升至 135 虽无拉伸，但更高电压在 90W 功耗墙约束下迫使 CPU 调低全核倍频，跑分净降 264 分。

---

## 4. 最终调优参数归档

```ini
[CPU Power & Voltage]
AC Loadline = 130
DC Loadline = 130
VccIn Aux Icc Max = MAX          ; 严禁维持默认 132，否则导致 VID 暴涨积热
IA ICC Unlimited mode = Enabled
IA CEP / GT CEP = Disabled
PL1 (Sustained) = 90W
PL2 (Burst) = 115W

[Core Ratios]
P-Cores = 54-54-51-51-49-49-49-49 (Default)
E-Cores = 41x4 / 39x4 (Default)

[GPU (RTX 4060 Laptop)]
Core Clock Offset = +250 MHz (Boost 锁在 2,730 MHz 电压墙)
Memory Clock Offset = +500 MHz
```

**实测基准验证**：Cinebench R23 单核 **2,051 pts**（5.4GHz 稳态 72W / 74°C），多核 **17,982 pts**（稳态 77°C / 90.2W），无报错、无蓝屏、无降频死锁。
