# 真机验证记录（opt60）

设备：真我 GT5 Pro（RMX3888 / SM8650 / ColorOS 16）｜内核 6.1.141 **opt60**（boot md5 `5fd7909866e0de04b8e46cd9b388cc2e`）

## 一、内核侧
```
/proc/hmbird_sched  存在，21 条：
cpuctrl_high cpuctrl_low heartbeat heartbeat_enable hmbird_stats hmbirdcore_debug
iso_free_rescue isoctrl_high_ratio isoctrl_low_ratio isolate_ctrl misfit_ds
parctrl_high_ratio parctrl_high_ratio_l parctrl_low_ratio parctrl_low_ratio_l
partial_ctrl save_gov scx_enable slim_for_app slim_stats
```
DT 兼容层按设计安全降级：
```
schedtune: hmbird_sched: no /soc/oplus,hmbird node in DT; version_type stays UNKNOWN (safe degrade)
```

## 二、厂商模块侧
按依赖序装载 rc=0；装载后 **27 条**（+ `cpu7_tl highres_tick_ctrl_dbg scx_shadow_tick_enable slim_freq_gov slim_walt yield_opt`）。
```
[scx_gov][assign_cluster_ids] cluster[0]→0 cluster[5]→2 cluster[2]→1 cluster[7]→3
[scx_gov][scx_cpufreq_init] num_cluster=4  id=0 cpumask=0-1 cap=379
                                           id=1 cpumask=2-4 cap=923
                                           id=2 cpumask=5-6 cap=867
                                           id=3 cpumask=7   cap=1024
```
`scx_enable` 权限 `0666`。

## 三、启用测试
`echo 1 > /proc/hmbird_sched/scx_enable` ⇒ **rc=0，系统未挂死**（uptime 持续增长；8×`yes` 轻负载 5 秒通过）。
但 `scx_enable` 回读 0、`<hmbird_sched>` 计数 0 ⇒ **调度未真正生效**，指向 fork 核心（`gdsqs`/`pcp`/`partial`）未搬入。

## 四、两条 WARNING（厂商侧）
- 2.0 s `proc_dir_entry '/proc/oplus_mem' already registered` ×6 —— 厂商模块重复注册；
- 86.7 s `tracepoint_add_func` WARNING，`Comm: autochmod.sh`（厂商自己的模块加载脚本）—— 同类冲突在 boot 1.5 s 已存在（`qcom_lpm: exports duplicate symbol …scx_sched_lpm_disallowed_time`）。

系统稳定：无 panic、无重启、蓝牙 ON。
