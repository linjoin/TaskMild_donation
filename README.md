# TaskMild

<p align="center">
  <strong>Rust 原生 Android 内存管理器</strong><br>
  智能进程压制 · ZRAM/Swap 调优 · PSI 压力感知
</p>

<p align="center">
  <img src="https://img.shields.io/badge/version-v7.3.0-blue" alt="version">
  <img src="https://img.shields.io/badge/platform-Android-aosp" alt="platform">
  <img src="https://img.shields.io/badge/arch-aarch64-green" alt="arch">
  <img src="https://img.shields.io/badge/license-proprietary-red" alt="license">
</p>

---

## ✨ 特性

- 🦀 **Rust 原生** — 静态编译，零 GC 停顿，1.3MB 二进制，极低运行开销
- 🧠 **v73 八维评分** — OOM / RSS / PSS / CPU / IO / 冻结 / 年龄 / 僵尸 八维度综合评分
- 📡 **纯事件驱动** — cgroup inotify + logcat watcher + PSI epoll，零轮询
- 🔒 **安全加固** — Ed25519 签名验证、obfstr 编译期字符串混淆、sysctl 权限锁定
- 🔄 **热重载** — VM 参数 (swappiness / watermark / extra_free_kbytes) 即时生效
- 🏭 **OEM 适配** — 自动检测 OPPO (NandSwap/Athena/Osense)、小米 (8Gen1 LMK)、Qualcomm (Kona) 并应用针对性优化
- 🛡️ **健壮守护** — 指数退避重启、崩溃熔断、stop_marker 安全退出

## 📋 要求

| 项目 | 最低版本 |
|------|---------|
| Android | 8.0 (API 26) |
| 内核 | 4.4+ |
| Root | Magisk / KernelSU / APatch |
| 架构 | aarch64 (arm64-v8a) |

## 📦 安装

1. 下载最新版 `TaskMild-v7.x.x-xxx.zip`
2. 在 Magisk / KernelSU / APatch 管理器中「从本地安装」
3. 安装过程中通过**音量键**选择 Swap 配置：
   - **第一级**: 是否启用 Swap/ZRAM？（15秒超时，默认: 启用）
   - **第二级**: 仅 ZRAM 还是 ZRAM + Swap 文件？（15秒超时，默认: 仅 ZRAM）
   - **第三级**（升级时）: 替换还是保留现有配置？（15秒超时，默认: 保留）
4. **重启设备**生效

### 卸载

在 Magisk / KernelSU / APatch 管理器中移除模块，重启即可。模块会自动清理临时文件，保留用户配置和日志。

## 🔧 快速配置

配置文件: `/data/adb/taskmild/taskmild.conf`

### 推荐默认配置（开箱即用）

```ini
# === Swap / ZRAM ===
swap_enable=on          # 总开关
zram=true               # 重新配置 ZRAM
zram_size=auto          # 自动计算大小
comp_algorithm=auto     # 自动选择压缩算法
zram_writeback=false    # 不启用写回
swap=false              # 不创建 swap 文件
swappiness=auto         # 自动设置

# === 进程管理 ===
enable=1                # 启用进程管理
进程压制=on              # 启用压制
进程压制方式=1           # 始终压制（推荐）
进程压制时间=60          # 60秒检查间隔
信号强度=2              # 智能信号策略
基础阈值=60             # 动态阈值基准

# 白名单: 这些进程永不压制
白名单=com.tencent.mobileqq:MSF,com.tencent.mobileqq:peak,com.tencent.mm:push
```

### 场景速查

| 场景 | 关键修改 |
|------|---------|
| 🎮 游戏/性能优先 | `进程压制时间=30`, `基础阈值=40` |
| 🔋 保守/省电 | `进程压制方式=0`, `基础阈值=80`, `进程压制时间=120` |
| 💾 仅 ZRAM 不压制 | `enable=0`, `swap_enable=on`, `zram=true` |
| 📱 大内存 + Swap | `swap=true`, `swap_size=8192` |
| 🟢 OPPO 设备 | `zram_writeback=true` (NandSwap) |

> 📖 完整配置说明请参阅 [用户操作手册](./TaskMild-v7.3.0-用户操作手册.md)

## ⚙️ 配置项一览

### Swap / ZRAM

| 配置项 | 可选值 | 默认 | 说明 |
|--------|--------|------|------|
| `swap_enable` | on / off | on | Swap 总开关 |
| `zram` | true / false | true | 是否重配 ZRAM |
| `zram_size` | auto / 数值(MB) | auto | ZRAM 大小 |
| `comp_algorithm` | auto / lz4 / zstd / lzo-rle / lzo | auto | 压缩算法 |
| `zram_writeback` | true / false / default | false | 写回策略 |
| `swap` | true / false | false | 额外 swap 文件 |
| `swap_size` | 数值(MB) | 空 | swap 文件大小 |
| `swap_priority` | -1 / 正整数 | -1 | swap 优先级 |
| `swappiness` | auto / 0-200 | auto | 内核换出积极度 |
| `extra_free_kbytes` | 数值(KB) | 空 | 额外保留内存 |
| `watermark_scale_factor` | 数值 | 空 | 水印缩放因子 |

### 进程管理

| 配置项 | 可选值 | 默认 | 说明 |
|--------|--------|------|------|
| `enable` | 1 / 0 | 1 | 进程管理总开关 |
| `进程压制` | on / off | on | 压制开关 |
| `进程压制方式` | 0 / 1 | 1 | 0=仅灭屏 1=始终 |
| `进程压制时间` | 10-300(秒) | 60 | 检查间隔 |
| `白名单` | 逗号分隔 | (见上方) | 完全保护 |
| `黑名单` | 逗号分隔 | 空 | 强制评分 |
| `Grain压制` | 逗号分隔 | 空 | 后台即杀 |

