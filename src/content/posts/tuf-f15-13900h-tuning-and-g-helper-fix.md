---
title: 华硕天选4 (FX507VV) 软硬件全栈调优定案：i9-13900H/4060/DDR5 极限压榨与开源复盘
published: 2026-09-20
description: 涵盖 CPU AC/DC Loadline 拓扑实测、GPU +250/+1200 超频、内存 5400 C36 压参、G-Helper #5895/#6039 开源排障与 Win11 系统底噪治理的终态基准。
tags: [Hardware, Intel, ASUS, Overclock, Undervolt, OpenSource, Windows]
category: Hardware
draft: false
---

## 1. 机器档案与运行基线

| 硬件维度 | 型号 / 规格 | 调优状态与散热环境 |
| :--- | :--- | :--- |
| **机型 / 散热** | ASUS TUF Gaming F15 (FX507VV, 2023) | CPU/GPU 更换 PTM7950 相变片 + 导热凝胶，机身尾部垫高进风 |
| **BIOS 固件** | FX507VV.332 | 解锁全功能高级菜单（支持按地址 / 菜单直接注入参数） |
| **CPU** | Intel Core i9-13900H (6P + 8E, 20 线程) | AC/DC Loadline 双 130，CEP 关闭，PL1 90W / PL2 115W |
| **GPU** | NVIDIA GeForce RTX 4060 Laptop 8GB (SK Hynix) | 核心 +250 MHz，显存 +1200 MHz，维持 140W Dynamic Boost |
| **内存** | 美光 16GB (8GB×2) DDR5-4800 (D8BNK 颗粒) | 超频至 5400 MHz @ C36-39-39-76 CR2，tREFI 翻倍至 9360 |
| **固态硬盘** | 西数 WD PC SN560 1TB NVMe SSD | TRIM 正常，C 盘维持 60GB 以上写入缓冲空间 |
| **显示规格** | 15.6 英寸 IPS (2560 × 1600, 16:10) @ 240Hz | G-Sync 开启 |
| **系统环境** | Windows 11 专业版 Insider (Build 26300.8935) | 默认 pwsh，VBS 开启 / HVCI 关闭，全局字体无进程拦截替换 |

---

## 2. CPU 供电降压与 Loadline 拓扑实测 (i9-13900H)

### AC/DC Loadline 六点连续测试拓扑
在 BIOS 332 解锁环境下，维持室内相同进气温度，针对 AC/DC Loadline 进行 110 至 135 档位的完整实测，以 Cinebench R23 多核循环跑分与 HWiNFO64 内核时钟有效交付率锁定物理极值点：

| AC/DC 设定 | Cinebench R23 多核 | 大核交付率 | 小核交付率 | 稳态功耗 / 温度 | 状态判定与底层现象 |
| :---: | :---: | :---: | :---: | :---: | :--- |
| **110** | 15,863 pts | 91.2% | 85.0% | 86.4W / 73.5°C | **严重时钟拉伸**：IA CEP 介入，有效频率大幅虚标 |
| **115** | 16,137 pts | 93.8% | 87.2% | 87.1W / 74.2°C | **时钟拉伸区**：小核交付率严重受损 |
| **120** | 17,253 pts | 98.1% | 92.5% | 88.5W / 75.8°C | 爬坡过渡区：交付率逐步释放 |
| **125** | 17,345 pts | 99.1% | 97.5% | 89.2W / 76.3°C | 接近满血，拉伸痕迹基本收敛 |
| **130** | **17,982 pts** | **99.7%** | **99.2%** | **90.2W / 77.0°C** | **【全局绝对峰值】** 时钟拉伸完全归零，能效比巅峰 |
| **135** | 17,718 pts | 99.7% | 99.2% | 90.0W / 78.4°C | **进入下降沿**：高电压撞 PL1 90W 功耗墙，倍频被迫降 20MHz |

### 物理机理与供电安全墙禁忌
1. **AC 严格等于 DC**：AC≠DC 会造成 CPU 内部 VID 计算逻辑脱节，引发高达 14°C 的额外积热与功耗读数虚标。
2. **`VccIn Aux Icc Max = MAX`**：**严禁维持出厂默认值 132**。默认 132 会导致电压瞬态响应失常，触发 VID 电压暴涨至 1.0V+ 并产生 90°C+ 瞬间积热；必须在 BIOS 中拉至 `MAX`。
3. **保护特性开关**：开启 `IA ICC Unlimited mode`，彻底禁用 `IA CEP` 与 `GT CEP`。
4. **倍频与功耗配额**：
   - P 核保持官方默认曲线：`54-54-51-51-49-49-49-49`（兼顾 1~2 核 5.4GHz 单核突发爆发力，全核稳态由功耗墙仲裁）。
   - E 核保持默认倍频：`41x4 / 39x4`。
   - 功耗配额设定：`PL1 = 90W`，`PL2 = 115W`。稳态均温压制在 75~77°C；若需彻底消除短时 90°C+ 的 PL2 尖峰，可将 PL2 进一步收敛至 90W~100W。

