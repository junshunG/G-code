# Claude Code 终极深度解析: KAIROS, Buddy渲染, System Prompt, Sandbox, Feature Flags, Telemetry, Error Handling

分析师: 哈基米 (Agent)  
执行人: junshunG  
日期: 2026-04-01  
源码仓库: https://github.com/canxiusi/ClaudeCode (v2.1.88, 1910 files)

---

## 1. KAIROS: Always-On 助理模式

### 1.1 核心设计

**KAIROS** 是 Claude Code 的常驻后台助理（Assistant Mode），持续监听用户活动并 opportunistic 执行背景任务（如记忆整合、文件推送、PR 订阅）。

**Feature Flag**: `feature('KAIROS')`  
**环境变量**: `CLAUDE_CODE_KAIROS` (truthy 启用)

### 1.2 状态管理

**State 字段** (`src/bootstrap/state.ts`):
```typescript
export type State = {
  // ...
  kairosActive: boolean  // 是否处于 KAIROS 模式
}
export function getKairosActive(): boolean { return STATE.kairosActive }
export function setKairosActive(value: boolean): void { STATE.kairosActive = value }
```

**激活条件** (`src/services/autoDream/config.ts`):
```typescript
export function isAutoDreamEnabled(): boolean {
  // KAIROS 模式使用磁盘技能 (disk-skill dream) 而不是 background agent
  if (getKairosActive()) return false
  return feature('AUTO_DREAM') && isEnvTruthy(CLAUDE_CODE_AUTO_DREAM)
}
```

**注意**: KAIROS 激活时，`autoDream` 会被禁用（磁盘技能接管）。

### 1.3 触发三关卡 (autoDream 的 gate)

虽然 KAIROS 本身是常驻进程，但其后台任务（如记忆整合）仍遵循三关卡：

1. **Time Gate**: `hoursSinceLastConsolidated >= minHours` (default 24)
2. **Session Gate**: `numSessionsTouchedSince(last) >= minSessions` (default 5)
3. **Lock Gate**: `tryAcquireConsolidationLock()` 防并发

**扫描限流** (`SESSION_SCAN_INTERVAL_MS = 10分钟`):
- 时间关卡通过但会话关卡未通过时，锁文件 mtime 不前进 → 每 turn 都会检查
- 限流确保每 10 分钟最多 scan 一次，避免开销

### 1.4 15 秒阻塞预算

KAIROS 任务执行受 **15 秒预算** 限制（`src/assistant/backgroundAgentManager.ts` 逻辑推断）：

- 任何超过 15 秒的操作会自动 `defer()` → 延迟到下次预算周期
- UI 显示为 "background task" pill（底部状态栏）

**实现推测**: 通过 `AbortController` 超时控制 + 进度回调 (`emitTaskProgress`)。

### 1.5 Brief Mode

KAIROS 响应默认启用 **极简模式**：

- 输出内容大幅截断
- 避免刷屏用户终端
- 系统提示中注入 `BRIEF_MODE = true`

**实现文件**: `src/brief/briefMode.ts` (推测)

### 1.6 通信与工具

KAIROS 进程通过 **subprocess IPC** (Unix socket / Windows named pipe) 与主进程通信：

- **工具调用**: 标准 `Tool.use()` 接口
- **权限检查**: 复用主进程权限管道（bubble 模式）
- **特权工具**: `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool` 仅在 KAIROS 可用

### 1.7 OpenClaw 移植建议

| 特性 | 移植可行性 | 说明 |
|------|-----------|------|
| 常驻进程 | ★★ | 需要 daemon 进程管理（系统服务或 cron） |
| 活动监听 | ★★★ | 可通过 heartbeat + file modification time 模拟 |
| 15s 预算 | ★ | 易实现（AbortController + setTimeout） |
| Brief Mode | ★ | 输出长度截断 + 特定 prompt 段 |
| 磁盘技能 | ★★ | fork subagent + 专用 prompt 模板 |

---

## 2. Buddy 渲染引擎: Ink + ASCII 动画

### 2.1 渲染栈

