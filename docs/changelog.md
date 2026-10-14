# 更新日志（Changes）

## 2026-10-14 · ★P0 判决：核心簇被 hmbird 排除出调度视野★（新增，请优先读这条）

### 一、结论先行

**「内核态 hmbird 调度类可用」这个口径要加一条硬限定**：它**能开、能关、能与厂商 `scx` 共存**，
但**在线时 cpu5/6/7 被排除出调度视野**，调度质量不成立。

**判决证据（同样 3 个线程，用 `taskset e0`（bit5/6/7）硬绑到 cpu5/6/7）**：

| 条件 | cpu5 / cpu6 / cpu7 | 频率 f5 / f6 / f7 |
|---|---|---|
| hmbird **OFF** + gov=uag | **100 / 100 / 100** | 2572800 / 2572800 / 2169600 |
| hmbird **ON** + gov=scx | **0 / 1 / 0** | 499200 / 499200 / 672000（最低档） |

⇒ CPU 物理、调频、thermal 全部正常（OFF 时三核满频 100%），**问题纯粹是 hmbird 在线时把 cpu5/6/7 从「任务可运行的核」里剔除**，
**连显式 `sched_setaffinity` 都绕不过去**。

### 二、三个用户态开关全部无效（不要再在它们上面花真机轮次）

| 开关 | 实测 |
|---|---|
| `partial_ctrl=1` | hmbird **OFF** 时写 1 保持（`1/1/1`）；hmbird **ON** 时（无论 gov=uag 还是 scx、无论有无 12 线程负载）**立刻被清**（`0/0/0`） |
| `isolate_ctrl=1` | 同上，写不住 |
| `iso_free_rescue=1` | **能写住**（读回 1 保持），但 cpu5/6/7 仍 `0/0/0` ⇒ 无效 |

`partial_ctrl` 写不住的机制：它绑的是裸 int `partial_enable`（`kernel/sched/hmbird_sched_proc_main.c:400-403`，
写处理器 `hmbird_common_write :201-211` 无保护），而 `partial_dynamic_ctrl()`（`kernel/sched/hmbird/hmbird.c:1006-1048`）
在 `l_under && b_under` 时执行 `set_partial_rescue(false,false,false)`（`:1032-1033`）→ `WRITE_ONCE(partial_enable,false)`（`:620-625`）
⇒ **用户写进去的值被状态机在下一拍收回**。hmbird OFF 时该函数不被调用，故写得住。

### 三、基线复现（12 个等优先级 spin，gov=scx + `scx_enable=1`）

12 个样本**全部**：

```
cpu0=100 cpu1=100 cpu2=12~60 cpu3=9~59 cpu4=11~61 cpu5=0 cpu6=0 cpu7=0
```

### 四、下一步（最高优先级）

源码定位「hmbird 在线时 cpu5/6/7 连 `taskset` 绑核都跑不动」的那一处机制。候选：
`hmbird.c:1400-1405`（EXCLUSIVE 早退）、`:1420-1434`（iso_free_rescue 出口）、
`task_only_blongs_to_cluster` 经 `:3273→:1276` 与消费侧共用 `gen_cluster_ctx_separate` 的放置侧路径。
**要能解释「为什么显式 `sched_setaffinity` 也不能让任务在这些核上运行」。**

> 完整现场：`gt5pro-kernel-lab/2026-10-14/`（阶段 I，含 p3v2 判决脚本与三个探针）；
> 台账：`gt5pro-kernel-kit/CHANGELOG.md` 第「三十四」章。

### 五、D14 标注

实验期间前台为 `com.deepseek.chat`（机主在用手机）。各臂对照在同污染条件下取得，且 A0 复现与绑核对照均为同 run 内前后对照
⇒ **方向可信**；绝对数值（cpu2–4 零散占比）可能被用户负载抬高，不作结论依据。

---

## 2026-10-11 · 进度口径更新：把「两条路」分清

### 一、结论先行

| | 状态 |
|---|---|
| **风驰游戏切换**（`opt60` + `fengchi-boot`）| ✅ **可用**，与上一节相同，无变化 |
| **内核态 hmbird 调度类**（`/proc/hmbird_sched/scx_enable=1`）| ✅ **可用** —— 与厂商 `scx` 调速器稳定共存并完整还原（opt161b 200 s / opt162 133.2 s）。完整证据链见 [`hmbird-内核态调度类-进展.md`](hmbird-内核态调度类-进展.md) |

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

**根因已定位并修复。** 三个**互相独立**的缺陷，逐个修掉后 hmbird 与 `scx` 可长期共存：① 超时兜底窗口只覆盖 `T2` 簇的一半 ⇒ 迟到任务无上界（饿死）；② 切类循环缺少与原厂参照补丁一致的过滤 ⇒ 产生 `balance_callback` 契约违反；③ watchdog work 每 60 s 周期性强切 governor。逐项现象 / 根因 / 修法 / 判据见 [`hmbird-内核态调度类-进展.md`](hmbird-内核态调度类-进展.md)。

### 三、本轮新证据（第一次拿到完整现场）

