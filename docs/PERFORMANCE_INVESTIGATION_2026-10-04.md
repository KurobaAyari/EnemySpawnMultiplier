# EnemySpawnMultiplier 性能调查（2026-10-04）

## 结论

反馈中的卡顿风险是可信的，但“70 ms、峰值 230 ms 全部由刷怪倍率逻辑造成”目前不能只凭 watchdog 的一张截图定案。代码审计已经确认一个高风险热点：Panel 版本的尸体快速消失功能在固定锚点失效后，会持续扫描整个实体内存区。旧实现每 0.5 秒读取最多 4 MiB，扫到末尾后立即从头开始；实体记录数据的最大已知锚点约为 `0x11E3200`（17.9 MiB），这意味着约 5 个大块、约 35.8 MiB/s 的跨进程读取和字符串匹配，而且没有 CPU 时间预算。

刷怪参数本身的正常路径每 0.1 秒运行一次，但它主要读取导演器、配置和候选表；没有全地址空间扫描。它仍可能在资源表重建、克隆或写回失败时变重，不过静态代码不足以把它判定为 230 ms 的唯一来源。

## watchdog 证据如何解读

`research/Mod Lag Watchdog` 的 payload 是 `mods/patpatpatrick/mod_lag_finder`，版本 `0.7.1`。它把 update 链上的 Lua wrapper 拆开，用 `debug.getupvalue/setupvalue` 找到每个 wrapper 的 previous 函数，再用 timing spy 计算各链路的时间差。它把 50 ms 以上的帧记录下来，只有某个模块占据约 60% 以上时才显示 150 ms 以上的告警。

这足以作为“长帧期间该模块在 update 链上占了主要时间”的强证据，前提是日志显示链已经完成映射。它不是零开销工具：启动和重映射会修改 upvalue，稳定运行时仍每帧执行计时、链统计、UI 状态检查；它自己的 wrapper 位于链顶端时，其自身耗时不计入任何 mod。日志也采用批量写入，但刷新仍可能受磁盘延迟影响。因此应使用 watchdog 的 60 秒 summary、链映射状态和同一场景 A/B 结果，而不是只看一条 toast。

## 已确认的问题

### 1. 动态尸体扫描是持续全量扫描

旧代码在 `src/corpse_clear.lua` 中使用 `PASS_INTERVAL = 0.5` 和 `DYNAMIC_CHUNK = 4 * 1024 * 1024`。`dynamic_pass()` 扫完 `region_size` 后把游标清零，下一轮继续扫描；没有“已完成后休眠”、失败退避或 CPU deadline。`windows_api.lua` 的 `api.read()` 每次还会分配缓冲区并调用 `ReadProcessMemory`。

### 2. 运行状态日志曾被性能采样字段强制打满

`src/archive_loader.lua` 的 `report()` 把每次变化的 `upd_ms/win_ms` 拼入 `state.detail`，因此 `state.detail` 几乎每个 0.1 秒都会变化，触发 `io.open(..., 'a')`、seek、写入和 close。现在性能采样只附加到已经节流的日志写入，不再把采样值当成状态身份。

### 3. 失败时的全地址空间扫描没有终止态

旧实现找不到实体表时会从 `LOW_LIMIT` 扫到 `HIGH_LIMIT`，到末尾后重新开始。现在连续三轮没有可写实体表会进入 `entities_scan_gave_up`，停止继续消耗帧时间；重新启用功能可再次尝试。

## 已实施的修复

- 动态读取块从 4 MiB 降为 256 KiB。
- 在模式签名和 Decayer 签名匹配内部加入 2 ms CPU 时间片；游标和匹配位置会跨 update 保存，不会因让出时间而漏掉块内记录。
- 一次动态全量扫描完成后进入稳定态；每 5 秒只检查表头和最多 8 条已知记录，只有发现表重建或记录失效才重新全量扫描。
- 连续空手的地址空间扫描最多三轮，之后进入终止状态。
- 性能采样从状态变化判据中移除，避免每帧文件 I/O。

## 验证

LuaJIT 离线测试通过：

- `tests/test_corpse_clear.lua`：13 项通过，新增测试确认动态扫描完成后 8 秒内不会自动重新全量扫描。
- `tests/test_panel_loader.lua`：8 项通过，确认真实 loader、日志 writer 和 Panel update 链仍可运行。
- `loadfile()` 解析 `corpse_clear.lua`、`archive_loader.lua`、`spawn_patch.lua` 通过。

这些测试没有替代实机帧时间测量。最终归因仍需要两组相同任务、相同配置的 watchdog summary：Panel + 快速清尸开启、Panel + 快速清尸关闭；并记录链映射完成、60 秒平均 `ms/s`、最坏帧和日志写入最慢值。若关闭快速清尸后 70/230 ms 峰值消失，热点基本可以确认；若仍存在，应继续对 `spawn_patch.apply()` 的资源克隆和候选表写回做单独 A/B。
