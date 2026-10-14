# 内核态 hmbird 调度类：进展与证据链

> 最后更新：2026-10-14 ｜ 内核树坐标基于 `/home/builder/kwork/wt-core`，HEAD `7cc87ec0d8dc`（分支 `opt162`）。
> 结论：**可开、可关、可与厂商 `scx` 调速器同时在线；但存在一个未修的 P0 缺陷 —— 在线时核心簇被排除出调度视野。**
> 本文只记录**能在设备上复现**的结论；每条都给出实测数字与出处（档案行号或源码坐标）。找不到出处的写「未验证」。

> ★**2026-10-14 重大限定（先读这条再引用本文的「已可用」）**★
> 「已可用」**只覆盖**「不崩、不饿死、能与 scx 共存」这三件事。**它不覆盖调度质量**：
> 实测 **hmbird 在线时 cpu5/6/7 被排除出调度视野** —— 12 个等优先级 spin 只落 cpu0/cpu1（cpu5/6/7 恒 0），
> 且**用 `taskset e0` 显式把线程绑到 cpu5/6/7 也跑不动**（0/1/0，频率停在最低档 499200/499200/672000）；
> 同样三个线程在 hmbird **OFF** 时是 100/100/100、满频 2572800/2572800/2169600。
> ⇒ CPU 物理正常，是 hmbird 在线时剔除了这些核。
> **三个用户态开关（`partial_ctrl` / `isolate_ctrl` / `iso_free_rescue`）实测全部无效**（前两者在 hmbird 在线时根本写不住，
> 写 1 立刻读回 0；后者能写住但拉不回核）。
> 详见 [`../` 归档] `gt5pro-kernel-lab/2026-10-14/`（阶段 I）与 `gt5pro-kernel-kit/CHANGELOG.md` 第「三十四」章。

## 一、一句话结论

内核态 hmbird 调度类曾经**一启用就坏**，根因不是玄学，而是**三类可定位、可修复的缺陷**：

1. **切类循环缺少 reject 过滤** ⇒ 产生 `balance_callback`，违反 core 契约 ⇒ 断言 `brk #0x800` ⇒ 整机冻结复位；
2. **超时兜底 DSQ 的簇窗口各只看一半** ⇒ 部分任务（`android.bg`）等待无上界 ⇒ 饿死 ⇒ 看门狗自禁；
3. **watchdog work 每 60 s 周期性强切 governor** ⇒ 与 `scx` 共存时反复重建调速器，是「硬挂」疑云的主要来源。

三类缺陷分别在 **opt158 / opt159 / opt160** 修掉；**opt161b** 首次在真机证明「hmbird + scx 同时在线 200 s」，**opt162** 再次复现 133.2 s 并补齐 `partial_util` 单变量臂。

## 二、主线时间线

| 版本 | 修掉的缺陷 | 实测判据 | 归档 |
|---|---|---|---|
| **opt158** | 平衡回调契约违反（`balance_callback` 断言） | t183：`balance_cb=0 ext_ctrl=0 task_rq_lock=0 __schedule=0 stall=0`，`boot_id` 未变 | `_audit/成功-opt158修复balance_callback断言.md` |
| **opt159** | T2 簇可见性 ⇒ 饿死 | t184：8 线程 200 s，`stall=0 balance=0 disabled=0`，`runq=10~16` | `_audit/成功-opt159修复饿死缺陷.md` |
| **opt160** | watchdog 周期性强切 governor | 二进制核验：`hmbird_watchdog_workfn` 反汇编里 `cpufreq` 调用点 = **0**；后续 t190 实测 `t42=0` | `_audit/成功-opt160停用周期强切governor.md` |
| **opt161b** | —（共存验证） | t190：`stall=0 balance=0 disabled=0 warn=0 pc=0 t42=0 veto=0 maxrunq=22 ... gov=scx scx=1` | `_audit/成功-opt161b上机验证hmbird与scx稳定共存.md` |
| **opt162** | 新增 `hmbird_partial_util` 旋钮（默认 N） | 共存窗口 uptime `124.42 → 257.64` = **133.2 s**；M1~M4 全 PASS | `_audit/成功-opt162上机验证.md` |

> 行号口径：标「**当前树**」的坐标已在 HEAD `7cc87ec0d8dc` 逐行核对；标「**档案行号**」的是当时那棵树的坐标，**未**在 HEAD 上复核，引用前请重新 `grep`。

## 三、缺陷 1：T2 簇可见性 ⇒ 任务饿死（opt159 修）

### 现象

