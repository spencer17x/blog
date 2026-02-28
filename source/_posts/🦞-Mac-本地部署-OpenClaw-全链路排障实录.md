---
title: "\U0001F99E Mac 本地部署 OpenClaw 全链路排障实录"
date: 2026-02-28 15:00:16
tags:
  - AI
---

## 从 `fetch failed` 到 Telegram 正常响应

> 环境：Mac + LaunchAgent + Clash + ChatGPT OAuth + Telegram Bot
>
> 现象：Web UI `fetch failed`、Telegram 无响应、日志不断刷网络失败

------

## 一、问题现象

本地部署 OpenClaw 后出现以下问题：

### 1️⃣ Web UI 无法聊天

访问：

```
<http://127.0.0.1:18789/chat>
```

发送消息后直接报：

```
fetch failed
```

------

### 2️⃣ Telegram Bot 不响应

`gateway.err.log` 中不断刷：

```
[telegram] setMyCommands failed: Network request failed
[telegram] deleteWebhook failed: Network request failed
[telegram] deleteMyCommands failed: Network request failed
```

------

### 3️⃣ Agent 执行失败

日志中还出现：

```
[agent/embedded] embedded run agent end ... error=fetch failed
```

------

## 二、第一步：确认服务本身是否正常

先排除“服务没启动”的情况。

### 端口监听检查

```bash
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

确认 Node 进程在监听：

```
TCP 127.0.0.1:18789 (LISTEN)
```

------

### Health 检查

```bash
curl <http://127.0.0.1:18789/health>
```

返回正常 JSON。

说明：

- Gateway 在跑
- Web UI fetch failed 不是因为服务没起来

------

## 三、问题真正的根因

问题出在 LaunchAgent 环境变量：

```xml
all_proxy = socks5://127.0.0.1:7890
```

但 Clash 的：

- 7890 是 **HTTP 代理端口**
- 不是 SOCKS5

很多 Node 库（包括 Telegram SDK）会优先读取：

```
ALL_PROXY / all_proxy
```

于是：

- 用 SOCKS 协议连接 HTTP 端口
- 所有 Telegram 请求失败
- Agent 内部 fetch 也失败
- 前端只显示 `fetch failed`

这是一个非常隐蔽但致命的代理配置错误。

------

## 四、关键修复步骤

### 删除 all_proxy

```bash
/usr/libexec/PlistBuddy -c "Delete :EnvironmentVariables:all_proxy" ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

如果存在 ALL_PROXY 也删除：

```bash
/usr/libexec/PlistBuddy -c "Delete :EnvironmentVariables:ALL_PROXY" ~/Library/LaunchAgents/ai.openclaw.gateway.plist
```

------

### 重载 LaunchAgent

```bash
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.gateway.plist
launchctl kickstart -k gui/$(id -u)/ai.openclaw.gateway
```

------

### 验证删除成功

```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway | grep -i all_proxy || echo "all_proxy removed"
```

输出：

```
all_proxy removed
```

此时：

- Telegram 网络错误立即停止
- Agent fetch 也恢复正常
- Web UI 可以正常聊天

------

## 五、Clash + Node 的正确代理配置

推荐只保留：

```bash
HTTP_PROXY=http://127.0.0.1:7890
HTTPS_PROXY=http://127.0.0.1:7890
NO_PROXY=localhost,127.0.0.1,::1
```

不要随便添加：

```
ALL_PROXY
all_proxy
```

除非你：

- 确认 Clash 的 SOCKS 端口
- 且确实需要 SOCKS 代理

------

## 六、Mac LaunchAgent 的一个常见坑

很多人会这样做：

```bash
export HTTP_PROXY=...
openclaw gateway start
```

但 LaunchAgent 启动的进程 **不会继承当前 shell 的 export**。

必须写入：

```xml
<key>EnvironmentVariables</key>
```

或者使用：

```bash
launchctl setenv
```

否则环境变量不会真正传递给 Gateway 进程。

------

## 七、Telegram 不响应的第二层原因：Pairing

网络修复后，Telegram 发送：

```
OpenClaw: access not configured.
Pairing code: ECALJNXG
```

这不是网络问题，而是 OpenClaw 的安全机制。

解决：

```bash
openclaw pairing approve telegram ECALJNXG
```

然后即可正常对话。

------

## 八、完整排障流程总结

### 1️⃣ 确认端口监听

```bash
lsof -nP -iTCP:18789 -sTCP:LISTEN
```

------

### 2️⃣ 检查 health

```bash
curl <http://127.0.0.1:18789/health>
```

------

### 3️⃣ 查看 LaunchAgent 环境

```bash
launchctl print gui/$(id -u)/ai.openclaw.gateway
```

------

### 4️⃣ 实时查看日志

```bash
openclaw logs --follow
```

或：

```bash
tail -f ~/.openclaw/logs/gateway.err.log
```

------

## 九、这次排障的核心经验

### ✅ `all_proxy` 是高危变量

很多库优先读取它。

### ✅ 不要用 SOCKS 协议连接 HTTP 端口

### ✅ LaunchAgent 不继承 shell 环境变量

### ✅ `fetch failed` 不一定是网络问题

可能是 401、代理协议不匹配或 SDK 内部异常。

### ✅ Telegram 不回消息不一定是网络问题

可能只是未配对（pairing）。

------

## 十、最终结论

> 在 Mac 上使用 Clash 运行 Node 服务时，
>
> **只使用 HTTP_PROXY / HTTPS_PROXY 即可。**
>
> 不要随意设置 ALL_PROXY，尤其不要把 SOCKS 协议指向 HTTP 端口。

问题本质不是 OpenClaw Bug，而是代理环境变量组合导致的协议错误。

------

排障完成。