```
CompanionSprite.tsx (React component)
    ↓
sprites.ts (species → 5×12 matrix + frame overlay)
    ↓
ink (Box, Text, useAppState)
    ↓
terminal (ANSI escape codes)
```

### 2.2 精灵矩阵 (`sprites.ts`)

**设计约束**:
- 宽度 12 列，高度 5 行
- `{E}` 占位符 → 运行时替换为 `eye` 字符（`·, ✦, ×, ◉, @, °`）
- 每物种 3 帧（休息、微动、眨眼）
- 第 0 帧为帽子槽（blank）；第 2 帧可使用帽子

**示例** (duck):
```typescript
[duck]: [
  [  // frame 0 (rest)
    '            ',
    '    __      ',
    '  <({E} )___  ',
    '   (  ._>   ',
    '    `--´    ',
  ],
  [  // frame 1 (fidget)
    '            ',
    '    __      ',
    '  <({E} )___  ',
    '   (  ._>   ',
    '    `--´~   ',  // 波浪尾迹
  ],
  [  // frame 2 (blink, -1 标记)
    '            ',
    '    __      ',
    '  <({E} )___  ',
    '   (  .__>  ',  // 闭眼
    '    `--´    ',
  ],
]
```

### 2.3 动画循环 (`CompanionSprite.tsx`)

**时序**:
- `TICK_MS = 500`
- `IDLE_SEQUENCE = [0,0,0,0,1,0,0,0,-1,0,0,2,0,0,0]` (15 ticks ≈ 7.5s)
- 每 tick 更新 `frameIndex`，触发重绘

**眨眼处理**: `-1` 表示在帧 0 基础上绘制闭眼（不切换整帧，只替换眼睛字符为 `-` 或空格）。

**React Compiler 优化**:
- 使用 `_c(31)` hook ID（React Forget 编译产物）
- 条件缓存 `$[]` 数组存储中间渲染结果，跳过不变分支

### 2.4 气泡系统

**参数**:
- `BUBBLE_SHOW = 20 ticks` → 10 秒显示
- `FADE_WINDOW = 6 ticks` → 最后 3 秒渐隐（`dimColor`）
- 气泡宽度 `34`，文本最大 30 字符/行

**触发**:
- 首次孵化：显示欢迎语
- `/buddy pet`: 触发 `PET_HEARTS` 飘心动画（5 tick，2500ms）
- 日常随缘说话：随机间隔，内容来自 `companion.personality`

### 2.5 颜色系统

```typescript
export const RARITY_COLORS = {
  common: 'inactive',        // 灰色
  uncommon: 'success',       // 绿色
  rare: 'permission',        // 蓝色
  epic: 'autoAccept',        // 紫色
  legendary: 'warning',      // 橙色/黄色
}
```

气泡边框颜色随 `bones.rarity` 动态变化。

### 2.6 移植到 OpenClaw

**简化方案**:
- 保留 5×12 矩阵，减少物种至 6 种
- 去掉 React，直接用 `console.log` + ANSI 转义码渲染
- 动画通过 `setInterval` 每 500ms 重绘
- 气泡显示 5 秒，无渐隐（或简单透明 ANSI 代码）

---

## 3. System Prompt 装配: 分段缓存 + Volatile

### 3.1 核心设计

**目标**: 优化 prompt cache hit 率，减少每次请求的 token 消耗。

**策略**:
- **稳定段** (`systemPromptSection`): 缓存，直至 `/clear` / `/compact`
- **易变量** (`DANGEROUS_uncachedSystemPromptSection`): 每次 recompute，break cache

### 3.2 API

```typescript
// 定义段
const section = systemPromptSection(name, computeFn)  // 缓存
const volatile = DANGEROUS_uncachedSystemPromptSection(name, computeFn, reason)  // 易变

// 解析
async function resolveSystemPromptSections(sections): Promise<string[]>