- 启用 hmbird 后 `running` 从 1 涨到 **20~40**，任务被压在 **CPU0/1/2**，大核与超大核空闲；
- 约 **60 s** 后 hmbird 自检到任务饿死（`failed to run for ~58s`）并**自动禁用自己**，系统随后恢复；
- 全程无 panic、无 WALT-BUG、无 PSI 不一致。

### 现场日志（决定性的一次：t184，opt159，8 线程 200 s）

```
[  444.976240] t184 BEGIN up=444 kernel=6.1.141-android14-11-o-ltcdz5-v1.1-opt159 kmsg0=15341 boot_id=57197068-73e6-41a7-b73e-493aa337632b
[  451.285436] t184 ENABLED scx=1
[  454.577612] t184 LOAD8 started load=4.77
[  582.093864] t184 t=125s scx=1 load=14.70 runq=11   ← 跨过 120 s 看门狗阈值
[  658.298810] t184 t=200s scx=1 load=16.55 runq=15
[  659.255741] t184 RESULT stall=0 balance=0 disabled=0
[  664.091742] t184 ALL DONE boot_id=57197068-73e6-41a7-b73e-493aa337632b load=16.26
```

历史对照（同一缺陷未修时）：

| 实验 | 内核 | 线程 | 时长 | 结果 |
|---|---|---|---|---|
| t169 | opt152 | 8 | 600 s | `stall` 出现（`android.bg` 饿死 167 s）⇒ 被自动禁用 |
| t172 | opt152 | 8 | 600 s | `stall` 出现（饿死 120 s）⇒ 被自动禁用 |
| **t184** | **opt159** | **8** | **200 s** | **`stall=0 balance=0 disabled=0`** |

### 根因（档案行号，opt159 时代）

`gen_cluster_ctx_separate`（档案行号 `hmbird.c:1197-1229`）给 BIG 簇的 DSQ 窗口是 `[5,8)`、给 LITTLE 的是 `[8,10)`：

- ⇒ `gdsqs[8]/[9]` 对 BIG 完全不可见，`gdsqs[5..7]` 对 LITTLE 完全不可见；
- ⇒ **各簇只能扫一半**，另一簇的桶只能靠**超时兜底**迟到服务 ⇒ **等待时间无上界**；
- ⇒ `android.bg`（cgroup `bg` → `gdsqs[9]`，档案行号 `cgrp_name_to_idx:1683-1687`）恰好落在只有 LITTLE 可见的那一半。

定量对账（wb-8/t90，出处 `_audit/成功-opt159修复饿死缺陷.md:56-61`）：

```
gdsq 入队总和 199,293 / 消费总和 137,086 / 缺口 62,207
逐桶 5:47 / 6:57 / 7:53258 / 8:8821 / 9:24  = 62,207   ← 精确吻合
gdsq[7] 入队 53362 / 正常消费 104        ← 99.8% 是迟到消费
gdsq[8] 入队 55738 / 正常消费 46917      ← 差 451 倍
```

### 修法

只放宽**超时兜底路径**的窗口（`consume_timeout_dsq` 内，档案行号）：

```c
ctx.lower = NON_PERIOD_START;   // 5
ctx.upper = NON_PERIOD_END;     // 10
```

### 判据

t184：`stall=0` / `disabled=0` / 8 线程满载 200 s / `runq=10~16` / `boot_id` 未变。

## 四、缺陷 2：`balance_callback` 契约违反 ⇒ 整机冻结复位（opt158 修）

### 现象

- 偶发**整机冻结复位**、零内核日志、256 MB DDR 转储里 `Oops/BUG/panic` 全 0；
- 三次同源现场，断言位置各不同但都是同一条 `brk #0x800`（`SCHED_WARN_ON`）：

| 事件 | 内核 | 断言位置 |
|---|---|---|
| t162 | opt150 | `ext_ctrl+0xc68` |
| t176 | opt154 | `ext_ctrl+0xcc0` |
| t177 | opt155 | `__schedule+0x7e8` |
| **t183** | **opt158** | **无**（已修复） |

### 根因（因果链）

```
RT 任务被切进 hmbird 类（切类循环里没有过滤）
  => hmbird_check_class_changed -> switched_from_rt          (rt.c:2632)
  => rt_queue_pull_task                                      (rt.c:2644)
  => queue_balance_callback 挂上 rt_pull_head               (rt.c:439)
  => core.c:2304-2305 契约：check_class_changed() 之后必须 drain
  => 我们只在循环外 drain（旧代码 :5058）
  => 循环内其它 CPU 的 rq_pin_lock 炸（sched.h:1679）        (brk #0x800)
```

