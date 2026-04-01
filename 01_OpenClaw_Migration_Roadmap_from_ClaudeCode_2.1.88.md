# OpenClaw 迁移完整路线图: 从 Claude Code 2.1.88 到 OpenClaw 的分阶段移植指南

分析师: 哈基米 (Agent)  
执行人: junshunG  
日期: 2026-04-01  
基于: Claude Code 2.1.88 源码深度分析 (8 份报告，~123KB)  
目标: OpenClaw 现有能力增强 + 缺失模块填补

---

## 1. 执行摘要

### 1.1 迁移目标

将 Claude Code 2.1.88 的 **核心架构与工程实践** 移植到 OpenClaw，包括：

- **查询引擎** (Query Engine) 与工具循环
- **权限系统** (8 步管道 + YOLO Classifier)
- **压缩机制** (Auto Compact + System Prompt Sections)
- **多代理协调** (AgentTool + Coordinator + Task)
- **协议集成** (MCP + Skills 插件系统)
- **基础设施** (State 管理、Memdir、Ink 渲染、Keybindings)

### 1.2 迁移原则

| 原则 | 说明 |
|------|------|
| **增量迁移** | 按 P0→P1→P2→P3 分阶段，每阶段可独立运行 |
| **能力对齐** | OpenClaw 现有技能（如 telescope、git）保持兼容 |
| **性能优先** | 优先移植影响响应速度的核心路径 |
| **安全底线** | 权限管道、Sandbox、Path Validation 必须保留 |
| **可观测性** | Analytics + Cost Tracking + Error Classification 一同迁移 |

### 1.3 迁移总览（4 阶段）

```
Phase 0 (基础框架)    → State + Query Core + Permissions
Phase 1 (代理能力)    → AgentTool + Compaction + MCP
Phase 2 (高级体验)    → System Prompt + Buddy + Dream + Coordinator
Phase 3 ( polish )    → OutputStyles + Voice + Native-TS + MoreRight
```

---

## 2. Phase 0: 基础框架 (Week 1-2) — 🔥 P0 必须

### 2.1 目标

建立可运行的查询循环，具备基本的工具执行和安全检查。

### 2.2 任务清单

| 任务 | Claude Code 源码 | OpenClaw 对应 | 工作量 | 依赖 |
|------|----------------|---------------|--------|------|
| **State 管理** | `state/AppState.tsx`, `store.ts` | `OpenClaw 已有 AppState` | 适配 | - |
| **Query Config Snapshot** | `query/config.ts` | 新增 `QueryConfig` 结构 | 2h | State |
| **工具池装配** | `tools.ts` `getAllBaseTools()` | 已有 `toolRegistry` | 重构 | - |
| **权限管道 8 步** | `utils/permissions/permissions.ts` `hasPermissionsToUseToolInner()` | 需新建 `permissions/` 模块 | 2d | State, Tools |
| **Sandbox** | `tools/BashTool/shouldUseSandbox.ts` | 需实现 `bwrap` 调用 | 1d | Permissions |
| **Path Validation** | `utils/pathValidation/validatePath.ts` | 需新建 `pathValidation/` | 1d | - |
| **基础 Error Handling** | `utils/errors.ts`, `classifyAxiosError` | 已有错误类，需补充分类 | 4h | - |
| **PTL 重试** | `query.ts` `truncateHeadForPTLRetry` | 需集成到 Query | 1d | Query Core, Compaction |
| **Query Engine 主循环** | `query.ts` `query()` | 需重写或大幅重构 | 3d | Config, Tools, Permissions |
| **Stop Hooks** | `query/stopHooks.ts` | 新增钩子机制 | 2h | Query |
| **LogEvent + AnalyticsSink** | `services/analytics/index.ts` | 已有 `logger`，需扩展事件类型 | 1d | - |
| **GrowthBook 初始化** | `services/analytics/growthbook.ts` | 需集成或降级为 env flags | 1d | - |

**Phase 0 总工作量**: ~10-12 天 (单人)

### 2.3 里程碑

