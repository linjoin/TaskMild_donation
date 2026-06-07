# TaskMild v7.3.0 更新日志

## v7.3.0 — 八维评分架构重构

### 新增

- 增加八维评分算法 v73(PSS/CPU/服务子进程/IO/僵尸/OOM/年龄/压力加成，总分上限从 90 提升至 201)
- 增加 OOM 评分维度(将 oom_score_adj 线性映射为 0~25 分，与 LMKD 优先级逻辑对齐)
- 增加进程年龄评分维度(基于 FgExitTracker 追踪前台退出时间，后台越久评分越高，阶梯式 0/5/10/15/18/20)
- 增加内存压力自适应模型(GREEN/YELLOW/RED 三级压力，影响评分加成和阈值系数)
- 增加主进程四级保护体系(PROTECT/GUARD/WARNING/DANGER，替代旧版主进程无条件保护)
- 增加 v73 信号策略(GUARD→SIGTERM 温和保护，WARNING→SIGTERM 警告级，DANGER→SIGKILL 危险级)
- 增加包级冷却追踪(PackageCooldown 跨 PID 追踪同一应用的杀进程频率)
- 增加指数递增冷却(base×2^(kill_count-1)，从 3 分钟递增至 30 分钟封顶)
- 增加 Decorrelated Jitter 冷却抖动(xorshift32 PRNG ±25% 随机抖动，防止雷鸣群效应)
- 增加包级冷却衰减(30 分钟无杀操作自动减 1，恢复正常杀节奏)
- 增加 SWAP 分层热重载(VM 参数即时生效，核心配置通知用户重启)
- 增加 PSI 内存压力事件源(第 8 个事件源，epoll 触发器 + timerfd 轮询降级)
- 增加 PSI write trigger 完整实现(写入 some 阈值触发条件 + epoll_ctl 注册 + drain/rearm)
- 增加 MemTotal 全局缓存(init_mem_total 启动时读取 /proc/meminfo，用于压力模型计算)
- 增加 FgExitTracker 前台退出时间追踪(on_fg_change 记录切换，bg_elapsed_secs 查询后台时长)
- 增加 KillEntry 信号策略提示字段(signal_hint: SignalPolicyV73，按 v73 策略选择 SIGTERM/SIGKILL)
- 增加 age_zombie_overlap_discount 年龄/僵尸重叠折扣(较小项打半折，避免双重计分)
- 增加 PREV_SWAP_CONFIG 全局缓存(修复 Swap 热重载 old_swap == new_swap 的比较 bug)
- 增加 v73 评分/冷却/压力/决策完整单元测试

### 修复

- 修复前台检测精度不足的问题(oom=0 为真正前台，oom=100 为降级前台，oom=200 不再误判为前台)
- 修复主进程无条件保护导致大内存主进程永远不会被杀的问题(GUARD/WARNING/DANGER 分级替代)
- 修复 Guard 乘数方向反转的问题(score×3/2 放大改为 score×2/3 缩小，等价阈值×1.5)
- 修复 Warning/Danger 信号策略未传递的问题(execute_kills 按信号策略选择 SIGTERM/SIGKILL)
- 修复 Swap 热重载 old_swap 与 new_swap 读取同一文件导致变更永远检测不到的问题(缓存旧值到 PREV_SWAP_CONFIG)
- 修复 funnel.rs 缺少 Instant 导入导致编译失败的问题
- 修复 auth.rs notify() 为私有函数导致 events.rs 无法调用的问题(改为 pub fn)
- 修复 FgExitTracker MutexGuard 生命周期冲突导致编译失败的问题(分离 let 语句避免临时值 Drop 冲突)
- 修复 5 处 unsafe 块缺少 SAFETY 注释的问题(swap.rs 3处 + events.rs 2处)
- 修复 bg_elapsed_secs 未从 FgExitTracker 填充的问题(get_detail_info 中查询 fg_exit_tracker)
- 修复 zombie_hours 始终为 0 的问题(从 frozen_since HashMap 计算冻结时长)
- 修复 IO bytes 始终为 0 的问题(RSS>20MB 进程调用 calculate_io_delta 计算 read/write delta)
- 修复 is_beast_exception 始终为 false 的问题(内存压力 <3000MB 且 PSS 超过阈值时标记巨兽例外)
- 修复 EvalContext 缺少 mem_total_mb 字段的问题(从全局 MEM_TOTAL_MB 读取)
- 修复 v73 动态阈值未使用的问题(algorithm_version≥73 时使用 calculate_dynamic_threshold_v73 含压力系数)
- 修复 decay_package_cooldowns() 全局未被调用的问题(press_background 每轮执行衰减)
- 修复 record_package_kill() 未被调用的问题(execute_kills 中每次杀进程后记录包级冷却)

### 优化

- 优化评分权重重构(PSS 上限 15→30，CPU 上限 10→15，IO 上限 20→25，僵尸上限 30→40)
- 优化前台检测为两级体系(优先 oom=0 FOREGROUND_APP，其次 oom=100 VISIBLE_APP 降级前台)
- 优化可见应用收集(oom>0 且 oom≤200 才归入可见集，oom=0 不算可见)
- 优化 PIP 检测精度(oom≤0 不算 PIP，oom>200 不是活跃 PIP)
- 优化冷却机制从固定延长改为指数递增(120s/300s → 3min/6min/12min/24min/30min 封顶)
- 优化冷却有效杀次数计算(取 PID 级和包级中较大值，避免换 PID 绕过冷却)
- 优化 FgExitTracker 定期清理(cleanup_stale_entries 中清理超过 24 小时的旧记录)
- 优化 2GB 设备支持移除(2GB 设备市场份额仅 1.7%，专注维护 8GB+ 用户)
- 优化 dumpsys 全部移除(纯事件驱动架构：cgroup inotify + logcat watcher + PSI epoll)
- 优化二进制编译选项(LTO=fat, codegen-units=1, panic=abort, strip=symbols, opt-level=3)

### 变更

- 变更算法默认版本从 v72 升级到 v73(algorithm_version 默认值 73)
- 变更 OOM 候选门槛从 oom<100 跳过改为 oom<0 系统进程跳过 + oom==0 永远保护
- 变更评分公式从五维(PSS/CPU/子进程/IO/僵尸)扩展为八维(+OOM/年龄/压力加成)
- 变更主进程决策从 Decision::Protect 改为 Decision::Guard/Warning/Danger 分级
- 变更冷却基础时间从 120 秒提升到 180 秒
- 变更冷却封顶时间从 300 秒提升到 1800 秒(30 分钟)