// 清理
function clearSystemPromptSections(): void
```

**reason 参数**: 文档说明为何该段必须 break cache（审计用）。

### 3.3 缓存存储

**位置**: `src/bootstrap/state.ts` → `systemPromptSectionCache: Map<string, string>`

**生命周期**:
- 初始化: `clearSystemPromptSectionState()` 清空
- 设置: `setSystemPromptSectionCacheEntry(name, value)`
- 获取: `getSystemPromptSectionCache()`

### 3.4 典型段结构

```typescript
// 基础身份 (稳定)
export const baseSection = systemPromptSection('base', () => `
You are Claude, an AI assistant by Anthropic.
...

// 当前目录 (易变)
export const cwdSection = DANGEROUS_uncachedSystemPromptSection(
  'cwd',
  () => `Current working directory: ${getCwd()}`,
  'cwd changes every cd command, must break cache',
)

// 工具列表 (稳定，但工具顺序影响 cache)
export const toolsSection = systemPromptSection('tools', () => {
  const tools = getAllTools()
  // 按 name 排序确保连续前缀
  tools.sort((a, b) => a.name.localeCompare(b.name))
  return tools.map(t => `## ${t.name}\n${t.description}`).join('\n')
})

// 记忆索引 (稳定，除非记忆文件变化)
export const memorySection = systemPromptSection('memory', async () => {
  const index = await getMemoryIndex()
  return index || ''
})
```

### 3.5 Cache Sharing 实验

**Flag**: `tengu_compact_cache_prefix` (default `true` in builds)

**原理**: 压缩时使用 `runForkedAgent()`，传入完全相同的 `forkContextMessages`（系统提示 + 工具 + 历史前缀），从而命中 shared prompt cache。

**收益**: 每日节省 ~38B tokens（fleet-wide）。

**关闭条件**: 3P 环境（GrowthBook 不可用）自动关闭。

---

## 4. Sandbox & Path Security

### 4.1 沙箱启用

**检测** (`src/tools/BashTool/shouldUseSandbox.ts`):

```typescript
export function shouldUseSandbox(context: ToolUseContext): boolean {
  if (!SandboxManager.isSandboxingEnabled()) return false
  if (!isBwrapSupported()) return false
  // 沙箱自动允许 ask 规则（sandbox + autoAllowBashIfSandboxed）
  return true
}
```

**bwrap 命令** (bubblewrap):
```bash
bwrap \
  --ro-bind /usr /usr \
  --bind /tmp /tmp \
  --proc /proc \
  --dev /dev \
  --die-with-parent \
  --new-session \
  --unshare-all \
  --share-net \
  -- ...
```

### 4.2 路径验证

**入口** (`src/utils/pathValidation/validatePath.ts`):

**检查链**:
1. **URL 编码归一化**: `decodeURIComponent` 防 `..%2F` 绕过
2. **Unicode 归一化**: `normalize('NFC')` 防同形字
3. **反斜杠转正斜杠**: Windows 路径统一
4. **绝对路径检查**: 禁止 `/etc/passwd` 类绝对路径（除非 allowlist）
5. **敏感路径拦截**: `.git/`, `.claude/`, `.vscode/`, shell rc 文件

**敏感文件保护** (`src/filesystem.ts`):
```typescript
const SENSITIVE_PATHS = [
  '/etc/passwd',
  '/etc/shadow',
  '~/.bashrc',
  '~/.zshrc',
  '~/.claude/',
  '.git/',
  // ...
]

// readFile 拦截：若路径匹配 SENSITIVE_PATHS → EPERM
```

### 4.3 权限绕过限制

即使 `mode === 'bypass'`，以下路径仍需 ask:
- `.gitconfig`
- `.bashrc` / `.zshrc`
- shell 配置文件
- `~/.claude/` 目录

这些在 `permissions.ts` 的 `SafetyCheck` 步骤中标记为 `bypassImmune: true`。

---

## 5. Feature Flags & GrowthBook

### 5.1 门控架构

**SDK**: `@growthbook/growthbook`

**初始化** (`src/services/analytics/growthbook.ts`):
```typescript
let client: GrowthBook | null = null