**M1**: State + Tools + Permissions 8 步管道可通过所有工具安全检查
- 运行 `/permissions` 测试
- 所有工具调用都经过 `hasPermissionsToUseTool()`
- 模式匹配、Ask 行为、bypass 规则生效

**M2**: Query 主循环可以处理单轮对话 + 工具调用
- 用户提问 → 模型调用 → 工具执行 → 结果返回
- PTL 触发时自动截断重试（最多 3 次）

**M3**: 完整 Phase 0 可用
- 支持 Bash、FileRead、FileEdit 等核心工具
- Sandbox 正常工作（bwrap 隔离）
- 错误分类 + withRetry 网络重试
- Analytics 事件记录（本地文件或 Datadog）

---

## 3. Phase 1: 代理能力 (Week 3-4) — 🔥 P0/P1

### 3.1 目标

实现多代理协作、自动压缩、MCP 协议集成。

### 3.2 任务清单

| 任务 | Claude Code 源码 | OpenClaw 对应 | 工作量 | 依赖 |
|------|----------------|---------------|--------|------|
| **AgentTool 基础** | `tools/AgentTool/AgentTool.tsx` | 需新增 | 2d | Phase 0 |
| **Fork Subagent** | `tools/AgentTool/forkSubagent.ts` | 需实现 fork 逻辑 | 2d | AgentTool |
| **Agent 颜色系统** | `tools/AgentTool/AgentTool.tsx` `setAgentColor` | 新增配置 | 4h | - |
| **Compaction 策略** | `services/compact/compact.ts` `compactConversation()` | 需实现 | 2d | Query, State |
| **Auto Compact Micro** | `services/compact/autoCompact.ts` `maybeCompact()` | 需实现 | 1d | Compaction |
| **System Prompt Sections** | `constants/systemPromptSections.ts` | 需新建 | 1d | Query |
| **Cache Sharing** | `forkSubagent.ts` `forkContextMessages` | 优化项，可延后 | 1d | Fork Subagent |
| **Task 系统** | `utils/task/framework.ts`, `tasks/*` | 需新建 | 2d | State |
| **Task 面板 UI** | `screens/TasksScreen.tsx` | 可选 UI，Phase 2 | - | Task |
| **MCP Connection Manager** | `services/mcp/useManageMCPConnections.ts` | 需实现 MCP client | 3d | - |
| **MCP Tool 转换** | `services/mcp/client.ts` `convertToolToAnthropic` | 需实现 | 2d | MCP Connection |
| **MCP 权限规则** | `utils/permissions/permissions.ts` `mcp__` 前缀 | 扩展 Permissions | 1d | Permissions, MCP |
| **MCP 大输出截断** | `utils/mcpValidation.ts` | 需实现 | 1d | MCP Tool |
| **MCP OAuth** | `services/mcp/client.ts` OAuth flow | 可选，Phase 3 | 2d | MCP |
| **Skills 加载器** | `skills/loadSkillsDir.ts` | 需实现 | 2d | - |
| **SkillTool 执行** | `tools/SkillTool/SkillTool.ts` | 需实现 | 1d | Skills |

**Phase 1 总工作量**: ~15-18 天

### 3.3 里程碑

**M4**: AgentTool 可以 spawn 独立 worker 并返回结果
- 使用 `forkSubagent()` 创建隔离进程
- Worker 工具集限制在 `ASYNC_AGENT_ALLOWED_TOOLS`
- Agent 颜色系统和 pills 显示

**M5**: Compaction 自动触发，Token 窗口保持健康
- Micro compact 每 15 轮检查
- Full compact 每天一次（或手动 `/compact`）
- System prompt sections cache 正常工作

**M6**: MCP 服务器连接成功，工具聚合到工具池
- `mcp__<server>` 权限规则生效
- 大输出自动截断（>10K tokens）
- MCP tools 可以像普通工具一样调用

**M7**: 完整 Phase 1 可用
- 多代理并行工作（Coordinator 模式可启用）
- Task 状态机运行
- Skills 可从 `~/.claude/skills/` 加载

---

## 4. Phase 2: 高级体验 (Week 5-6) — 🟡 P1/P2

