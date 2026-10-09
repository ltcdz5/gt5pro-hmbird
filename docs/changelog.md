# 更新日志（Changes）

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