### 信号与阈值

| 配置项 | 可选值 | 默认 | 说明 |
|--------|--------|------|------|
| `信号强度` | 1 / 2 / 3 | 2 | 1=全TERM 2=智能 3=全KILL |
| `基础阈值` | 数值 | 60 | 动态阈值基准 |
| `单轮最大压制` | 0 / 正整数 | 0 | 0=不限 |
| `后台宽限期` | 秒 | 15 | 前台切换保护 |
| `冷却期` | 秒 | 120 | 压制后冷却 |

### 高级

| 配置项 | 可选值 | 默认 | 说明 |
|--------|--------|------|------|
| `墓碑跳过选项` | 0 / 1 / 2 | 0 | 冻结进程处理 |
| `系统应用压制强度` | 0 / 1 / 2 | 0 | 0=完全保护 |
| `算法版本` | 72 / 73 | 73 | 评分算法版本 |
| `log_level` | debug / info / warn / error | info | 日志级别 |
| `log_max_size` | 字节数 | 1048576 | 日志最大 1MB |

## 🔄 热重载

修改配置后，进程管理参数和 VM 参数**即时生效**，无需重启：

```bash
# 修改配置
vi /data/adb/taskmild/taskmild.conf

# 或手动触发重载
kill -HUP $(cat /data/adb/taskmild/bin/taskmild.pid)
```

Swap 核心配置（ZRAM 大小/算法/写回等）变更需**重启设备**生效。

### 变更分类

| 配置项 | 热重载 | 需重启 |
|--------|:------:|:------:|
| swappiness / watermark / extra_free_kbytes | ✅ | |
| 白名单 / 黑名单 / Grain压制 | ✅ | |
| 信号强度 / 基础阈值 / 冷却期 | ✅ | |
| 进程压制时间 | ✅ | |
| swap_enable / zram / zram_size | | ✅ |
| comp_algorithm / zram_writeback | | ✅ |
| swap / swap_size / swap_priority | | ✅ |

## 📁 模块文件

安装后文件布局：

| 路径 | 说明 |
|------|------|
| `/data/adb/taskmild/bin/taskmild` | 守护进程二进制 |
| `/data/adb/taskmild/taskmild.conf` | 配置文件 |
| `/data/adb/taskmild/run.log` | 运行日志 |
| `/data/adb/taskmild/bin/taskmild.pid` | PID 文件 |
| `/data/adb/modules/taskmild/` | Magisk 模块目录 |
| `/data/swapfile` | Swap 文件（若启用） |

## 📄 日志

```bash
# 查看运行日志
tail -100 /data/adb/taskmild/run.log

# 实时监控
tail -f /data/adb/taskmild/run.log

# 查看启动诊断
grep "启动自检完成" /data/adb/taskmild/run.log
```

日志格式:
```
[2026-06-07 09:12:21.086] [信息] TaskMild v7.3.0 启动
[2026-06-07 09:12:21.150] [信息] 启动自检完成: 8/8 子系统正常
```

## 🛑 故障排除

| 问题 | 解决方案 |
|------|---------|
| 安装后不生效 | 确认已重启设备 |
| 消息推送延迟 | 将推送进程加入白名单 |
| 日志显示 "ZRAM 激活失败" | 检查内核是否支持 ZRAM，或设 `zram=false` |
| 守护进程频繁重启 | 检查 `run.log` 中的错误信息 |
| 想临时停止 | `echo "" > /data/adb/taskmild/stop_marker` |
| 想完全卸载 | Magisk 管理器移除模块 → 重启 |

## 📊 性能

| 指标 | 数值 |
|------|------|
| 二进制大小 | ~1.3 MB (stripped, static) |
| 运行内存 | < 5 MB RSS |
| CPU 开销 | 空闲时 < 0.1% |
| 评分延迟 | < 50ms (单次全进程扫描) |
| 前台检测 | 事件驱动，亚秒级响应 |

## 📝 版本历史

### v7.3.0 (730)

- ✨ 八维评分算法 (v73)，替代 v72 三维评分
- ✨ PSI 内存压力感知 (epoll 触发器 + 轮询降级)
- ✨ 主进程四级保护机制
- ✨ 冷却期指数递增 + ±25% 随机抖动
- ✨ 前台退出时间追踪 (年龄评分)
- ✨ 配置热重载 (VM 参数即时生效)
- ✨ 安装时 Swap 交互选择
- 🔧 移除所有 dumpsys 依赖，纯事件驱动
- 🔧 Ed25519 签名验证 + 格式自动检测
- 🔧 obfstr 编译期字符串混淆
- 🔧 MountGuard RAII 绑定挂载清理
- 🔧 持久化重试计数器 + stop_marker 安全退出

## 🙏 致谢

- **嘟嘟Ski** — [Scene](https://www.coolapk.com/apk/com.omarea.vtools) 附加模块(二) v4.2.2R：TaskMild 的 Swap/ZRAM 内核调优逻辑（ZRAM 自动计算、压缩算法检测、NandSwap/SmartSwap writeback、swappiness/watermark/extra_free_kbytes 自动配置）基于此模块参考实现

## ⚖️ 许可证

本作品为闭源专有软件。未经授权不得复制、修改或分发。

---

<p align="center">
  <sub>TaskMild v7.3.0 — 让每一兆内存都被善待</sub>
</p>