1. `running` **不是单调泄漏** —— 80 秒里在 4~36 之间来回波动、多次回落到 4。
2. 任务**全压在 CPU0/1/2**，CPU3~7 完全空闲。
3. `sysrq-l` 抓到的全部 CPU 栈显示 **没有任何 CPU 卡在锁上**（大核全 idle，小核在正常干活）。
4. hmbird 启用后，`/sys/kernel/debug/sched/debug` 的 `cfs_rq` 段落从 **39 个变成 0 个**。

### 四、下一步

> ⛔ **已否决，勿再作为下一步。** 依据：2026-10-03 实测 `bpftool struct_ops register` **一加载就硬挂死整机**（无 panic / pstore，看门狗复位）；去掉 `scx_bpf_switch_all` 的「部分模式」（opt43）**同样硬挂**。该路线的失败面至今没有解法。以下保留为历史记录。
>
> ~~评估改走上游官方 **sched_ext（BPF struct_ops 调度器）** 路线 —— 内核本来就支持（`CONFIG_SCHED_CLASS_EXT=y`，`/sys/kernel/sched_ext/` 存在且从未启用），且**不依赖 WALT 帧率感知 util**。首个实验：用最小 BPF 调度器验证官方路径能否在本机跑起来。~~

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

---

## 2026-10-13

### 一、内核态 hmbird 调度类打通（从「不可用」到「与 scx 共存」）

| 阶段 | 修的是什么 | 实测判据 |
|---|---|---|
| `opt158` | 切类循环缺过滤 ⇒ `balance_callback` 契约违反，`rq_pin_lock` 触发 `brk #0x800`（参照补丁 `refwork/ref/kernel__sched__ext.c:3141-3143` 本来就有这条过滤，我们曾删除） | t183：`balance_cb=0 ext_ctrl=0 task_rq_lock=0 __schedule=0 stall=0` |
| `opt159` | 超时兜底窗口只覆盖 `T2` 簇的一半（BIG `[5,8)` / LITTLE `[8,10)`）⇒ 迟到任务无上界，`android.bg` 落 `gdsqs[9]` 饿死（定量缺口 62,207 人次） | t184：8 线程 200 s `stall=0 balance=0 disabled=0` |
| `opt160` | watchdog work 里每 60 s 周期性强切 governor（一局 6m41s 重建 6~7 次） | 反汇编核验：`hmbird_watchdog_workfn` 中 `cpufreq` 引用数 = 0 |
| `opt161b` | 与厂商 `scx` 调速器共存（顺序闸门 + 还原路径加固） | t190：200 s `stall=0 balance=0 disabled=0 warn=0 pc=0 t42=0 veto=0 maxrunq=22 maxload=21.84 maxtz=51200` |
| `opt162` | 新增 `hmbird_partial_util` 旋钮（默认 `N`）；端到端复测 | 共存窗口 **133.2 s**，M1~M4 全 PASS，退出终态 `gov=uag×4 scx=0 slim=0 veto=N` |

**同时被证伪的历史说法**（见 `hmbird-内核态调度类-进展.md`）：

- 「`scx_enable` 回读 0 ⇒ 调度未真正生效」——回读的是**用户意图**，不是状态；
- 「`/proc/hmbird_sched/hmbird_stats` 全是 0 ⇒ 没接管」——那一段当时是**硬编码假数据**（opt156 才改成真 `stats_print()`）；
- 「疑点：两份 `slim_walt_ctrl` / 野指针 / ABI 错位」——均已排除（清单见进展文档）。

### 二、守护模块 `fengchi-boot` 修到 v2.6

- `versionCode 17`，zip **19,976 B**，md5 `3737bd566c0d373bf61b6403572d8314`；独立复核判定 **PASS**；
- 修的两件事：`D4 版本闸门`、`F1 内核自禁盲区`；
- **未验证**：是否发布 / 是否已在设备上跑过（本文不写发布状态）。

### 三、★架构事实：`scx` governor 来自厂商模块，而唯一装载它的是守护模块★

- `scx` 这个 **cpufreq governor 不是内核自带的**，它由厂商模块 `oplus_bsp_sched_ext.ko` 注册；
- 设备上**唯一** `insmod` 它的是守护模块 `fengchi-boot` 的 `service.sh`（依赖序 `oplus_bsp_game_opt` → `oplus_bsp_sched_assist` → `oplus_bsp_sched_ext`）；
- ⇒ **停用守护模块 = 同时移除 `scx` governor**（`policy*/scaling_available_governors` 里会没有 `scx`，`echo scx` 会失败）。这是 t194 轮首次失败的真实原因，不是 hmbird 的问题。

> 出处：`_audit/成功-t194游戏态启用与退出完整还原.md:14-18`（`service.sh:33-38`，16:23 `touch disable` 后 `scx` 消失）。

### 四、本轮新增文档

- [`hmbird-内核态调度类-进展.md`](hmbird-内核态调度类-进展.md) —— 主线证据链 + 已排除嫌疑 + 未复现清单
- [`opt162-真机验证.md`](opt162-真机验证.md) —— opt162 真机验证记录（md5 / boot_id / ENTER-EXIT / M1~M4）
- [`启用与还原-操作手册.md`](启用与还原-操作手册.md) —— 五步顺序闸门 + 四步还原 + 禁止项
- [`判据与陷阱.md`](判据与陷阱.md) —— 假 PASS / 假 FAIL / 判据速查
