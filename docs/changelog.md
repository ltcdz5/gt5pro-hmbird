# 更新日志（Changes）

## 2026-10-11 · 进度口径更新：把「两条路」分清

### 一、结论先行

| | 状态 |
|---|---|
| **风驰游戏切换**（`opt60` + `fengchi-boot`）| ✅ **可用**，与上一节相同，无变化 |
| **内核态 hmbird 调度类**（`/proc/hmbird_sched/scx_enable=1`）| ⚠️ **仍不可用** |

> 这两条是**不同的路**。README 里说的「显示 `scx` = 风驰已生效」指前者。

### 二、内核态 hmbird 调度类：本轮二分结果

**现象**：启用后 `running` 从 1 涨到 20~40，任务被压在 **CPU0/1/2**（大核与超大核空闲），
约 60 秒后 hmbird 自检到任务饿死（`failed to run for ~58s`）并**自动禁用自己**，系统随后恢复。
全程 **无 panic、无 WALT-BUG、无 PSI 不一致**。

**已排除（每条都有现场证据）**：

| 嫌疑 | 排除方式 |
|---|---|
| partial 簇 / rescue / partial util 采样 | 把 `partial_enable` 恒置 0 后**仍然**出现同样现象 |
| `scx_gov` 调速器 | 全程实测调速器是 `uag`、从未变成 `scx_gov`，现象照样出现 |
| 野指针写 | 相应补丁被判定为**结构性 no-op**（4 个调用点全在 hmbird 自己的类回调内）|
| 两份 `slim_walt_ctrl` 状态不一致 | 模块那份是模块局部符号，**不共享、不互读** |
| `hmbird_ops` ABI 错位 | 设备模块里**根本没有**这些符号（全量 nm 扫描）|
| 内核侧 irq_work 全 rq 锁 | 已停用该排队后**仍然**出现 |
| 「WALT 与 hmbird 两套全速跑」 | 那 15 处版本让路**在设备二进制里根本不存在**（内核/模块全 0 命中）|

**根因仍未定位。** 目前的判断：问题更可能出在**这套调度类的放置策略**上。

### 三、本轮新证据（第一次拿到完整现场）

1. `running` **不是单调泄漏** —— 80 秒里在 4~36 之间来回波动、多次回落到 4。
2. 任务**全压在 CPU0/1/2**，CPU3~7 完全空闲。
3. `sysrq-l` 抓到的全部 CPU 栈显示 **没有任何 CPU 卡在锁上**（大核全 idle，小核在正常干活）。
4. hmbird 启用后，`/sys/kernel/debug/sched/debug` 的 `cfs_rq` 段落从 **39 个变成 0 个**。

### 四、下一步

评估改走上游官方 **sched_ext（BPF struct_ops 调度器）** 路线 ——
内核本来就支持（`CONFIG_SCHED_CLASS_EXT=y`，`/sys/kernel/sched_ext/` 存在且从未启用），
且**不依赖 WALT 帧率感知 util**。首个实验：用最小 BPF 调度器验证官方路径能否在本机跑起来。

### 五、设备状态

- 已回退到干净镜像 **opt58**（`boot-v1.1-opt60-repacked.img`，md5 `5fd7909866e0de04b8e46cd9b388cc2e`）
- `scx_enable=0`、调速器全 `uag`、`/sys/kernel/sched_ext/enabled=0`


## 2026-10-09 · 风驰（scx）实测可用 + 统一附加模块

### 一、变化

1. **风驰实测生效** ✓
   - 游戏运行时，`/sys/devices/system/cpu/cpufreq/policy*/scaling_governor` 自动变为 **`scx`**
   - 退出游戏后回到**官方默认 `uag`**（注意：`walt` 是 Scene 的策略，不是官方默认）
   - 实测：第五人格、绝区零均正确切换；`Oops/BUG/panic = 0`

2. **新增 `fengchi-boot` 附加模块**（3 文件 / 约 2.3 KB）
   - 开机把厂商 `scx/hmbird` 栈按依赖序加载好（`scx` 调速器由此注册）
   - 游戏触发切换：有游戏 ⇒ `scx`；无游戏 ⇒ `uag`
   - 顺手并入 **MGLRU 运行时恢复**（原 `lru_gen_on`）与 **horae 拉起**（原 `horae_once`）
   ⇒ 这两个模块**可以删掉**了

3. **必须卸载 `IMS_VAROS`（"官方调度屏蔽模块"）** ✗
   - 它会清空 `persist.sys.orms.name` / `oiface.feature` / `hardcoder.name` 并禁 `OrmsUrcc`/`OiFace`/**`GameOpt`**/**`COSA`** ⇒ 与风驰直接冲突
   - ⚠️ 要用 `ksud module uninstall` 正常卸载；**改名目录会导致 KSU 卸载失败**（按目录名认模块）

4. **关掉 Scene 的调度/调速器设置** ✗（`walt` 会盖掉官方 `uag`）

5. **不要用任何工具"固定" scx** ✗ —— **会整机硬挂死**（内核开关那条路是另一个血统，本项目实测复现过）

### 二、实际是否可用

**可用** ✓（真机实测：游戏内 `policy0/policy7 = scx`，重负载稳定，无 Oops/BUG/panic）

边界：内核导出集 **15,489** 逐名未变；493 个厂商模块 missing=0/crc=0 ✓

### 三、版本
- 内核：opt60（6.1.141，`boot md5 5fd7909866e0de04b8e46cd9b388cc2e`）
- 模块：`fengchi-boot v1.3`（idle=`uag`；含 MGLRU）