关键旁证：**参照补丁全树零 drain 代码**，它的做法是**根本不产生回调** —— 它在自己的切类循环里做了同样的过滤（`refwork/ref/kernel__sched__ext.c:3141-3143`）。我们当时删掉了这个过滤。

### 修法

enable / disable 两个切类循环各加同一句过滤（**当前树**坐标：`hmbird.c:4792` 为 disable 循环、`hmbird.c:5196` 为 enable 循环，两处即全部调用点）：

```c
if (reject_change_to_hmbird(p, p->prio))
    continue;
```

> 对比 opt155 的「加 drain」思路：那是治标。wb-5（t85）指出两处真实缺陷：**HIGH** — opt155 的 drain 落在 `hmbird_tasks_lock` 临界区内（违反「锁内只记录、解锁后执行」）；**MED** — 单次 drain 不是不动点（`core.c:5201-5229` 在调 func 前就把 `rq->balance_callback` 置 `NULL`，drain 期间还能重挂）。这解释了为什么 opt155 只「部分有效」。

### 判据

t183（opt158，hmbird ON + 4 线程满载 60 s）：

```
[  150.371102] t183 BEGIN up=150 kernel=6.1.141-android14-11-o-ltcdz5-v1.1-opt158
[  155.747333] t183 ENABLED scx=1
[  222.751171] t183 RESULT balance_cb=0 ext_ctrl=0 task_rq_lock=0 __schedule=0 stall=0
[  225.494054] t183 ALL DONE boot_id=fbde553d-e91e-4db3-84a2-92bc82b1495d load=14.07
```

★五个判据全 0，且 `boot_id` 未变（没有崩溃、没有重启）★

## 五、缺陷 3：watchdog 周期性强切 governor（opt160 修）

### 现象

- 启用 hmbird 且 governor 为 `scx` 时，**每 60 s** governor 被 hmbird 抢回 `uag`，守护脚本（`POLL_SEC=4`）再设回 `scx`；
- 每次来回都走 `cpufreq_set_policy()`（`drivers/cpufreq/cpufreq.c:2571`）的**完整 exit+init+start**，`scx` 调速器被反复销毁重建（注销／重注册 walt 回调）；
- 实测一局 6m41s 重建 **6~7 次**，间隔均值 **61.4 s**（08:34:14~08:40:55）。

### 根因（档案行号，opt160 时代）

`kernel/sched/hmbird/hmbird.c:3737-3738`（旧行号）在 watchdog work 内无条件执行：

```c
if (atomic_read(&__hmbird_ops_enabled))
    hmbird_gov_switch_away();
```

原依据是 2026-10-10 20:46 的一次「hmbird 开 + scx → 硬挂」，但该依据已被证明不成立：同内核 opt138 几小时后的四格矩阵（含 `h1_scx`）60 样本判定「卡死=否」；且 20:46 的现场与「切类循环缺过滤」这个已知缺陷**无法区分**（该缺陷 opt158 才修），现场还有 4 个并发诊断脚本。wb-hm 的全表统计：hmbird+scx 共 9 次、成功 6 次，失败 3 次中 2 次已归因于别的缺陷。

### 修法

- 把上述周期性强切用 `#if 0` 包起来（保留原文与说明注释），`scripts/setlocalversion` 改 `opt159` → `opt160`；
- **保留**启用前的一次兜底（**当前树**：`hmbird.c:5043` 是 `hmbird_gov_switch_away()` 的**唯一**调用点，失败则拒绝启用并返回 `-EBUSY`）；
- 关闭时恢复原值的路径保留（**当前树**：`hmbird.c:3717` 是 `hmbird_gov_restore()` 的定义）。

### 判据

二进制核验（确认改动真的进了镜像）：

```
llvm-objdump -d --disassemble-symbols=hmbird_watchdog_workfn vmlinux | grep -c cpufreq  ⇒ 0
System.map: hmbird_gov_restore 仍在（ffffffc008158730）；ext_ctrl 里仍有 cpufreq_cpu_get（启用前兜底保留）
hmbird_watchdog_workfn 大小 0x3cc
```

上机侧：opt161b 的 t190 实测 `t42=0`（全程无强切）——见下一节。

## 六、共存验证：hmbird 与 scx 同时在线

### opt161b（200 s / 4→8 线程）

```
t190 RESULT stall=0 balance=0 disabled=0 warn=0 pc=0 t42=0 veto=0 maxrunq=22 maxload=21.84 maxtz=51200 gov=scx scx=1
t190 ALL DONE boot_id=5c5f10cc-afac-482a-b35d-ed36102f7dbe same=1 load=20.33
```