### 4.1 目标

实现特殊模式、缓存优化、基础设施完善。

### 4.2 任务清单

| 任务 | Claude Code 源码 | OpenClaw 对应 | 工作量 | 依赖 |
|------|----------------|---------------|--------|------|
| **KAIROS 总控** | `services/autoDream/autoDream.ts` | 需实现常驻进程 | 3d | Phase 1 |
| **KAIROS 15s 预算** | `assistant/backgroundAgentManager.ts` | 需实现超时控制 | 2d | KAIROS |
| **KAIROS Brief Mode** | `brief/briefMode.ts` | 需实现输出截断 | 1d | KAIROS |
| **Buddy 渲染** | `buddy/CompanionSprite.tsx`, `sprites.ts` | 需 Ink/TUI 实现 | 3d | Ink (Phase 0) |
| **Buddy 气泡系统** | `CompanionSprite.tsx` bubble logic | 需实现 | 2d | Buddy |
| **Buddy Gacha** | `buddy/companion.ts` | 需实现 | 1d | Buddy |
| **Dream 四阶段** | `tasks/DreamTask/DreamTask.ts` | 需集成到 Task 系统 | 3d | Task, KAIROS |
| **Dream 锁机制** | `services/autoDream/consolidationLock.ts` | 需实现 | 1d | Dream |
| **Coordinator 模式** | `coordinator/coordinatorMode.ts` | 需实现多代理协调 | 3d | AgentTool, Task |
| **Scratchpad** | `coordinatorMode.ts` `getCoordinatorUserContext` | 需实现共享目录 | 1d | Coordinator |
| **Cache Sharing 实验** | `forkSubagent.ts` `tengu_compact_cache_prefix` | 优化项 | 1d | Fork Subagent |
| **Memdir 索引** | `memdir/memdir.ts`, `memoryScan.ts` | 需实现 | 2d | - |
| **Memdir 类型** | `memdir/memoryTypes.ts` | 需实现 | 4h | - |
| **Entrypoint 截断** | `memdir.ts` `truncateEntrypointContent` | 需实现 | 4h | Memdir |
| **Server Direct Connect** | `server/createDirectConnectSession.ts` | 可选，CCR 用 | 2d | - |
| **REPL 恢复** | `screens/ResumeConversation.tsx` | 需实现 | 2d | State, Memdir |
| **Ink TUI 渲染** | `ink/Box.tsx`, `Text.tsx` | 已有 `ink` 库，需集成 | 2d | - |
| **Ansi 颜色** | `ink/constants.ts` | 需实现颜色映射 | 1d | - |
| **Keybindings 系统** | `keybindings/` | 需实现 | 2d | - |
| **默认绑定** | `keybindings/defaultBindings.ts` | 需实现 | 1d | Keybindings |
| **Binding 匹配** | `keybindings/match.ts` | 需实现 | 1d | Keybindings |

**Phase 2 总工作量**: ~25-30 天

### 4.3 里程碑

**M8**: KAIROS 常驻进程可以执行后台任务（15s 预算）
- 通过 `/dream` 手动触发
- Brief mode 输出截断

**M9**: Buddy 渲染在 TUI 中显示，支持动画和气泡
- 物种、稀有度、颜色系统
- 气泡内容随 `companion.personality` 变化

**M10**: Dream 自动触发，memory index 更新
- 锁文件机制防并发
- `filesTouched` 追踪

**M11**: Coordinator 模式可用，Worker 并行执行
- `<task-notification>` XML 协议
- `continue` vs `spawn` 决策
- Scratchpad 共享目录

**M12**: Memdir 索引可以扫描并格式化清单
- MEMORY.md 200 行/25KB 限制
- 类型标记和 frontmatter 解析

**M13**: Keybindings 系统工作，默认快捷键生效
- 平台适配（Windows alt+v vs ctrl+v）
- VT mode 检测
- chord 支持

---

## 5. Phase 3: Polish 与优化 (Week 7-8) — 🟢 P2/P3

### 5.1 目标

完善用户体验、性能优化、高级特性。

### 5.2 任务清单

