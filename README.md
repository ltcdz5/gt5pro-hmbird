# 风驰 for 真我 GT5 Pro（自编内核）

**一句话**：让自编内核上的「风驰游戏内核」真正生效 —— 打开游戏自动切到风驰调度，退出游戏自动回到系统原本的调度。

> 作者：**github@ltcdz5** ／ **酷安@天玑1100逆天功耗** ｜ 由 DeepSeek 协助整理

## 目前做到哪一步（2026-10-12 更新）

> 这一节是本仓库**唯一**的进度口径；下面「怎么用」那几节仍然有效，但请先读这里。

### ✅ 已经能用的：风驰游戏切换（本仓库主线）

`opt60` 内核 + `fengchi-boot` 模块 —— 进游戏自动切到厂商的 `scx` 调速器、退出回系统默认 `uag`。

- 实测：**第五人格、绝区零** 均正确切换；重负载稳定；`Oops / BUG / panic = 0`
- 边界：内核导出集 **15,489** 逐名未变；493 个厂商模块 `missing=0 / crc=0`
- 回退件：`boot-v1.1-opt60-repacked.img`，md5 `5fd7909866e0de04b8e46cd9b388cc2e`

### ⚠️ 仍在攻克、**当前不可用**的：内核态 hmbird 调度类

指 `/proc/hmbird_sched/scx_enable=1` 那条路（把任务真正切进内核里的 hmbird 调度类）。

**两类现象，别混在一起：**

1. **饿死**（稳定可复现）：启用后 `running` 从 1 涨到 20~40，任务被压在 **CPU0/1/2**（大核与超大核空闲），
   约 60 秒后 hmbird 自检到任务饿死（`failed to run for ~58s`）并**自动禁用自己**，系统随后恢复。
   全程 **无 panic、无 WALT-BUG、无 PSI 不一致**。
2. **整机冻结复位**（偶发）：在**启用 62 秒后写 `scx_enable=0` 关闭**的瞬间，整机瞬时冻结并硬复位，
   **零内核日志**（连无关的充电刷屏日志都同时停止）。256 MB 完整 DDR 转储里
   `list_add double add` / `__list_add_valid` / `WALT-BUG` / `Oops` / `BUG:` / `panic` **全部为 0**。

**2026-10-12 新增的关键发现：看门狗 60 秒窗口**

- `kernel/sched/hmbird/hmbird.c:64-70`：`HMBIRD_WATCHDOG_MAX_TIMEOUT = 120 * HZ`
- 启用路径 `:4579-4584`：`queue_delayed_work(…, hmbird_watchdog_timeout / 2)` ⇒ **首次触发 = enable 后恰好 60 秒**
- 关闭路径 `:4326`：`cancel_delayed_work_sync(&hmbird_watchdog_work)`
- 实测对照：**每轮启用 62 秒**的实验（t157）在第 2 轮冻结；**每轮启用 10 秒**（t158，20 轮）与
  **30 秒**（t159，关闭后观察 15 分钟）全部通过 —— 但它们**没有跨过 60 秒窗口**，
  所以「全过」**不构成关闭路径安全的证据**。
- 已划掉一个嫌疑：设备实测 `/proc/hmbird_sched/watchdog_enable = 0`，
  看门狗的 stall 上报链（`check_rq_for_timeouts` → `hmbird_err_exit_workfn`）**不会触发**。

**已被排除**（每条都有现场证据）：partial 簇 / rescue / partial util 采样 · `scx_gov` 调速器 · 野指针写 ·
两份 `slim_walt_ctrl` 状态不一致 · `hmbird_ops` ABI 错位 · 内核侧 irq_work 全 rq 锁 ·
「WALT 与 hmbird 两套全速跑」· 关闭路径的 `list_add double add`（Stage BS/BQ 三件套已修，20 轮验证）·
00:55:41 那次重启（是 `system_server` 用户态主动 reboot，与 hmbird 无关）

**根因仍未定位。** 目前的判断是：问题更可能出在**我们这套调度类的放置策略**上，而不是「hmbird 一开就坏」。

