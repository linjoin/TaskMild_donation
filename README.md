
# TaskMild 捐赠版操作文档

## 1. 简介

TaskMild 捐赠版是基于公益版继续增强的版本。

捐赠版当前定位：

- 继续沿用公益版的安装路径、工作目录和配置路径
- 主程序使用 Rust 实现
- 增加授权验证
- 在保留前台、主进程、系统应用、白名单保护的前提下，提供更明显的后台处理效果
- 配置比公益版更灵活，但仍尽量保持简洁

捐赠版与公益版的关系：

- 两者**不允许并存**
- 捐赠版安装后会直接沿用原有工作目录和配置
- 用户从公益版切换到捐赠版时，不需要重新设置路径和基础文件

---

## 2. 目录结构

### 模块目录

```text
/data/adb/modules/taskmild/
  module.prop
  service.sh
  action.sh
  uninstall.sh
```

### 工作目录

```text
/data/adb/taskmild/
  taskmild.conf
  run.log
  auth/
    Unique_identifier
    Unique_identifier.sig
  bin/
    taskmild
    taskmild.pid
```

---

## 3. 重要文件说明

### 主程序

```text
/data/adb/taskmild/bin/taskmild
```

### 配置文件

```text
/data/adb/taskmild/taskmild.conf
```

### 日志文件

```text
/data/adb/taskmild/run.log
```

### 授权缓存

```text
/data/adb/taskmild/auth/Unique_identifier
/data/adb/taskmild/auth/Unique_identifier.sig
```

### 模块信息

```text
/data/adb/modules/taskmild/module.prop
```

---

## 4. 授权机制说明

捐赠版带授权验证，公益版不带。

### 验证触发时机

当前仅在以下命令前做授权验证：

- `run`
- `once`

以下命令不做授权验证：

- `status`
- `stop`
- `disable`
- `log`

### 验证逻辑

1. 读取设备标识：
   - `getprop ro.serialno`
2. 优先读取本地授权缓存：
   - `Unique_identifier`
   - `Unique_identifier.sig`
3. 用程序内置公钥对本地缓存做签名校验
4. 校验当前设备标识是否在授权列表中
5. 本地缓存无效或不存在时，再联网从 Gitee 下载新的授权列表和签名
6. 下载成功并校验通过后，覆盖本地授权缓存
7. 验证失败时拒绝启动捐赠版主功能

### 授权失败时

会更新：

```text
/data/adb/modules/taskmild/module.prop
```

中的 `description` 字段，显示例如：

- `未授权设备`
- `联网验证失败`
- `授权签名无效`
- `设备标识获取失败`

### 授权文件格式

#### `Unique_identifier`
纯文本，一行一个设备 ID，例如：

```text
ABC123456789
DEF987654321
1234567890ABCDEF
```

#### `Unique_identifier.sig`
对 `Unique_identifier` 原始内容做签名后的 Base64 文本。

---

## 5. 配置文件完整示例

```ini
# 模块总开关
enable=1

# 进程压制总开关
# on=开启
# off=关闭
进程压制=on

# 进程压制方式
# 0=只在熄屏且后台时处理
# 1=只要后台就处理
进程压制方式=1

# 进程压制周期（秒）
进程压制时间=15

# 深度压制
# 0=关闭
# 1=模式1，较激进
# 2=模式2，更激进
深度压制=2

# 白名单
# 支持包名和完整进程名
# 多个项目使用英文逗号分隔
白名单=com.tencent.mobileqq:MSF,com.tencent.mm:push,com.tencent.mobileqq,com.tencent.mm,com.tencent.tim,com.tencent.tim:MSF

# Grain压制
# 建议只填写包名
# 多个项目使用英文逗号分隔
Grain压制=

# 日志级别
# debug=调试
# info=信息
# warn=警告
# error=错误
log_level=info
```

---

## 6. 配置项说明

### enable

模块总开关。

- `1` 为开启
- `0` 为关闭

---

### 进程压制

进程压制功能总开关。

- `on` 为启用
- `off` 为关闭

当关闭时，捐赠版主程序仍可运行，但不执行进程压制逻辑。

---

### 进程压制方式

控制在哪些场景下对后台进程进行处理。

#### `0`
- 只在**熄屏且应用处于后台**时处理

#### `1`
- 只要应用处于后台就处理
- 不区分当前是否熄屏
- 推荐使用这个模式

---

### 进程压制时间

表示处理周期，单位为秒。

例如：

```ini
进程压制时间=15
```

表示每隔 15 秒做一次处理判断。

---

### 深度压制

控制处理力度。

#### `0`
- 关闭深度压制
- 只走普通逻辑

#### `1`
- 启用模式 1
- 比普通逻辑更激进

#### `2`
- 启用模式 2
- 比模式 1 更早、更强地处理后台目标

---

### 白名单

白名单优先级最高。

支持两种写法：

- 包名
- 完整进程名

例如：

```ini
白名单=com.tencent.mobileqq:MSF,com.tencent.mm
```

含义：

- `com.tencent.mobileqq:MSF` 只保护这个进程
- `com.tencent.mm` 保护整个微信包

### 注意
白名单命中后，无论普通压制、深度压制、Grain压制，都不会处理。

---

### Grain压制

Grain压制是捐赠版的重要增强项。

作用是：

- 对指定包的后台子进程使用更积极的处理逻辑
- 用于你明确想重点处理的目标应用

建议：

- 只填**包名**
- 不建议填带 `:` 的子进程名

例如：

```ini
Grain压制=com.ss.android.ugc.aweme,com.xunmeng.pinduoduo
```

---

### log_level

日志级别。

可选值：

- `debug`
- `info`
- `warn`
- `error`

建议：

- 日常使用：`warn`
- 想看实际处理效果：`info`
- 想排查逻辑细节：`debug`

---

## 7. 当前保护规则

不管普通压制还是深度压制，以下对象都不会处理：

- 白名单
- 当前前台应用
- 主进程
- 系统应用
- `top-app / foreground` 关键运行态进程

### 主进程规则

当前只认：

- `进程名 == 包名`

例如：

- `com.tencent.mm` 是主进程
- `com.tencent.mm:push` 不是主进程
- `com.tencent.mm:main` 也不默认视为主进程

---

## 8. 推荐配置

### 省电优先版

```ini
enable=1
进程压制=on
进程压制方式=0
进程压制时间=20
深度压制=1
白名单=com.tencent.mobileqq:MSF,com.tencent.mm:push,com.tencent.mobileqq,com.tencent.mm,com.tencent.tim,com.tencent.tim:MSF,com.xiaomi.xmsf,com.android.vending,com.google.android.webview
Grain压制=
log_level=warn
```

适合：

- 更省电
- 更稳
- 对系统和常用应用更保守

---

### 效果优先版

```ini
enable=1
进程压制=on
进程压制方式=1
进程压制时间=15
深度压制=2
白名单=com.tencent.mobileqq:MSF,com.tencent.mm:push,com.tencent.mobileqq,com.tencent.mm
Grain压制=com.ss.android.ugc.aweme,com.xunmeng.pinduoduo
log_level=info
```

适合：

- 想明显看到处理效果
- 更积极地处理后台子进程
- 但仍保留前台、主进程、系统应用和白名单保护

---