| 任务 | Claude Code 源码 | OpenClaw 对应 | 工作量 | 依赖 |
|------|----------------|---------------|--------|------|
| **OutputStyles** | `outputStyles/loadOutputStyles.ts` | 需实现 | 2d | Memdir |
| **Voice 模式** | `voice/voiceModeEnabled.ts` | 需实现 OAuth + WebSocket | 5d | Auth, WebSocket |
| **UpstreamProxy** | `upstreamproxy/upstreamproxy.ts` | CCR 专用，可选 | 3d | Remote |
| **Context (Mailbox)** | `context/mailbox.tsx` | 需实现异步队列 | 2d | State |
| **Notifications** | `context/notifications.tsx` | 需实现 Toast | 1d | Context |
| **Token Budget 二分** | `query/tokenBudget.ts` | 需优化输入/输出分配 | 1d | Query |
| **Query Config Gating** | `query/config.ts` | 已实现，需完善 | 1d | - |
| **Migrations 框架** | `migrations/` | 需实现 | 2d | Settings |
| **MoreRight** | `moreright/useMoreRight.tsx` | Vim 增强，可选 | 2d | Vim |
| **Vim 完整集成** | `vim/` 全套 | 高级编辑模式 | 5d | Keybindings, Cursor |
| **Native-TS ColorDiff** | `native-ts/color-diff/` | 可访问性，可选 | 2d | - |
| **Native-TS FileIndex** | `native-ts/file-index/` | Grep 加速，可选 | 3d | - |
| **Native-TS Yoga** | `native-ts/yoga-layout/` | 复杂 UI，可选 | 3d | Ink |
| **Remote/CCR** | `remote/`, `upstreamproxy/` | 远程卸载，可选 | 5d | UpstreamProxy |
| **Bridge (claude.ai)** | `bridge/` | 桥接网页版，可选 | 5d | Remote |
| **Transcription** | `voice/transcription.ts` | 语音转文本，需 API | 3d | Voice |
| **Hook 执行引擎** | `utils/hooks.ts` | 需完善 | 2d | Permissions |
| **Plugin Hot-Reload** | `plugins/builtinPlugins.ts` | 需实现 fs.watch | 2d | Plugins |

**Phase 3 总工作量**: ~30-40 天 (可选任务多)

### 5.3 里程碑

**M14**: OutputStyles 可从 `.claude/output-styles/` 加载并注入系统提示
- Markdown frontmatter 解析
- `keep-coding-instructions` 标志支持

**M15**: Voice 模式可用（如果 OAuth 和 API 就绪）
- 按住说话 → 转文本 → 用户消息
- Claude 语音回复流式播放

**M16**: 完整 Error Classification + withRetry 网络恢复
- 5 分类（auth/timeout/network/http/other）
- 指数退避 + fallback model

**M17**: Migrations 框架可以平滑升级配置
- `applied.json` 跟踪
- 幂等迁移函数

---

## 6. 优先级矩阵 (P0-P3) 速查