### 🔭 下一步方向（已修正）

⚠️ **不再走上游官方 sched_ext（BPF struct_ops）路线。** 2026-10-11 之前的评估曾把它列为方向，
但更早的实测已经否决：`bpftool struct_ops register` 一加载就硬挂死整机（2026-10-03），
`opt43` 的「部分模式」同样硬挂。该路线的失败面（`ext.c` 的 `cpus_read_lock` 持锁窗口）
至今没有解法。

现在的方向是 **继续修内核态 hmbird 调度类**，重点是：

1. **看门狗窗口**：跑启用 ≥70 秒的对照实验，判定 60 秒点是否是触发条件（4 条可证伪判据见下）。
2. **放置策略**：`select_cpu` 的亲和性守卫（Stage BL）只守一个入口，`sticky_cpu` / `critical_affinity` /
   migrate 路径可以绕过；`hmbird_pick_idle_cpu` 传的是 `cpu_possible_mask` 而非 `p->cpus_ptr`。
3. **关闭路径加固**：给 `kernel/sched/fair.c:12442`（`set_next_task_fair` 的 `list_move`）补类守卫，
   给 `hmbird_sched_class` 补 `.switched_from` 摘 `group_node`。

**可证伪判据（下一轮）**

| 编号 | 判据 |
|---|---|
| W1 | 启用 ≥70 s × ≥5 轮，崩溃率显著高于 10 s 组 |
| W2 | 每使能秒崩溃率在 10/20/30/45/60 s 梯度上近似恒定（线性模型）|
| W3 | 关闭路径崩溃前 dmesg 有 `watchdog failed to check in for` |
| W4 | 把 `timeout/2` 改成 `timeout*4` 后 70 s 组不再崩 |

### ⚠️ 一条必须说清楚的区分

README 下面写的「显示 `scx` = 风驰已生效」，指的是**厂商模块注册的 `scx` 调速器**；
和上面那条「内核态 hmbird 调度类」是**两条完全不同的路**。
**前者可用，后者不可用**，请不要把两者的现象混在一起。

## 怎么确认真的生效

进游戏后看 CPU 调速器（命令或直接用 Scene 看都行）：

```
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
```

显示 `scx` = 风驰已生效 ✓；退出游戏后回到系统默认（本机是 `uag`）✓

## 怎么用（三步）

1. **刷入配套的自编内核**（6.1.141 OKI 基线，源码见「内核源码」仓库）
2. **安装 `fengchi-boot` 模块**（本页右侧 Releases 下载）
3. 把游戏**加进「游戏助手」**，然后直接开游戏 —— 不需要手动切任何东西

## 这个模块做了什么

- **开机**：把厂商的 `scx/hmbird` 相关模块按顺序加载好（风驰要用的 `scx` 调速器就在这里注册）
- **玩的时候**：检测到游戏就切 `scx`；退出游戏就切回系统默认
- **顺便**：恢复 MGLRU、拉起 horae（原来要装两个小模块，现在一个就够）

## 三个坑（务必看）

1. **不要用任何工具去「固定」风驰调度** —— 会让手机直接死机（我们实测过）
2. **卸载「官方调度屏蔽」类模块**（例如 `IMS_VAROS`）：它会清空官方调度相关属性，风驰就没法工作
3. **关掉 Scene 的调度/调速器设置**：可能与风驰冲突（未做完整对比测试）

## 支持哪些游戏

以厂商云控适配为准。SM8650 机型已知覆盖数十款：原神、崩坏星穹铁道、绝区零、鸣潮、王者荣耀、和平精英、英雄联盟手游、PUBG Mobile、穿越火线、使命召唤手游、暗区突围、三角洲行动、永劫无间、无畏契约手游、第五人格、阴阳师、金铲铲之战、光遇 等。

另外：游戏需要**在「游戏助手」列表里** ✓

## 相关仓库

- 内核源码：**gt5pro-kernel-src**
- 工具箱与全部过程档案：**gt5pro-kernel-kit**

## 许可

GPL-2.0。个人自编自用，仅供交流学习。