- 这是第一次在真机上验证「hmbird + scx 同时在线」穿过 120 s watchdog 窗口并持续 200 s 以上，无自禁、无 WARN/panic/饥饿；
- **D4 判定**：四次 `hmbird disabled finished` 每一次都在同一时刻有配对的 `scx_enable=0 -> hmbird_ctrl() rc=0, enabled=0`，且 `dmesg | grep -iE 'err_exit|err_idx|hmbird_ops_error'` 全空 ⇒ 都是**用户态主动关闭**，不是内核自动禁用；
- t188 否决消融（唯一变量 `hmbird_gov_veto`）：`A veto=off refuse_delta=0` → `B veto=Y refuse_delta=4` → `C veto=off refuse_delta=0`，阶段 B 的 4 条拒绝日志 `t93: refuse governor scx while hmbird is enabled (EBUSY)` 只出现在 B。

### opt162（133.2 s）

- 共存窗口：uptime `124.42 → 257.64` = **133.2 s**（跨 60 s watchdog 周期两次）；
- 退出侧：`sw_rec[1] = 4294956702 1 0 0`（`is_success=1`，`end_state=0`=`HMBIRD_DISABLED`）；M1~M4 全 PASS；
- 详细记录见 `docs/opt162-真机验证.md`。

### 顺序闸门（五步，可照抄）

依据：`hmbird.c:5043` 的 enable 联锁是 `hmbird_gov_switch_away()` 的唯一调用点，它**只在当前 governor 匹配 `scx` 时动作**。

1. 全 policy 写 `uag`，并硬断言每个 policy 都回读为 `uag`；
2. `echo 1 > /proc/hmbird_sched/slim_stats`；
3. `echo 1 > /proc/hmbird_sched/scx_enable`（此刻 governor 是 `uag` ⇒ 内核不会强切）；
4. 用 dmesg 断言 `enabled=1`，**且本轮 `t42:` 计数为 0**（governor 没被强切过）；
5. 全 policy 写 `scx`，并硬断言每个 policy 都回读为 `scx`。

完整操作与判据见 `docs/启用与还原-操作手册.md`。

## 七、已排除的嫌疑（每条都有现场证据）

| 嫌疑 | 排除方式 |
|---|---|
| partial 簇 / rescue / partial util 采样 | 把 `partial_enable` 恒置 0 后**仍然**出现同样现象 |
| `scx_gov` 调速器 | 全程实测调速器是 `uag`、从未变成 `scx_gov`，现象照样出现 |
| 野指针写 | 相应补丁被判定为**结构性 no-op**（4 个调用点全在 hmbird 自己的类回调内） |
| 两份 `slim_walt_ctrl` 状态不一致 | 模块那份是模块局部符号，**不共享、不互读** |
| `hmbird_ops` ABI 错位 | 设备模块里**根本没有**这些符号（全量 nm 扫描） |
| 内核侧 irq_work 全 rq 锁 | 已停用该排队后**仍然**出现 |
| 「WALT 与 hmbird 两套全速跑」 | 那 15 处版本让路**在设备二进制里根本不存在**（内核/模块全 0 命中） |
| `nr_running` 泄漏（`runq=901`） | 静态对账证明 `add_nr_running`/`sub_nr_running` 由同一个 `HMBIRD_TASK_QUEUED` 位门控（该位全树只有 set/clear 两个写点，其余只读）⇒ **不是泄漏**；同镜像复测 `runq` 只有 8~16 |
| 「hmbird 一开就坏」 | 三类缺陷修完后，200 s / 133.2 s 共存均通过（见第六节） |

## 八、未复现 / 未判定（未复现 ≠ 不存在）

1. **`runq=901`**（t177，opt155，4 线程）：同镜像 t179/t181 复测只有 8~14，**未复现**；t177 以 kernel crash 收尾，读数来自已损坏镜像；`load` 与 `runq` 同源（`loadavg.c:78-91` 的 `nr_active = this_rq->nr_running - adjust`）。**原因未定**。
2. **`hmbird_partial_util=Y`**：轻载 90 s 无异常；历史「opt142 置 0 后 8 s 内 running 2→22、61 s 自检 stall 并自禁」**未复现、也未被判死**。要判死需要一次**带 8 线程负载**的 Y 臂（未做）。
3. **policy7 的 `scaling_cur_freq` 恒 672000**：8 线程负载（load≈20）下不涨，**未判定**是 scx 没把大核拉起来，还是 cpu7 确实空闲／读数口径问题。
4. **`16:55 静默硬挂`是否可复现**：未验证。
5. **真实游戏负载下的启用/还原**：未做（t194 全程无游戏）⇒ 尚未在真实游戏负载下复测。
6. **`maxtz`（51.2 ℃）不是游戏场景温升**：只是 8 个自旋线程的产物，降温结论必须用真实游戏复测。
7. **`CONFIG_HMBIRD_SCHED_CORE` 的编译态**：`hmbird_stats` 里 `sw_rec` 段的渲染假定它编入；未查 `.config`。