| 模块 | 优先级 | Phase | 工作量 | 核心价值 |
|------|--------|-------|--------|----------|
| State 管理 | 🔥 P0 | 0 | 2d | 全局状态单一事实源 |
| Query 主循环 | 🔥 P0 | 0 | 3d | 请求生命周期 |
| Permissions 8 步 | 🔥 P0 | 0 | 2d | 安全底线 |
| Error Handling | 🔥 P0 | 0 | 1d | 生产必需要 |
| PTL 重试 | 🔥 P0 | 0 | 1d | 长上下文支持 |
| AgentTool | 🔥 P0 | 1 | 2d | 多代理基础 |
| Fork Subagent | 🔥 P0 | 1 | 2d | 隔离 + cache sharing |
| Compaction | 🔥 P0 | 1 | 2d | 性能优化核心 |
| System Prompt Sections | 🟡 P2 | 1 | 1d | Token 节省 |
| MCP Connection | 🔥 P1 | 1 | 3d | 协议集成 |
| MCP Tool 转换 | 🔥 P1 | 1 | 2d | 工具聚合 |
| MCP 权限规则 | 🔥 P1 | 1 | 1d | 安全 |
| Skills Loader | 🟡 P2 | 1 | 2d | 插件生态 |
| Task 系统 | 🔥 P1 | 1 | 2d | 任务生命周期 |
| Coordinator | 🟡 P2 | 2 | 3d | CEO 模式 |
| KAIROS | 🟢 P3 | 2 | 3d | 常驻助理（复杂） |
| Buddy | 🟡 P2 | 2 | 3d | 趣味 + 可访问性 |
| Dream | 🟡 P2 | 2 | 3d | 自动记忆整合 |
| Memdir | 🔥 P1 | 2 | 2d | 长期记忆基础 |
| Keybindings | 🔥 P1 | 2 | 2d | TUI 必备 |
| Ink 渲染 | 🔥 P1 | 2 | 2d | TUI 基础 |
| OutputStyles | 🟢 P3 | 3 | 2d | 输出定制 |
| Voice | 🟢 P3 | 3 | 5d | 高级交互 |
| Vim 模式 | 🟢 P4 | 3 | 5d | 小众需求 |
| Native-TS | 🟢 P4 | 3 | 3d | 性能加速可选 |

---

## 7. 技术依赖图 (关键路径)

```
Phase 0: Foundation
State → Query Config → Tools → Permissions (8 steps) → Query Engine → Error Handling
         ↓
      PTL Retry → Stop Hooks → Analytics

Phase 1: Agent Capabilities
AgentTool → Fork Subagent (cache prefix) → Compaction → System Prompt Sections
         ↓
      Task System → MCP Connection → MCP Tool → Permissions (mcp__)
         ↓
      Skills Loader → SkillTool

Phase 2: Advanced Experience
KAIROS (daemon) → Background Agent Manager → DreamTask → Memdir
         ↓
      Coordinator (Scratchpad) → Task UI
         ↓
      Ink + Keybindings → Buddy (sprites, animation) → Voice (future)

Phase 3: Polish
OutputStyles (frontmatter) → Migrations → Notifications/Mailbox
         ↓
      Vim (state machine) → MoreRight
         ↓
      Native-TS (color-diff, file-index, yoga)
```

---

## 8. OpenClaw 现有能力对照表

| OpenClaw 已有 | Claude Code 需移植 | 对齐策略 |
|-------------|-------------------|----------|
| `skills/` 架构 | Skills 加载器 | 适配 `loadSkillsDir()` |
| `memory/` 文件 | Memdir 索引 | 复用 `scanMemoryFiles()` 逻辑 |
| `state.ts` (AppState) | State 架构 | 已对齐，增 `tasks` map |
| 工具系统 | 工具池装配 | 已有 `toolRegistry`，添加 `assembleToolPool()` |
| `/permissions` 命令 | Permissions 8 步管道 | 需重写为 `hasPermissionsToUseToolInner()` |
| `telescope` | MCP 集成 | 升级为通用 MCP client |
| `git` 技能 | AgentTool worker | 实现 fork subagent |
| `gc` 命令 | Compaction | 实现 `compactConversation()` |
| `config` 命令 | Settings + Migrations | 添加 migrations 框架 |
| 快捷键系统？ | Keybindings | 需新建 |
| TUI 渲染？ | Ink | 需集成 `ink` 或自研 |

---

## 9. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| **Phase 0 延期** (权限管道复杂) | 整个项目延迟 2-4 周 | 并行开发：先实现简化版（rule-only），Classifier 延后 |
| **MCP 协议兼容性** | 第三方服务器无法连接 | 优先实现 stdio + SSE，HTTP 延后；严格测试 `mcp__` 规则 |
| **Sandbox 平台差异** (bwrap Linux only) | Windows/macOS 无法使用 | Windows: 用 `WindowsSandbox` 或 `wsl`？ macOS: `sandbox-exec` (deprecated) 或容器化 |
| **Buddy 渲染性能** | TUI 卡顿 | 使用 React Compiler 优化，或降级为静态 ASCII |
| **KAIROS daemon 管理** | 常驻进程崩溃 | 实现 watchdog (supervisor 或 `pm2`) |
| **Voice OAuth 复杂度** | 依赖 claude.ai login | 首次发布跳过 Voice，Phase 3 再试 |
| **Native-TS 编译** | 用户环境无 Rust/C++ | 提供预编译二进制或 JS fallback |
| **迁移数据兼容** | 旧版本 MEMORY.md 格式不兼容 | Migrations 框架 + 转换脚本 |

