# gt5pro-hmbird

真我 GT5 Pro（RMX3888 / SM8650 / ColorOS 16 / 6.1.141 **OKI 基线**）上 **hmbird（风驰）接口层**的移植记录与工具。

> 本仓库内容由 **DeepSeek** 协助机主整理撰写。

## 现状（2026-10-09，opt60 真机验证）

| 项 | 状态 |
|---|---|
| 内核侧接口 `/proc/hmbird_sched` | ✅ 出现，**21 条**（与出厂内核半套一致）|
| 厂商模块按依赖序装载 | ✅ `oplus_bsp_game_opt` → `oplus_bsp_sched_assist` → `oplus_bsp_sched_ext` rc=0 |
| 装载后节点数 | ✅ **27 条**（+6 模块侧）|
| 模块调度侧初始化 | ✅ `[scx_gov][scx_cpufreq_init] num_cluster=4` |
| `echo 1 > /proc/hmbird_sched/scx_enable` | ✅ **不挂死**（opt43 的整机硬挂死未复现）|
| **调度器真正生效** | ⚠️ **尚未** —— 缺 hmbird sched_class 核心（`gdsqs`/`pcp`/`partial` 那套）|

## 适用条件（不满足则不适用）

- 内核基线必须是 **OPPO 系（OKI）**，例如 `6.1.141-android14-11-o-…`，**不是**纯 GKI 或别的厂商基线；
- 设备树里 hmbird 类型为 **`HMBIRD_OGKI`**（SM8650 即此档；`HMBIRD_GKI` / `HMBIRD_EXT` 属别的平台，不能混用）；
- 厂商模块 `oplus_bsp_sched_ext.ko` 及其依赖在位，且**按依赖顺序加载**。

## 方法（简述）

1. **以出厂内核为基准**：符号、CRC、结构体布局逐条比对（结构体用 BTF 逐字段对偏移）；
2. **以设备上的 493 个厂商模块为硬判据**：全量审计"能否装载"，缺失或 CRC 不符一律不放过；
3. **每步都要过三道闸门 + 真机验收**，过不了就回退。

详见 [`docs/方法.md`](docs/方法.md)。

---

## 借鉴、引用与致谢

本项目的结论**建立在下列公开工作之上**。它们**没有明确的开放许可证**，因此本仓库**只做引用、借鉴与致谢，不包含其代码**；有明确许可证的，按其许可证执行。

### 一、基准（不随仓库分发）
- **出厂内核本体与其 BTF**：本项目所有"逐字段 / 逐指令 / 逐 CRC"对齐的**唯一基准**。

### 二、官方公开源码（内核本体为 GPL-2.0）
- **OPPO**：`oppo-source`（含 `android_kernel_common_oppo_sm8650`）
- **realme**：`realme-kernel-opensource`（GT5 Pro / GT5 / GT6 各代 drop）
- **OnePlus**：`OnePlusOSS`（`android_kernel_common_oneplus_sm8650`）

> 已核实：这些公开 drop 中**不含** hmbird 实现。

### 三、社区来源（**仅引用与致谢，未包含代码**）

| 来源 | 借鉴了什么 | 许可证状态 |
|---|---|---|
| **ferstar** — `realme_GT5pro-AndroidV-common-source` / `-vendor-source` / `kernel_manifest` | GT5 Pro 源码镜像、**`scx` 分支**（OGKI 形态的内核侧实现，作为对齐参照）| 未标注 / 无 |
| **reigadegr** — `sun_action` | **6.1 sched_ext 补丁**（`6.1sched_ext.diff`，作为缺失文件与调用点的参照）| 无 |
| **reigadegr** — `hmbird_controller` | **风驰节点（knob）清单**与开关名 `scx_enable` 的用法 | 无 |
| **WildKernels** — `kernel_patches` | `oneplus/hmbird/*.patch`（A15 = 6.1 的 hmbird 补丁思路）| 无 |
| **wanwei1028** — `oneplus_hmbird_fix` | **DT `version_type` 兼容层**的思路与降级写法 | 无 |
| **murongruyan** — `cezai-hmbird-ko` | **风驰 DTBO 模块**：SoC → HMBIRD 类型对照表（SM8650 = `HMBIRD_OGKI`）| **GPL-3.0** |
| **mcLYX** — `RMX3888_KN_kernel_manifest` | GT5 Pro 的**构建清单**（`build_v.sh`：`APPLY_SCX=y` 切 `scx` 分支）| 无 |
| **Numbersf** — `Action-Build` | 一加系内核 Action 构建工程（`patches/hmbird_patch.patch` 的 DT 改写思路）| 未标注 |
| **TheVoyager0777** — `Platform_Phantom` | 仅作对照（其 14 个 `.ko` 经 ABI 校验判定**不可用**，未采用其任何二进制）| **GPL-2.0** |

**郑重感谢以上作者与组织的公开工作。** 若其中任一作者认为本项目的引用或表述不妥，请联系，我会立即调整或移除。

### 四、我们自己的部分
接口 hub 的设计、逐条对齐与审计方法、CRC 定点核对方法、三道闸门工具，以及全部过程记录。

---

## 本仓库不包含什么（重要）

- **不包含**来自**无明确许可证**来源的代码（仅引用与致谢，见上）；
- **不包含**任何厂商二进制（`.ko` / `Image.stock` 等）；
- 详细来源清单见 [`docs/来源与致谢.md`](docs/来源与致谢.md)。

## 许可

GPL-2.0
