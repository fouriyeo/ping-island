---
title: "过滤 qoderwake 后台误唤起"
date: 2026-07-30
lifecycle: active
summary: "qoderwake 常驻 worker 自动触发的 Qoder 会话与人工会话共用 ~/.qoder/settings.json 的 hooks，Qoder 侧无法区分，导致桌面宠物被后台任务频繁误唤起。改为在 bridge 启动脚本入口用 QODERWAKE_HOME 环境变量短路退出，并同步改 HookInstaller.swift 脚本模板以防 app 启动覆盖。"
---

# 过滤 qoderwake 后台误唤起

## 目标

让 qoderwake 后台自动触发的 Qoder 会话不再走 PingIsland bridge，从而不再误唤起桌面宠物；同时保证人工使用的 Qoder CLI 不受影响，且修复不被 app 启动覆盖。

## 触发原因

用户观察到桌面宠物在无人操作时被频繁唤起。排查发现 qoderwake（独立常驻 agent worker 产品，`~/.qoderwake/`）通过 `qodercli-wake` 启动 Qoder CLI 进程处理钉钉/远程会话等后台任务，这些进程读取与人工会话相同的 `~/.qoder/settings.json` hooks，因此每次后台任务都触发 bridge → socket → 宠物动画。

## 关键决策

1. **用 `QODERWAKE_HOME` 环境变量作为过滤信号** (confirmed) — `ps eww` 验证 qodercli-wake 进程环境带 `QODERWAKE_HOME=/Users/.../.qoderwake` 与 `QODER_SESSION_TYPE=qoder_wake`，其 fork 出的 hook 子进程会继承该变量；人工会话不带，故可精确区分。
2. **在 bridge 启动脚本入口短路退出** (confirmed) — `~/.ping-island/bin/ping-island-bridge` 实为 zsh wrapper（非编译二进制），首行加 `[ -n "$QODERWAKE_HOME" ] && exit 0`，命中即静默返回成功，Qoder CLI 视为 hook 正常完成不报错，事件也不进 PingIsland。
3. **同步改 `HookInstaller.swift` 的脚本生成模板** (inferred?) — 用户重启后亲眼见证手动改的 wrapper 被 app 启动时的 HookInstaller 重写覆盖；为抗覆盖，主动把同一行加进 `installBridgeLauncherIfNeeded` 的脚本模板，使后续 app 重写也自带过滤。用户当时只说"先不验证 build"，未显式授权改源码，此持久化手段为 AI 主动决定。

## 取舍说明

- **没选改 `~/.qoder/settings.json` 的 hook command**：app 启动时 `HookInstaller` 会重写该文件，手动加的环境判断会被覆盖（已实测）。
- **没选重编 bridge Swift 二进制**：用户询问 wrapper 能否改，确认是 shell 脚本后选择改 wrapper，等于隐式放弃重编 `Prototype/Sources/IslandBridge` 的链路；wrapper 路改动最小、即改即生效。
- **没选在 PingIsland 侧静默标记 background session**：那需要 bridge 仍把事件送进 app 再过滤，链路更长且宠物仍可能闪一下；在入口直接短路最干净。

## 受影响文件

- `~/.ping-island/bin/ping-island-bridge`（用户机器上的 wrapper 脚本，非仓库文件）：首行加 qoderwake 短路，立即生效。
- `PingIsland/Services/Hooks/HookInstaller.swift`：`installBridgeLauncherIfNeeded` 的脚本模板加同一行，防止 app 启动覆盖手动修复。

## 已知边界 / 坑

- wrapper 脚本被覆盖的根因是 `HookInstaller.swift:1383` 每次启动用模板重写并比对内容；只改 wrapper 不改模板必然被覆盖，两处必须一起改。
- 过滤依赖 qoderwake 始终注入 `QODERWAKE_HOME`；若未来 qoderwake 改环境变量名或人工会话误带该变量，过滤会失效或误伤，需重新核对（用 `ps eww -p <pid>` 复查）。
- 验证字符串是否编进 app 时，Debug 模式主二进制 `Ping Island` 只是 59KB launcher，全部 Swift 代码在 `Ping Island.debug.dylib`；`strings`/`grep` 必须扫 dylib，否则误判"改动没生效"。

## 关联迭代

- 关联 `2026-07-30-重定义宠物动画状态语义.md` — 同次迭代，分别从"入口过滤"和"状态映射"两端治理宠物误唤起/误显示。