---

## 10. 测试策略

### 10.1 单元测试

- **Permissions**: 测试 8 步管道所有分支（allow/deny/ask）
- **Compaction**: 模拟 200K token 上下文，验证截断正确性
- **Query**: PTL 重试 3 次上限，插入 marker
- **AgentTool**: fork 子进程隔离，工具集限制
- **MCP**: 工具名转换 `mcp__<server>__<tool>` 正确性
- **Vim**: 状态机覆盖所有 `CommandState` 转换

### 10.2 集成测试

- **End-to-End Query**: 用户消息 → 工具调用 → 结果返回（全链路）
- **Agent 协作**: Coordinator 模式，3 个 worker 并行
- **Dream 自动记忆**: 会话达到阈值后自动触发压缩
- **MCP 服务器连接**: 真实 fs 服务器 + OAuth 流程
- **Memdir 索引**: 200 个文件扫描 < 2s

### 10.3 性能测试

- **Token 预算**: 200K 上下文二分搜索 < 10ms
- **Compaction**: 大型对话（100 轮）压缩 < 5s
- **Agent 启动**: fork subagent < 2s
- **MCP 工具调用**: 延迟 < 100ms（本地服务器）

### 10.4 安全测试

- **Path Traversal**: `../../../etc/passwd` 读取必须拦截
- **Sandbox Escape**: bwrap 配置验证（no-privileged mounts）
- **Classifier Bypass**: YOLO 模式下危险命令（rm -rf）必须 ask
- **MCP 权限泄露**: `mcp__filesystem` 工具不能越权访问其他项目

---

## 11. 发布计划

### 11.1 Alpha (Phase 0 + 部分 Phase 1)

**功能**:
- State + Query + Permissions (rule-based, 无 classifier)
- AgentTool 基础（fork，workspace 隔离）
- Compaction (manual only)
- MCP 基础（stdio 服务器）

**发布范围**:
- OpenClaw CLI v0.9.0-alpha.1
- 文档更新：Migration Guide
- 已知限制：无 KAIROS、Buddy、Coordinator、Voice

### 11.2 Beta (Phase 1 完成)

**功能**:
- Full Permissions (YOLO Classifier)
- Auto Compact (micro + full)
- System Prompt Sections + cache sharing
- Task 系统 + 任务面板
- Skills 加载器
- MCP 完整（OAuth、截断、权限）

**发布范围**:
- OpenClaw CLI v0.9.0-beta
- 所有核心能力就绪
- 开始 public testing

### 11.3 RC (Phase 2 完成)

**功能**:
- KAIROS + Dream
- Buddy 渲染
- Coordinator 模式
- Memdir 索引优化
- Keybindings + Ink 集成
- OutputStyles

**发布范围**:
- OpenClaw CLI v0.9.0-rc.1
- Feature complete
- Last bug fixes

### 11.4 Stable (Phase 3 完成)

**功能**:
- Voice (if ready)
- Vim 模式 (optional)
- MoreRight (optional)
- Native-TS 可选加速
- UpstreamProxy (CCR)

**发布范围**:
- OpenClaw CLI v1.0.0
- Production ready
- 完整文档 + 迁移指南

---

## 12. 资源分配建议

### 12.1 团队配置 (假设 3 人团队)

| 角色 | 人数 | Phase 0 | Phase 1 | Phase 2 | Phase 3 |
|------|------|---------|---------|---------|---------|
| **Core Engineer** | 1 | 全职 | 全职 | 全职 | 全职 |
| **Agent/Systems Engineer** | 1 | 50% | 全职 | 全职 | 50% |
| **UX/Infra Engineer** | 1 | 50% | 50% | 全职 | 全职 |
| **Total Person-Weeks** | - | 4 | 6 | 6 | 4 |
| **Total Duration** | - | 2w | 3w | 3w | 2w |