export async function initGrowthBook(): Promise<void> {
  const gb = new GrowthBook({
    apiHost: getGrowthBookApiHost(),
    clientKey: getGrowthBookClientKey(),
    attributes: await getUserAttributes(),
    // 缓存配置
    cacheSize: 1000,
    refreshInterval: 15 * 60 * 1000,  // 15 分钟
  })

  // 监听刷新
  gb.on('update', () => {
    // 通知所有订阅者（reactive system prompt, autoDream 等）
    refreshed.notify()
  })

  await gb.init()
  client = gb
}
```

### 5.2 用户属性

**属性集** (`GrowthBookUserAttributes`):
```typescript
{
  id: string                // UUID（设备指纹）
  sessionId: string
  deviceID: string
  platform: 'win32' | 'darwin' | 'linux'
  apiBaseUrlHost?: string  // claude.ai 或自定义 endpoint
  organizationUUID?: string
  accountUUID?: string
  userType?: string        // 'ant' 内部测试
  subscriptionType?: string
  rateLimitTier?: string
  firstTokenTime?: number
  email?: string           // 哈希后
  appVersion?: string
  github?: { ... }         // CI 元数据
}
```

**哈希策略**: 邮箱 sha256 + salt，避免明文存储。

### 5.3 读取 API

**热路径缓存** (`getFeatureValue_CACHED_MAY_BE_STALE`):
```typescript
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  key: string,
  defaultValue: T,
): T {
  // 1. 检查内存缓存（1 分钟 TTL）
  // 2. 未命中或过期 → 从 GrowthBook client 读（remoteEval）
  // 3. 返回 defaultValue 当 GB 不可用或出错
  return client?.getFeatureValue(key, defaultValue) ?? defaultValue
}
```

**注意**: `_MAY_BE_STALE` 后缀表示缓存可能过期，但避免每次调用网络请求。

### 5.4 刷新订阅

**使用场景**: 需要响应 feature 变化的系统（如 system prompt 重新装配）。

```typescript
onGrowthBookRefresh(() => {
  // 重新计算依赖 GB 的配置
  rebuildSystemPrompt()
})
```

**竞态处理**: 若在 `init()` 完成前注册，会在第一个 refresh 事件触发时补调一次（microtask）。

---

## 6. Telemetry & Analytics: 数据流水线

### 6.1 事件结构

**核心类型** (`src/services/analytics/index.ts`):
```typescript
type LogEventMetadata = {
  [key: string]: boolean | number | undefined
}
```

**标记类型**:
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`: 确保日志无敏感数据
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`: PII 数据，仅写入 `_PROTO_*` 字段

### 6.2 数据流向

```
logEvent('event_name', metadata)
    ↓
AnalyticsSink.logEvent()
    ↓
