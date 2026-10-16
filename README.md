# 风驰 for 真我 GT5 Pro（自编内核）

**一句话**：让自编内核上的「风驰游戏内核」真正生效 —— 打开游戏自动切到风驰调度，退出游戏自动回到系统原本的调度。

> 作者：**github@ltcdz5** ／ **酷安@天玑1100逆天功耗** ｜ 由 DeepSeek 协助整理

## 目前做到哪一步（2026-10-17 更新）

> 这一节是本仓库**唯一**的进度口径；下面「怎么用」那几节仍然有效，但请先读这里。

### ⏸ 2026-10-17 更新：★第二条路（内核态 hmbird 调度类）已暂缓★

**放弃原因：工程量大、难度高。** 完整记录见
→ [`docs/opt163-opt172-收尾结论与暂缓-20261017.md`](docs/opt163-opt172-收尾结论与暂缓-20261017.md)

三条叠在一起：① 要让调度真变好，得同时复现**五段链**（内核 `sched_class` + 5 个厂商 `.ko` +
3 个用户态服务 + 参数库 + 云控注入），少一段数据就是假的 —— 本轮光"把官方桥修好"就占掉大半轮次，
而修好后真实对局 **FPS 46.53 仍不过**；② 核心症结（hmbird 在线时 cpu5/6/7 连 `taskset` 都跑不动）
**没能定位到机制**，三个用户态开关全无效，而「内核里摘掉 sched_class」这条退路又被 ABI 堵死
（`CONFIG_HMBIRD_SCHED_CORE=n` ⇒ 3 个导出 CRC 改变 + 1 个导出消失 ⇒ 蓝牙与厂商 sched_ext 双双拒载）；
③ 效应量 ≤1~2%，本机噪声带 I/O 20% / 功率 CV 41% ⇒ **改对了也测不出来**。

**⇒ 本仓库现在只有第一条（风驰游戏切换）还在维护。** 第二条第 3 项的守卫模块下一版**未发布、未上机**，
随本方向一起搁置。

### ✅ 第一条（本仓库主线）：风驰游戏切换

`opt60` 起的自编内核 + `fengchi-boot` 模块 —— 进游戏自动切到厂商的 `scx` 调速器、退出回系统默认 `uag`。

- 实测：**第五人格、绝区零** 均正确切换；重负载稳定；`Oops / BUG / panic = 0`
- 边界（opt60 基线核对，此后各 opt 未逐一复核 ⇒ **未验证**）：内核导出集 **15,489** 逐名未变；493 个厂商模块 `missing=0 / crc=0`
- 已知可用的干净回退件：`boot-v1.1-opt60-repacked.img`，md5 `5fd7909866e0de04b8e46cd9b388cc2e`

★一条必须记住的架构事实★：`scx` 调速器由**厂商模块** `oplus_bsp_sched_ext.ko` 提供，而设备上**唯一 insmod 它**的就是本仓库守护模块的 `service.sh`（依赖序 `oplus_bsp_game_opt` → `oplus_bsp_sched_assist` → `oplus_bsp_sched_ext`）。
⇒ **停用守护模块 = 顺手把 `scx` governor 一起删掉**（`scaling_available_governors` 里就没 `scx` 了）。

### ✅ 第二条（2026-10-13 新打通）：内核态 hmbird 调度类

指 `/proc/hmbird_sched/scx_enable=1` 那条路（把任务真正切进内核里的 hmbird 调度类）。
**状态：从「不可用」变成「能用」** —— 能开、能长时间稳定、能关干净，并且能与厂商 `scx` 调速器**同时在线**。

当前验证内核：`6.1.141-android14-11-o-ltcdz5-v1.1-opt162`

```
boot-v1.1-opt162-repacked.img   md5 80a9c6251d9012241bb392f21e0a5e38   (201,326,592 B)
Image.opt162.crc                md5 918c4c8551b632d9a01063282890017b
```

**修掉的三个真实缺陷**

| # | 缺陷（启用后的现象） | 修在 | 现场判据 |
|---|---|---|---|
| 1 | **任务饿死**：`running` 1→20~40、任务全压 CPU0/1/2（大核超大核空闲），约 60 s 后自禁 | `opt159`：T2 簇可见性 —— `gen_cluster_ctx_separate` 给 BIG 的窗口是 `[5,8)`、LITTLE 是 `[8,10)`，**各簇只能扫一半**；只放宽超时兜底路径（`consume_timeout_dsq` 的 `ctx.lower/upper` = 5/10） | t184：8 线程满载 **200 s**、跨过 120 s 看门狗阈值，`stall=0 balance=0 disabled=0`，`runq=10~16`，`boot_id` 未变 |
| 2 | **`balance_callback` 契约违反** ⇒ `brk #0x800` 硬挂（`__schedule+0x7e8` 那条断言） | `opt158`：我们漏了参照补丁的 `reject_change_to_scx` 过滤 —— enable/disable 两个切类循环各加 `if (reject_change_to_hmbird(p, p->prio)) continue;`（当前树 `hmbird.c:4792` / `:5196`）⇒ **不产生回调**，于是不需要 drain | t183：`balance_cb / ext_ctrl / task_rq_lock / __schedule / stall` **五项全 0**，`boot_id` 未变 |
| 3 | **每 60 s 周期性强切 governor** ⇒ `scx` 调速器被反复销毁重建（一局 6m41s 重建 6~7 次、间隔均值 61.4 s） | `opt160`：把 watchdog work 里的周期性强切（第二调用点）用 `#if 0` 停掉，**只保留启用前那一次兜底**（当前树 `hmbird.c:5043` 的 `hmbird_gov_switch_away()`） | t190：`t42=0` 全程、`gov=scx` 全程 |