## 九、被证伪的历史说法（本仓库最有价值的部分之一）

| 曾经的判断 | 现在的结论 | 依据 |
|---|---|---|
| 「hmbird 一开就坏」（启用即硬挂） | 不是必然：三类缺陷修复后 200 s 共存通过 | t190 / opt162 |
| 「`nr_running` 单向泄漏」 | **不泄漏**：`add`/`sub` 由同一位门控 | wb-2 静态对账（t84） |
| 「`hmbird_stats` 是可信诊断数据」 | 很长一段时间里它是**硬编码假数据**（`hmbird_sched_proc_main.c:324-370` 全是为 0 的字面量），是我们社区移植版带入的 stub；opt156 起改为调用真 `stats_print()` 并打开 `slim_stats` 门控 | t87/t88、opt156 |
| 「`scx_enable` 回读 0 ⇒ 没生效」 | 回读的是**意图值**，不是状态；真值看 dmesg 的 `enabled=`（`hmbird_sched_proc_main.c:285` 先发布意图、`:306-307` 失败不回滚） | t133 复核 |
| 「装 drain 就能修断言」 | 治标：不产生回调才是治本（opt158） | wb-5、wb-hm |

## 十、证据索引

**事实档案（`F:\工作区\_audit\`）**

| 档案 | 用到的地方 |
|---|---|
| `成功-opt158修复balance_callback断言.md` | 缺陷 2 的根因/修法（`:33-51`）、t183 判据（`:7-29`）、三次同源表（`:72-82`） |
| `成功-opt159修复饿死缺陷.md` | 缺陷 1 的定量根因（`:54-72`）、t184 实测（`:7-23`）、历史对照（`:27-34`） |
| `成功-opt160停用周期强切governor.md` | 缺陷 3 的改动与理由（`:11-28`）、二进制核验（`:40-44`） |
| `成功-opt161b上机验证hmbird与scx稳定共存.md` | t190 RESULT（`:44-49`）、顺序闸门（`:23-30`）、t188 消融（`:78-96`） |
| `成功-opt162上机验证.md` | 镜像/刷机（`:6-10`）、ENTER/EXIT 与 M1~M4（`:22-44`）、`partial_util=Y`（`:46-66`）、`Call trace:` 澄清（`:68-88`） |
| `成功-t194游戏态启用与退出完整还原.md` | 守护模块 = scx governor 唯一下载器（`:14-18`）、132 s 实测（`:34-66`）、M1~M4（`:67-77`）、遗留（`:85-92`） |

**内核坐标（当前树 HEAD `7cc87ec0d8dc`，分支 `opt162`，已逐行核对）**

| 坐标 | 内容 |
|---|---|
| `hmbird.c:70` | `HMBIRD_WATCHDOG_MAX_TIMEOUT = 120 * HZ` |
| `hmbird.c:3486` | stall 判据 `time_after(jiffies, last_runnable + hmbird_watchdog_timeout)) \|\| watchdog_enable` |
| `hmbird.c:3628` / `:5043` | `hmbird_gov_switch_away()` 定义 / **唯一调用点** |
| `hmbird.c:3717` | `hmbird_gov_restore()` 定义 |
| `hmbird.c:4373` | `DEFINE_SCHED_CLASS(hmbird) = {` |
| `hmbird.c:4743` | 关闭路径 `cancel_delayed_work_sync(&hmbird_watchdog_work)` |
| `hmbird.c:4792` / `:5196` | disable / enable 切类循环里的 `reject_change_to_hmbird(p, p->prio)` 过滤 |
| `hmbird.c:5095` / `:5099-5100` | 启用路径设超时 / 首次排队 `timeout/2` |
| `hmbird.c:3841-3842` | watchdog workfn 自重排 `timeout/2` |
| `hmbird_sched_proc_main.c:270` / `:285` / `:299` / `:306-307` / `:319` | `scx_enable` 写、发布意图、`enabled=` 判据串、失败不回滚、节点注册 |

> ⚠️ 行号会随 opt 漂移，**连内核自己的注释都在引过期行号**（`hmbird.c:3805` 仍在引 `hmbird.c:4579`；`:3669-3671` 的注释仍称 `hmbird_gov_switch_away()` 有两个调用点，第二处已由 opt160 删除）。引用前请重新 `grep`。