**总工期**: 10 周 (~2.5 个月)

### 12.2 单人开发者调整

- 每个 Phase 延长 1.5-2 倍时间
- 总工期 ~5-6 个月
- 建议优先完成 Phase 0 + Phase 1 (核心功能)，Phase 2/3 逐步迭代

---

## 13. 决策检查清单

在启动每个 Phase 前，确认：

- [ ] **依赖就绪**: 前一 Phase 的里程碑全部达成
- [ ] **测试覆盖**: 核心路径单元测试 > 80%
- [ ] **性能基线**: 关键操作延迟在可接受范围 (Query < 5s, Agent fork < 2s)
- [ ] **安全评审**: Permissions 管道、Sandbox、Path Validation 通过审计
- [ ] **文档同步**: 用户文档更新（新命令、设置选项）
- [ ] **回滚计划**: 若严重阻塞，可退回到前一稳定版本

---

## 14. 成功指标

### 14.1 功能完备性

- [ ] 支持 Claude Code 所有核心工具（≥30 个）
- [ ] AgentTool 可并行 spawn ≥5 个 worker
- [ ] MCP 至少 2 种 transport (stdio, SSE) 工作
- [ ] Permissions 8 步管道 100% 覆盖
- [ ] Auto Compact 每日自动触发 ≥1 次

### 14.2 性能

- [ ] P99 Query 延迟 < 10s（含网络）
- [ ] Token budget 200K 上下文稳定
- [ ] PTL 重试成功率 > 90%
- [ ] Memory index 扫描 < 2s (200 files)

### 14.3 可靠性

- [ ] 7x24 运行无崩溃（KAIROS daemon）
- [ ] Error classification 准确率 > 95%
- [ ] Sandbox 逃逸 0 次（安全测试）

---

## 15. 后续步骤 (Action Items)

### 立即执行 (Week 0)

1. **搭建分支结构**:
   ```
   main (stable)
   ├── phase-0-foundation
   ├── phase-1-agents
   ├── phase-2-experience
   └── phase-3-polish
   ```

2. **Phase 0 任务分配**: Core Engineer 负责 Query + Permissions；Agent Engineer 负责 AgentTool + Compaction

3. **环境准备**:
   - 安装 bwrap (Linux) 或准备替代方案
   - 配置 MCP 测试服务器（fs 服务器）
   - 准备 GrowthBook 本地 mock 或降级方案

4. **测试基础设施**:
   - 设置 Jest + 集成测试框架
   - 创建性能基准测试脚本
   - 搭建安全测试沙箱

---

## 16. 结语

这份路线图基于 **Claude Code 2.1.88 生产级架构** 的完整逆向工程，覆盖 **~123KB 文档** 和 **1910 文件** 的核心逻辑。

**OpenClaw 迁移不是重写，而是 transplant**——将 Claude Code 经过大规模 A/B 测试验证的工程实践，移植到 OpenClaw 的 extensible 架构上。

**关键成功因素**:
1. **Phase 0 必须稳**——State + Permissions 是所有上层建筑的地基
2. **Don't skip Tests**——每个 Phase 的里程碑必须有测试保障
3. **Monitor in production**——Analytics + Error Classification 确保问题早发现

**预期成果**: OpenClaw 将成为 **功能对齐 Claude Code 2.1.88** 的第二实现，具备多代理、压缩、MCP、KAIROS 等高级能力，同时保持 OpenClaw 的 modularity 和 agent-first 设计哲学。

---

**附录**:
- A: Claude Code 模块覆盖率矩阵 (8 份报告索引)
- B: OpenClaw 现有 API 对照表
- C: Permissions 8 步详细决策树
- D: Compaction 算法伪代码
- E: MCP 工具名转换规则

*报告完成于 2026-04-01 17:55 (Asia/Shanghai)*  
*分析师: 哈基米 (Agent)*  
*执行人: junshunG*