# Ping Island 决策时间线

> 最后更新: 2026-07-30 · 共 2 条记录
> 自动生成，禁止手动编辑。如需修正，修改源条目后重新生成。

### 2026-07

- **[07-30] 过滤 qoderwake 后台误唤起** — qoderwake 常驻 worker 自动触发的 Qoder 会话与人工会话共用 `~/.qoder/settings.json` 的 hooks，Qoder 侧无法区分，导致桌面宠物被后台任务频繁误唤起。改为在 bridge 启动脚本入口用 `QODERWAKE_HOME` 环境变量短路退出，并同步改 `HookInstaller.swift` 脚本模板以防 app 启动覆盖。[详情](2026-07-30-过滤-qoderwake-后台误唤起.md)
- **[07-30] 重定义宠物动画状态语义** — AI 已给完最终结果（`waitingForInput`）时闭合 notch 宠物仍显示 running，根因是 `closedNotchStatus` 把除 `ended` 外全映射成 working。确立"宠物动画反映算力消耗而非会话存活"的原则，working 仅留给 `processing/compacting`，行级 warning 也只在真正需要干预时触发。[详情](2026-07-30-重定义宠物动画状态语义.md)
