# gt5pro-hmbird

真我 GT5 Pro（RMX3888 / SM8650 / ColorOS 16 / 6.1.141 **OKI 基线**）上 **hmbird（风驰）**的移植记录 —— **已在自编内核上实测生效** ✓

> 本仓库内容由 **DeepSeek** 协助机主整理撰写。

## 结论（2026-10-09 实测）

**风驰在自编内核上跑通了，而且是全自动的** ✓

~~~
第五人格在前台时：
  /sys/devices/system/cpu/cpufreq/policy0/scaling_governor = scx ✓
  /sys/devices/system/cpu/cpufreq/policy7/scaling_governor = scx ✓
  critical_task = UnityMain UnityGfxDevice ✓（OPGS 正确识别关键线程）
  稳定性: Oops=0 BUG=0 panic=0 ✓
~~~

## 完整链路（五环）

| 环节 | 做法 |
|---|---|
| ① 内核提供 `scx` 调速器 | 本项目自编 6.1.141 内核（导出集 **15,489** 逐名未变、493 个厂商模块不受影响）+ 厂商 `oplus_bsp_sched_ext.ko` |
| ② 厂商栈自动加载 | `/data/adb/service.d/99-fengchi.sh`（开机自愈：起 HAL 服务、按依赖序 insmod `game_opt`→`sched_assist`→`sched_ext`）|
| ③ OPGS 关键线程通路 | SCRC 的 `kmodule`（`ctn_patch.ko` + `opgs_daemon` ⇒ 补出 `/proc/game_opt/task_boost/critical_task_name`）|
| ④ 云控配置 | SCRC v5.6 `bin/inject`（往 COSA 库注入 SM8650 共 **49 款游戏**的风驰配置）|
| ⑤ **游戏触发切换** | `/data/adb/fengchi-gov.sh` 守护：检测到游戏 ⇒ 写 `scx`；退出 ⇒ 写回 `uag` |

## 为什么需要②⑤（重要）

自编内核上，**厂商的"按需自动加载"头部不会被触发** ✗（`oplus_bsp_game_opt`/`oplus_bsp_sched_ext` 一直不加载 ⇒ `scx` 调速器不注册 ⇒ 后面整条链全断）。②把这一环补齐；⑤补上"游戏触发切换"这一动作。

## 必须遵守

- ⛔ **必须卸载 `IMS_VAROS`（"官方调度屏蔽模块"）**：它清空 `persist.sys.orms.name`/`oiface.feature`/`hardcoder.name` 并禁 `OrmsUrcc`/`OiFace`/**`GameOpt`**/**`COSA`** ⇒ 与风驰直接冲突
  - ⚠️ 卸载要用 `ksud module uninstall`（**改名目录会导致 KSU 卸载失败** ✗）；卸载+重启后属性才恢复为 `ORMS`/`oiface:1f,oifaceim:ffffffff`/`oiface`
- ⛔ **关掉 Scene 的调度**（调度/CPU 亲和/线程绑定）；不要用任何第三方调度或游戏线程模块
- ⛔ **不要用内核开关/搞机工具去固定 scx 调度** —— **会整机硬挂死**（本项目实测复现过；社区也明确警告）
- 对外发布的镜像请保持 `CONFIG_HMBIRD_SCHED_CORE=n`（默认即 n）

## 支持的游戏（SM8650 档，SCRC v5.6 共 49 款）

原神(官/B/国际) / 崩坏星穹铁道 / 绝区零 / 鸣潮(官/B/国际) / 战双帕弥什 / 王者荣耀(官/体验/国际) / 和平精英 / 英雄联盟手游 / PUBG Mobile(6 服) / 穿越火线 / 使命召唤手游 / 暗区突围 / 三角洲行动 / 永劫无间 / 无畏契约手游 / 逆战未来 / QQ飞车 / 火影忍者 / 金铲铲之战 / 第五人格(3 服) / 香肠派对 / 阴阳师 / 光遇 / 迷你世界 / 对峙2 / 高能英雄 / 失控进化 / 洛克王国 / 萤火突击 / 异环 …

## 怎么判断风驰生效

~~~
cat /sys/devices/system/cpu/cpufreq/policy*/scaling_governor
~~~
游戏运行时出现 `scx` ⇒ 生效 ✓（退出游戏后回到 `uag` 属正常）

## 现状

| 项 | 状态 |
|---|---|
| 内核 `scx` 调速器 | ✅ 可注册/可切换/可启动（真机实测）|
| 接口层 `/proc/hmbird_sched` | ✅ 21 条（装厂商模块后 27 条）|
| ABI | ✅ 导出集 15,489 逐名未变；493 模块 missing=0/crc=0 |
| 全自动链路 | ✅ 开机自愈 + 游戏触发切换（实测生效）|
| 内核开关强开（`scx_enable=1`）| ✗ **硬挂死**（不要用）|

## 方法（简述）

1. 以**出厂内核**为基准：符号、CRC、结构体布局逐条比对（BTF 逐字段对偏移）；
2. 以设备上的 **493 个厂商模块**为硬判据：全量审计"能否装载"；
3. 每步过三道闸门 + 真机验收。

详见 [docs/方法.md](docs/方法.md) 与 [docs/measured-2-20261009.md](docs/measured-2-20261009.md)。

## 借鉴与致谢

见 [docs/来源与致谢.md](docs/来源与致谢.md)（含各来源仓库链接与许可证状态；**无明确许可证的来源只作引用与致谢，不包含其代码**）。
特别致谢 **SC​RC / 星海亦有岸**（`xhai-git/SCRC`）—— 云控注入与 OPGS 关键线程通路。

## 许可

GPL-2.0
