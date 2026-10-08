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

## 本仓库不包含什么（重要）

- **不包含**来自**无明确许可证**来源的代码（仅作引用与致谢）：`ferstar/*`、`reigadegr/*`、`WildKernels/*`、`wanwei1028/*`、`mcLYX/*`；
- **不包含**任何厂商二进制（`.ko` / `Image.stock` 等）；
- 详见 [`docs/来源与致谢.md`](docs/来源与致谢.md)。

## 许可

GPL-2.0