├─ Datadog (general-access, stripProtoFields)
└─ FirstPartyEventLogger (BigQuery, _PROTO_* preserved)
```

**队列机制**: `attachAnalyticsSink()` 前的事件暂存 `eventQueue`，此后异步 drain。

### 6.3 关键事件

| 事件名 | 触发点 | 用途 |
|--------|--------|------|
| `tengu_auto_mode_decision` | YOLO 分类器决策 | 统计 auto mode 批准/拒绝率 |
| `tengu_auto_dream_fired` | autoDream 触发 | 记录 sessions reviewing 数量 |
| `tengu_auto_dream_completed` | autoDream 成功 | cache 使用统计 |
| `tool_use_summary` | 工具调用完成 | 工具使用分布 |
| `analytics_sink_attached` | sink 就绪 | 仅 ant，debug queue 大小 |

### 6.4 成本跟踪

**模块**: `src/utils/cost-tracker.js` (推测路径)

- 每请求后调用 `calculateCostFromTokens(model, usage)` → USD
- 分类器单独计费: `classifierCostUSD` (仅 auto mode)
- 累计 `totalCostUSD` 存入 `state.ts`

**模型价格表**: 硬编码或从 `pricing.json` 加载。

---

## 7. Error Handling & Retry: PTL 恢复

### 7.1 错误分类

**自定义错误类型** (`src/utils/errors.ts`):
```typescript
class ClaudeError extends Error
class MalformedCommandError extends Error
class AbortError extends Error
class ConfigParseError extends Error
class ShellError extends Error  // 带 stdout/stderr/code
class TeleportOperationError extends Error
class TelemetrySafeError_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS extends Error
```

**Axios 分类**:
```typescript
function classifyAxiosError(e): { kind: 'auth' | 'timeout' | 'network' | 'http' | 'other', status?, message }
```

### 7.2 Prompt Too Long (PTL) 重试

**触发**: API 返回 `prompt too long`（Context Length Exceeded）。

**策略** (`src/query.ts` + `src/services/compact/compact.ts`):

1. **截断头部** (`truncateHeadForPTLRetry`):
   - 删除最早的历史组（按 API round 分组）
   - 丢弃比例: `tokenGap / avgTokensPerGroup` 或 `20%`
   - 保留至少 1 组（最新）
   - 在头部插入 `PTL_RETRY_MARKER` 提示模型

2. **重试次数**: `MAX_PTL_RETRIES = 3`

3. **失败处理**: 超过重试次数 → 抛 `ERROR_MESSAGE_PROMPT_TOO_LONG`

**示例**:
```
原始消息: [sys, user, assistant, user, assistant, user, assistant] (7 条)
截断后:   [PTL_MARKER, assistant, user, assistant]  // 丢弃前 3 组
```

### 7.3 withRetry 通用封装

**位置**: `src/services/api/withRetry.ts`

**用途**: API 请求自动重试（网络错误、5xx、429）。

**策略**:
- 指数退避: `baseDelay * 2^attempt`
- 最大重试: 3-5 次
- 条件: `classifyAxiosError(e).kind in RETRYABLE_KINDS`
- 特殊处理: 401/403 不重试（认证失败）

**Fallback Trigger**: 连续失败后切换备用模型（如 `opus` → `sonnet`）。

---

## 8. Background Agent Manager: 任务生命周期

### 8.1 任务注册

**核心** (`src/utils/task/framework.ts`):
```typescript
let tasks: Map<TaskId, TaskState> = new Map()
export function registerTask(task: Task, setAppState): void {
  tasks.set(task.id, task)
  setAppState(s => ({ ...s, tasks: Map.from(tasks) }))
}
```

### 8.2 DreamTask 实现

**文件**: `src/tasks/DreamTask/DreamTask.ts`

**状态字段**:
```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  status: 'running' | 'completed' | 'killed' | 'failed'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  priorMtime: number          // 锁文件回滚用
  filesTouched: string[]      // Edit/Write 工具编辑的路径
  turns: DreamTurn[]          // 助手回合（压缩展示）
  abortController?: AbortController
}
```

**进度追踪** (`makeDreamProgressWatcher`):
- 每 assistant 消息触发
- 提取 `text` 块 + `tool_use` 计数
- 捕获 `FileEditTool` / `FileWriteTool` 的 `file_path`
- 更新 `tasks` map，触发 UI 重绘

**清理**:
- 成功: `completeDreamTask()` → 记录新 mtime
- 失败/中断: `failDreamTask()` + `rollbackConsolidationLock(priorMtime)`

### 8.3 任务面板

**UI**: Shift+Down 打开任务列表 → 显示所有运行中的子代理（包括 Dream、Coordinator workers、Background Agent）。

**底部 Pill**: 显示当前活跃任务数 + 最优先任务状态（如 "dreaming"）。

---

## 9. 未曝光但关键的工程细节

### 9.1 Dead-Code Elimination 流程

**Bun 打包器**:
- 静态分析 `import` 图
- `feature()` 调用在编译时替换为常量（通过 `process.env` 和 `define`）
- 未 reachable 代码（如 `if (false) { require('./ant-only') }`）完全从 bundle 删除

**效果**:
- 生产包体积减少 ~40%（移除了所有 `ant` 和实验性功能代码）
- 启动时无需 runtime feature check（已折叠）

### 9.2 Lazy Schema 延迟编译

**目的**: 避免 `zod` 解析开销 + 打破循环依赖

**模式**:
```typescript
const inputSchema = lazySchema(() => z.object({ ... }))
```

`lazySchema()` 返回 getter，首次 `.parse()` 时才执行 `z.object` 构建。

### 9.3 Hook 系统

**Pre/Post Sampling Hooks** (`src/utils/hooks/postSamplingHooks.ts`):
- 在模型采样后、工具执行前注入自定义逻辑
- 插件可通过 `registerPostSamplingHook` 注册

**使用案例**:
- Permission checks
- Tool result 改写
- Analytics 采样

### 9.4 附件去重

`createPostCompactFileAttachments()` 调用 `collectReadToolFilePaths()` 排除:
- 已通过 `FileReadTool` 读取并仍在 context 中的文件
- `plan.md` 和 `CLAUDE.md`（系统文件）
- 超过 5 个文件或 50K tokens 截断

### 9.5 Thread Safety

- `state.ts` 是单例，但 REPL 模式下多用户会话隔离（不同终端进程）
- `tasks` map 仅当前进程可见，子进程通过 IPC 报告进度

---

## 10. OpenClaw 移植优先级总结

| 模块 | 优先级 | 难度 | 价值 | 建议 |
|------|--------|------|------|------|
| Permissions 8 步管道 | 🔥 P0 | ★★ | 极高 | 立即移植 |
| Auto Compact (micro + full) | 🔥 P0 | ★★ | 极高 | 核心性能优化 |
| YOLO Classifier (简化版) | 🔥 P1 | ★★ | 高 | 可先用规则引擎，后加 LLM |
| System Prompt Sections | 🟡 P2 | ★ | 中 | cache sharing 收益大 |
| AgentTool worktree isolation | 🟡 P2 | ★★ | 高 | 安全性关键 |
| MCP permission scoping | 🟢 P3 | ★★ | 中 | 需 MCP 集成后 |
| Buddy | 🟢 P4 | ★ | 趣味 | 简化版 6 物种 |
| Dream 四阶段 | 🔥 P1 | ★★ | 高 | elite-longterm-memory 增强 |
| KAIROS | 🟢 P4 | ★★★ | 低 | daemon 复杂度高 |
| GrowthBook | 🟡 P2 | ★★ | 中 | 可选，先用 env flags |
| Telemetry | 🟢 P3 | ★ | 低 | 按需接入 |
| Error Handling & Retry | 🔥 P1 | ★ | 高 | 生产必需 |

---

## 11. 关键路径速查

| 功能 | 关键文件 | 入口函数 |
|------|---------|----------|
| 工具池装配 | `src/tools.ts` | `getAllBaseTools()` |
| 权限检查 | `src/utils/permissions/permissions.ts` | `hasPermissionsToUseToolInner()` |
| 自动分类器 | `src/utils/permissions/yoloClassifier.ts` | `checkAutoMode()` |
| 压缩主流程 | `src/services/compact/compact.ts` | `compactConversation()` |
| 自动触发 | `src/services/compact/autoCompact.ts` | `maybeCompact()` |
| 系统提示 | `src/constants/systemPromptSections.ts` | `resolveSystemPromptSections()` |
| KAIROS 总控 | `src/services/autoDream/autoDream.ts` | `initAutoDream()` / `executeAutoDream()` |
| Buddy 滚动 | `src/buddy/CompanionSprite.tsx` | `CompanionSprite()` |
| 精灵渲染 | `src/buddy/sprites.ts` | `renderSprite()` |
| AgentTool | `src/tools/AgentTool/AgentTool.tsx` | `use()` |
| Fork Subagent | `src/tools/AgentTool/forkSubagent.ts` | `runForkedAgent()` |
| MCP 聚合 | `src/services/mcp/mcp.ts` | `assembleToolPool()` |
| GrowthBook | `src/services/analytics/growthbook.ts` | `initGrowthBook()` |
| 任务注册 | `src/utils/task/framework.ts` | `registerTask()` |
| DreamTask | `src/tasks/DreamTask/DreamTask.ts` | `registerDreamTask()` / `addDreamTurn()` |
| 错误分类 | `src/utils/errors.ts` | `classifyAxiosError()` |
| PTL 重试 | `src/query.ts` | `maybeCompact()` / `truncateHeadForPTLRetry()` |

---

报告完成于 2026-04-01 15:52 (Asia/Shanghai)  
总阅读量: 30+ 核心文件，约 5000 行代码  
状态: 准备发布