**共存与还原（都是真机实测）**

- `opt161b`（t190，4→8 线程共 200 s）：`stall=0 balance=0 disabled=0 warn=0 pc=0 t42=0 veto=0 maxrunq=22 maxload=21.84 gov=scx scx=1`，`boot_id` 未变
- `opt161b` 上「进入 → 保持 → 退出」整周期（t194，132 s 共存）：退出后 M1~M4 **四项判据全 PASS**，`boot_id` 不变
- `opt162` 上同一套 enter/exit 脚本：hmbird + scx **同时在线 133.2 s**（uptime 124.42 → 257.64，跨 60 s 看门狗周期两次），M1~M4 全 PASS
- 退出终态：governor 全 `uag`、`scx_enable=0`、`slim_stats=0`、`boot_id` 不变

**★两条最容易踩的判据口径（务必照做）★**

1. **退出还原必须由用户态显式做**：内核**不会**自动把 governor 写回。正确顺序（先 `uag` → 开 hmbird → 再 `scx`）下 `hmbird_gov_switch_away()` 整块跳过，`hmbird_orig_gov` 从未被写；退出时 `hmbird_gov_restore()` 又因空判据整段跳过 ⇒ 脚本必须自己写 `uag`，且 `echo 0 > scx_enable` 之后**再补写一次**。反过来（顺序被打乱）没有内核兜底：启用时若 governor 已是 `scx`，内核会把 `scx` 记为原值并**当场**写回 `scx` + 清空（该 enable 因 `t42` 联锁被拒，`hmbird.c:5058`）；启用时若 governor 是 `uag`，内核什么都不记录、退出时也不写任何 governor。两种走法的终点都是 governor 残留 `scx`（裸 `scx`），内核不会帮还原 ⇒ 顺序闸门必须**硬断言**，不能只写注释。
2. **`scx_enable` 是意图值，不是状态**（`hmbird_sched_proc_main.c:285` 先发布意图、`:306-307` 失败不回滚）⇒ `cat` 回 0 **不证明** hmbird 已关。真实态只看 dmesg 的 `scx_enable=%d -> hmbird_ctrl() rc=%d, enabled=%d`（`:299`）或 `/proc/hmbird_sched/hmbird_stats` 里环形 `sw_rec` 的最新一条。

**还剩下的边界（诚实说）**

- `hmbird_partial_util`（opt162 新增旋钮）：默认 `N` 与 `opt161b` 行为一致；置 `Y` 目前只测到**轻载 90 s 无异常**，历史记录（opt142 曾「8 s 内 running 2→22、~61 s 自检 stall 并自禁」）**未复现、也未被判死**（要判死需要一次带 8 线程负载的 Y 臂）。
- 8 线程负载下 policy7 的 `scaling_cur_freq` 恒 672000，与 `load≈20` 不相称 —— **未判定**是 scx 没把大核拉起来，还是读数口径问题。
- **真实游戏负载下尚未复测**（t194 全程无游戏）；`16:55 静默硬挂` 是否可复现也**未验证**。
- 对非游戏应用的调度影响，目前只用四项统计判据（M1~M4）覆盖，**长时日常使用的对比未做**。

### 🔭 下一步

1. **真实游戏态复测**：把负载从合成自旋线程换成真实游戏，跑一遍「进游戏启用 → 退出完整还原」。
2. **判据收窄**：`warn` 只数 `WARNING:`，别把 `Call trace:` 算进去 —— 厂商 TEE 每 ~61.4 s 打一条周期栈（`__switch_to` → `__schedule` → `qcom_scm`），与调度无关、任何内核上都有。
3. **守护模块下一版**：`D4 版本闸门` + `F1 内核自禁盲区` 两处已修——**是否发布 / 上机未验证**，不写进本篇进度。
4. **不再走上游官方 sched_ext（BPF struct_ops）路线**（结论不变）：`bpftool struct_ops register` 一加载就硬挂死整机（2026-10-03），`opt43` 的「部分模式」同样硬挂；该路线的失败面至今没有解法。

### ⚠️ 一条必须说清楚的区分（两条路的关系）

本仓库涉及**两条不同的路**，2026-10-13 起**两者都可用，而且可以同时在线**：

| | 到底是什么 | 怎么看它生效 |
|---|---|---|
| 厂商 `scx` 调速器 | 厂商模块 `oplus_bsp_sched_ext.ko` 注册的 **cpufreq governor** | `cat /sys/devices/system/cpu/cpufreq/policy*/scaling_governor` 显示 `scx` |
| 内核态 `hmbird` 调度类 | 我们编进内核的 **`sched_class`**（`DEFINE_SCHED_CLASS(hmbird)`，当前树 `hmbird.c:4373`） | `/proc/hmbird_sched/scx_enable=1` + dmesg 的 `enabled=1` |

下面「怎么确认真的生效」那一节说的是**第一条**（governor）；不要用它的方法去判断第二条。

## 怎么确认真的生效

进游戏后看 CPU 调速器（命令或直接用 Scene 看都行）：

```
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
```

显示 `scx` = 风驰已生效 ✓；退出游戏后回到系统默认（本机是 `uag`）✓（这一步由守护脚本显式完成：内核不会自动写回 governor）

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