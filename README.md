# gt5pro-hmbird

真我 GT5 Pro（RMX3888 / SM8650 / ColorOS 16 / 6.1.141 **OKI 基线**）上 **hmbird（风驰）接口层与调度核心**的移植记录。

> 本仓库内容由 **DeepSeek** 协助机主整理撰写。

## ⚠️ 先说结论（务必看）

- ✅ **接口层已打通**：`/proc/hmbird_sched` 21 条（与出厂内核半套一致）、厂商模块可装载并初始化。
- ✅ **调度核心已搬入并能编进内核**（19 文件 / 7,990 行，`CONFIG_HMBIRD_SCHED_CORE=y` 下 rc=0），且**全程 ABI 不变**（导出集 15,489 逐名一致，493 个厂商模块不受影响）。
- ✗ **但还不能用**：写入 `echo 1 > /proc/hmbird_sched/scx_enable` 会**整机硬挂死**（详见 [docs/实测记录-20260910.md](docs/实测记录-20260910.md)）。
- ⛔ **不要**尝试写 `scx_enable=1`；任何对外镜像请保持 `CONFIG_HMBIRD_SCHED_CORE=n`（默认即 n）。

## 现状

| 项 | 状态 |
|---|---|
| 内核侧接口 `/proc/hmbird_sched` | ✅ 21 条（装厂商模块后 27 条）|
| 厂商模块按依赖序装载 | ✅ `game_opt` → `sched_assist` → `sched_ext` rc=0 |
| 模块调度侧初始化 | ✅ `[scx_gov][scx_cpufreq_init] num_cluster=4` |
| 核心家族编入内核 | ✅ `CORE=y` 全量 rc=0；`/proc/kallsyms` 有 180 个 hmbird 符号 |
| 接进 `core.c` 调度路径 | ✅（`sched_fork` / `__task_prio` / `scheduler_tick` 等）|
| ABI | ✅ 导出集 15,489 逐名未变；493 模块 missing=0 crc=0 |
| **真正启用（scx_enable=1）** | ✗ **硬挂死** —— 缺 fork 的运行期语义 |

## 适用条件（不满足则不适用）

- 内核基线必须是 **OPPO 系（OKI）**，例如 `6.1.141-android14-11-o-…`，**不是**纯 GKI 或别的厂商基线；
- 设备树里 hmbird 类型为 `HMBIRD_OGKI`（SM8650 即此档）；
- 厂商模块 `oplus_bsp_sched_ext.ko` 及依赖在位，且**按依赖顺序加载**。

## 方法（简述）

1. **以出厂内核为基准**：符号、CRC、结构体布局逐条比对（结构体用 BTF 逐字段对偏移）；
2. **以设备上的 493 个厂商模块为硬判据**：全量审计"能否装载"，缺失或 CRC 不符一律不放过；
3. **每步都要过三道闸门 + 真机验收**，过不了就回退。

详见 [docs/方法.md](docs/方法.md) 与 [docs/实测记录-20260910.md](docs/实测记录-20260910.md)。

## 借鉴与致谢

见 [docs/来源与致谢.md](docs/来源与致谢.md)（含各来源仓库链接与许可证状态；**无明确许可证的来源只作引用与致谢，不包含其代码**）。

## 本仓库不包含什么

- 不包含来自**无明确许可证**来源的代码；
- 不包含任何**厂商二进制**（`.ko` / `Image.stock` 等）。

## 许可

GPL-2.0