### 华硕主板底层的 MSR 与 EC MMIO 仲裁
笔记本 CPU 功耗受两个独立寄存器管辖：

$$\text{实际执行功耗} = \min(\text{MSR}, \text{MMIO})$$

- **MSR**：BIOS 或调节软件设定的静态寄存器。
- **MMIO**：主板 EC 硬件控制器直接管理。华硕在未受奥创底层接管状态下，会将 EC MMIO 死焊在 **90.0W**。即便在 BIOS 内将 MSR 改到 100W+，长时负载一旦触发 MMIO 钳制，实际稳态依然会被按死在 90W。

---

## 3. 开源排障双连击：G-Helper 华硕共享模具命名陷阱 (#5895 & #6039)

华硕天选4 酷睿版（FX507VV）与锐龙版共用模具，其主板 WMI 写入的系统模型标识串为：
`ASUS TUF Gaming F15 FX507VV_FA507XV`。
末尾的 `_FA507XV` 字符串在 G-Helper 源码中引发了两次经典的“型号误杀”，两起问题均由我向官方提交 Issue、定位源码根因并推动修复合并闭环。

### 3.1 连击一：Fn+F10 触控板开关失灵 (Issue #5895)
- **现象**：按下 `Fn+F10` 时屏幕弹出 OSD 提示“Touchpad On”，但触控板从未被禁用，OSD 永远固定为“On”，且日志每按一次均刷新 `WMI event 107 + Touchpad status:1`。
- **源码根因**：G-Helper `AppConfig.cs:682` 中的 `IsHardwareTouchpadToggle()` 使用了模糊包含检测，识别到了字符串里的 `FA507` 返回 `true`。程序误以为该机型是具备 EC 硬件独立开关的锐龙天选，在 `InputDispatcher.cs:753` 中跳过了向系统注入 `Win+Ctrl+F24` 快捷键的软件模拟路径；而该 Intel 模具的 EC 根本不支持硬件开关触控板，导致开关完全失效。
- **开源修复**：提交 [Issue #5895](https://github.com/seerge/g-helper/issues/5895)，建议在判定时改用 `GetModelShort()` 剥离后缀，使 `FX507VV` 正常走软件模拟注入通道（触控板 I2C HID `ASUF1204` 驱动接收后正常停用并写回注册表），作者采纳并合并发布于 v0.275。

### 3.2 连击二：功耗滑块锁定 90W 再次误杀 (Issue #6039)
- **现象**：酷睿满血 i9-13900H 理论具备 115W~150W 动态调节配额，但在 G-Helper“风扇+功耗”界面的滑块上限被强行压死在 **90W**。
- **源码根因**：虽然此前在触控板模块修复了前缀判定，但在 CPU 功耗模块 `app/AppConfig.cs:652` 的 `IsCPULight()` 中依然残留了旧代码：
  ```csharp
  public static bool IsCPULight()
  {
      return ... || ContainsModel("FA507X") || ...;
  }
  ```
  该行再次精准命中了 `_FA507XV`，使 `app/AsusACPI.cs:363` 触发截断：
  ```csharp
  if (AppConfig.IsCPULight())
  {
      MaxTotal = 90; // 再次将 Intel 误当成锐龙轻薄款截断为 90W
  }
  ```
- **开源修复**：提交 [Issue #6039](https://github.com/seerge/g-helper/issues/6039)，给出源码行号与匹配补丁，作者当天确认并关闭 Issue，完整合入官方 Release（v0.284+），彻底恢复 150W 原生滑块调节。

---

## 4. GPU 极限超频与显存拓展 (RTX 4060 Laptop 140W)

使用 G-Helper 进行超频管理，开机自启应用，彻底摒弃 MSI Afterburner（避免底层曲线重置与驱动冲突）：

```ini
[GPU Overclocking]
Core Clock Offset = +250 MHz
Memory Clock Offset = +1200 MHz
Power Target = 140W (Dynamic Boost Active)
```

- **核心状态**：实际高负载 Boost 频率直接顶满 **2,730 MHz** 硬件电压墙，无降频掉帧，0 WHEA 报错。
- **显存状态**：海力士 GDDR6 颗粒显存频率偏移拉升至 **+1200 MHz**，等效显存频率突破 18.4 Gbps，大幅缓解 128-bit 位宽带来的高分分辨率带宽瓶颈。
- **动态功耗保留**：**严禁在设备管理器中禁用 NVPCF 虚拟设备**。禁用 NVPCF 会破坏 NVIDIA Dynamic Boost 协议，导致独立显卡功耗被锁死在 115W，白白损失 25W 核心动力。

---

## 5. DDR5 内存时序深度压榨 (免焊改模具极限)

天选4 采用双通道叠放非对称散热布局，且主板 PMIC 固件硬锁内存 VDD 电压于 **1.095V**（BIOS 强制填 1.2V 均不生效）。在无法加压的前提下，对原厂美光 DDR5-4800（16Gb D8BNK 颗粒）进行极限压参：

| 内存参数 | 默认出厂值 | 调优终态参数 | 优化机理与物理边界 |
| :--- | :---: | :---: | :--- |
| **等效频率** | 4800 MHz | **5400 MHz** | 模具 1.095V 物理耐受上限（冲击 5600 MHz 必蓝屏） |
| **主时序 (CL-TRCD-TRP-TRAS)** | 40-39-39-77 | **36-39-39-76** | 压缩 CL 至 36，降低基础寻址周期 |
| **Command Rate (CR)** | 2T | **CR2** | 维持双通道稳定拓扑 |
| **刷新间隔 (tREFI)** | 4680 | **9360** | 原生参数精准翻倍，减少电容刷新停顿时间（拉到 12000 必冻死） |
| **行周期 (tRAS)** | 77 | **76** | 实测压到 68 反而引发重试导致读取带宽下降 2.5%，76 为真实甜点 |

**压榨成效**：AIDA64 内存读取达 **82.3 GB/s**，物理内存延迟压至 **78.6 ns**，理论带宽利用率达到 95%，重载高压测试温度稳定在 59~61°C。

---

## 6. Windows 11 系统底噪治理与现代渲染

### 字体渲染：原生高质方案
彻底抛弃易崩溃、需常驻后台的旧式挂钩工具（如 NoMeiryoUI / MacType）：
- **中文字体全局替换**：通过系统注册表 FontSubstitutes，将 `Microsoft YaHei` 与 `Microsoft YaHei UI` 映射为 **`HarmonyOS Sans SC`**（系统内置 19.7MB 可变字体，全字重矢量清晰渲染）。
- **保留拉丁原生排版**：**严禁对 `Segoe UI` 或 `Segoe UI Variable` 进行任何注册表篡改**，确保 Windows 11 现代 Fluent UI 图标与英文字符无缺字、无不对齐。

### 进程与监控软件治理纪律
1. **卸载 Intel XTU**：其后台常驻服务 `XtuService` 会恶意将主板 BIOS 的 PL1/PL2 覆写截断为 35W/70W，引发性能暴跌。
2. **严禁在性能测试中引入游戏加加（GamePP）**：其 Electron 架构的 `gpu-process` 后台空转会霸占 1 个物理核心，直接导致 Cinebench R23 跑分虚假缩水 1,000+ 分。跑分与传感器监测严格以**轻量原生 C++ 架构的 HWiNFO64** 为唯一准绳。
3. **消除系统冗余驻留**：
   - 组策略停用开始菜单必应 Web 搜索，消除 `SearchHost.exe` 关联的多个后台 WebView2 冗余进程。
   - 日常 Web 工具优先采用 Edge 侧边栏常驻（复用 Edge 主进程仅增加 ~60MB 内存），拒绝单独开启多进程独立 Chromium PWA（避免额外产生 400MB+ 底噪）。

---

## 7. 全场景稳定性与实测性能基准

```ini
[Benchmark Scoreboard]
Cinebench R23 Multi-Core  = 17,982 pts (稳态 77.0°C / 90.2W, 时钟拉伸 0%)
Cinebench R23 Single-Core = 2,051 pts (5.4GHz 稳态 72W / 74°C)
3DMark Time Spy Overall   = 11,639 (Graphics 11,287 / CPU 14,145)
3DMark Steel Nomad        = 2,472 pts
AIDA64 Memory Read        = 82.3 GB/s (Latency 78.6 ns)

[Game Real-World Performance]
CS2 (2560x1600, 全开)       = 平均 369.1 fps | 1% Low 167.2 fps (NVIDIA Reflex 开启)
Delta Force 高压战斗段       = 平均 128 fps | 卡顿帧率 0.26% | GPU 占用 90%~99%
```

全套调优方案在保证整机硬件无损、不破保、不进行破坏性物理改造的前提下，将 i9-13900H、RTX 4060 与 DDR5 内存的物理吞吐压榨至甜点极值，同时维持了极低的发热量与纯净的系统底噪。
