# Claude Code 终极深度解析 Part 2: Coordinator, Tasks, MCP, Skills, Bridge, Query Engine

分析师: 哈基米 (Agent)  
执行人: junshunG  
日期: 2026-04-01  
基于: canxiusi/ClaudeCode v2.1.88 (1910 files)

---

## 1. Coordinator Mode: 多代理协调架构

### 1.1 设计哲学

**Coordinator** 是 Claude Code 的多代理协作模式，CEO 角色：
- 不直接干活，只分配任务
- 使用 Worker 并行执行研究、实现、验证
- 自身只保留 Synthesis（整合）和 Direct Answers（直接回答）

### 1.2 启用机制

**Feature Flag**: `feature('COORDINATOR_MODE')`  
**Env**: `CLAUDE_CODE_COORDINATOR_MODE=1`

**状态持久化**:
```typescript
// session resume 时恢复模式
export function matchSessionMode(sessionMode: 'coordinator' | 'normal'): string | undefined {
  if (sessionIsCoordinator && !currentIsCoordinator) {
    process.env.CLAUDE_CODE_COORDINATOR_MODE = '1'
    return 'Entered coordinator mode to match resumed session.'
  }
  // ...
}
```

### 1.3 系统提示 (`getCoordinatorSystemPrompt()`)

**核心内容**:

```
You are Claude Code, an AI assistant that orchestrates software engineering tasks across multiple workers.

## 1. Your Role
- Help the user achieve their goal
- Direct workers to research, implement and verify code changes
- Synthesize results and communicate with the user
- Answer questions directly when possible — don't delegate work that you can handle without tools

## 2. Your Tools
- AgentTool() — Spawn a new worker
- SendMessageTool() — Continue an existing worker
- TaskStopTool() — Stop a running worker
- subscribe_pr_activity / unsubscribe_pr_activity

## 3. Task Workflow: Research → Synthesis → Implementation → Verification

## 4. Concurrency
Parallelism is your superpower. Workers are async. Launch independent workers concurrently whenever possible.

### Managing concurrency:
- Read-only tasks (research) — run in parallel freely
- Write-heavy tasks (implementation) — one at a time per set of files
- Verification can sometimes run alongside implementation on different file areas

## 5. Writing Worker Prompts (IMPORTANT)
Workers can't see your conversation. Every prompt must be self-contained with everything the worker needs.

### Always synthesize — your most important job
When workers report research findings, **you must understand them before directing follow-up work**.
Never write "based on your findings" or "based on the research."
These phrases delegate understanding to the worker instead of doing it yourself.
You never hand off understanding to another worker.

### Good example:
"Fix the null pointer in src/auth/validate.ts:42. The user field on Session is undefined when sessions expire but the token remains cached. Add a null check before user.id access — if null, return 401 with 'Session expired'. Commit and report the hash."

### Bad example:
"Based on your findings, fix the auth bug"

### Choose continue vs. spawn by context overlap

| Situation | Mechanism | Why |
|-----------|-----------|-----|
| Research explored exactly the files that need editing | **Continue** (SendMessage) | Worker already has files in context |
| Research was broad but implementation is narrow | **Spawn fresh** | Avoid dragging exploration noise |
| Correcting a failure | **Continue** | Worker has error context |
| Verification of code a different worker just wrote | **Spawn fresh** | Verifier should see code with fresh eyes |
| First implementation wrong entirely | **Spawn fresh** | Clean slate avoids anchoring |
| Completely unrelated task | **Spawn fresh** | No useful context to reuse |

## 6. Example Session (detailed)
```

### 1.4 Worker 工具集 (`getCoordinatorUserContext`)

**简单模式** (`CLAUDE_CODE_SIMPLE`):
```
[BASH_TOOL_NAME, FILE_READ_TOOL_NAME, FILE_EDIT_TOOL_NAME]
```

**完整模式** (`ASYNC_AGENT_ALLOWED_TOOLS`):
```typescript
export const ASYNC_AGENT_ALLOWED_TOOLS = new Set([
  FILE_READ_TOOL_NAME,
  WEB_SEARCH_TOOL_NAME,
  TODO_WRITE_TOOL_NAME,
  GREP_TOOL_NAME,
  WEB_FETCH_TOOL_NAME,
  GLOB_TOOL_NAME,
  ...SHELL_TOOL_NAMES,
  FILE_EDIT_TOOL_NAME,
  FILE_WRITE_TOOL_NAME,
  NOTEBOOK_EDIT_TOOL_NAME,
  SKILL_TOOL_NAME,
  SYNTHETIC_OUTPUT_TOOL_NAME,
  TOOL_SEARCH_TOOL_NAME,
  ENTER_WORKTREE_TOOL_NAME,
  EXIT_WORKTREE_TOOL_NAME,
])
```

**排除工具** (`ALL_AGENT_DISALLOWED_TOOLS`):
- `AgentTool` (防递归)
- `TaskOutputTool` (防递归)
- `ExitPlanModeTool` (主线程抽象)
- `TaskStopTool` (需主线程 task state)
- `TungstenTool` (单例虚拟终端冲突)
- `AskUserQuestionTool` (人工交互)

### 1.5 Scratchpad 共享目录

**Feature Gate**: `tengu_scratch`

**路径**: `getCoordinatorUserContext(scratchpadDir)` 传入

**用途**: 跨 Worker 持久化知识的共享文件系统

**权限**: Workers 可读写无需权限提示 (scratchpad 沙箱)

### 1.6 通信协议

**Worker 通知格式** (`<task-notification>`):
```xml
<task-notification>
<task-id>{agentId}</task-id>
<status>completed|failed|killed</status>
<summary>{human-readable status summary}</summary>
<result>{agent's final text response}</result>
<usage>
  <total_tokens>N</total_tokens>
  <tool_uses>N</tool_uses>
  <duration_ms>N</duration_ms>
</usage>
</task-notification>
```

**更新 Worker**:
- 通过 `SendMessageTool` 发送 `to: agentId` 继续会话
- 不通过 `AgentTool`（那会新建）

---

## 2. Task 系统: 任务生命周期管理

### 2.1 Task 状态机

**Base** (`src/Task.js`):
```typescript
type TaskStateBase = {
  id: string                // agentId or taskId
  type: string              // 'agent' | 'dream' | 'teammate' | ...
  status: 'running' | 'completed' | 'failed' | 'killed'
  createdAt: number
  updatedAt: number
  description?: string
  error?: string
}
```

### 2.2 Task 注册与更新

**核心 API** (`src/utils/task/framework.ts`):
```typescript
let tasks = new Map<TaskId, TaskState>()

export function registerTask(task: TaskState, setAppState): void {
  tasks.set(task.id, task)
  setAppState(s => ({ ...s, tasks: Map.from(tasks) }))
}

export function updateTaskState<T extends TaskState>(
  taskId: TaskId,
  setAppState: SetAppState,
  updater: (task: T) => T | null,
): void {
  const task = tasks.get(taskId)
  if (!task) return
  const updated = updater(task as T)
  if (updated) {
    tasks.set(taskId, updated)
    setAppState(s => ({ ...s, tasks: Map.from(tasks) }))
  } else {
    tasks.delete(taskId)
    setAppState(s => {
      const copy = new Map(s.tasks)
      copy.delete(taskId)
      return { ...s, tasks: copy }
    })
  }
}
```

### 2.3 任务面板 UI

**触发**: `Shift+Down` 打开任务列表弹窗

**底部 Pill**:
- 显示活跃任务数
- 最优先任务的状态（如 "dreaming", "researching"）
- 点击可打开任务面板

### 2.4 DreamTask 实现细节

**文件**: `src/tasks/DreamTask/DreamTask.ts`

**字段**:
```typescript
type DreamTaskState = TaskStateBase & {
  type: 'dream'
  phase: 'starting' | 'updating'
  sessionsReviewing: number
  priorMtime: number            // 锁文件旧 mtime（失败回滚用）
  filesTouched: string[]        // 通过 FileEdit/Write 编辑的路径
  turns: DreamTurn[]            // 助手回合历史（tool_use 计数 + 文本）
  abortController?: AbortController
}
```

**进度更新** (`addDreamTurn`):
- 提取 `text` + `tool_use` 计数
- 从 `FileEditTool` / `FileWriteTool` 的 `file_path` 参数收集 `filesTouched`
- 空回合且无新文件 → 跳过更新（避免重绘）

**完成/失败处理**:
- 成功: `completeDreamTask()` → 更新锁 mtime 到当前时间
- 失败: `failDreamTask()` + `rollbackConsolidationLock(priorMtime)`

---

## 3. MCP 集成: Model Context Protocol

### 3.1 架构概览

```
MCPConnectionManager.tsx (React context)
    ↓
useManageMCPConnections() —— 连接生命周期
    ↓
Client (from @modelcontextprotocol/sdk)
    ↓
Transport: Stdio / SSE / StreamableHTTP
    ↓
Tools: ListTools → convertToTool()
Resources: ListResources → ReadMcpResourceTool
Prompts: ListPrompts → (暂无工具，future)
```

### 3.2 连接管理 (`useManageMCPConnections`)

**配置来源**:
- `~/.claude/mcp.json` (用户配置)
- `prefetchOfficialMcpUrls()` (官方注册表)

**连接逻辑**:
1. 解析 `mcp.json` → `McpServerConfig[]`
2. 去重 (name + command/url) → 复用已有连接
3. 根据 `type` 选择 transport:
   - `stdio`: `StdioClientTransport` (本地命令)
   - `sse`: `SSEClientTransport` (HTTP event stream)
   - `http`/`https`: `StreamableHTTPClientTransport`
4. `client.connect()` → 获取 `tools` + `resources`
5. 转换: `convertToolsToAnthropicFormat()` 生成 `Tool[]`

### 3.3 工具转换

```typescript
function convertToolToAnthropic(tool: McpServerTool, serverName: string): Tool {
  return {
    name: buildMcpToolName(tool.name, serverName),  // e.g. "mcp__filesystem__read_file"
    description: tool.description,
    inputSchema: tool.inputSchema,  // JSON Schema (MCP)
    // MCP-specific metadata
    isMcp: true,
    mcpServerName: serverName,
    // ...
  }
}
```

### 3.4 权限规则作用域

支持 `mcp__<serverName>` 前缀规则:

```
allow:  mcp__filesystem
deny:   mcp__github  (危险操作)
ask:    mcp__slack
```

**过滤** (`filterToolsByDenyRules`):
```typescript
if (tool.isMcp && tool.mcpServerName) {
  const rule = findRule(`mcp__${tool.mcpServerName}`)
  if (rule.behavior === 'deny') removeTool(tool)
  if (rule.behavior === 'ask') markAsAsk(tool)
}
```

### 3.5 大输出处理

**截断策略** (`mcpValidation.ts`):
```typescript
const MAX_OUTPUT_TOKENS = 10_000
const TRUNCATION_WARNING = '... (output truncated)'

function truncateMcpContentIfNeeded(content, maxTokens = MAX_OUTPUT_TOKENS) {
  if (estimateTokens(content) <= maxTokens) return content
  return content.slice(0, maxTokens * 4) + TRUNCATION_WARNING  // 粗略 chars → tokens
}
```

**二进制存储**:
- 超限输出 → 保存到 `%TEMP%/claude/mcp-outputs/{timestamp}.bin`
- 返回 `file://` URI 供后续读取

### 3.6 OAuth 支持

**流程**: 通过 MCP server 的 `oauth` 配置 → 本地 OAuth 代理 → 获取 token → 注入请求头

**URL 构造**: `getOAuthCallbackUrl()` → `http://localhost:{port}/oauth/callback`

**Token 刷新**: `checkAndRefreshOAuthTokenIfNeeded()` 自动续期

---

## 4. Skills 系统: 可扩展技能包

### 4.1 发现与加载

**技能目录**:
- 内置: `src/skills/` (打包进二进制)
- 用户: `~/.claude/skills/` (运行时加载)

**加载逻辑** (`loadSkillsDir.ts`):
```typescript
export async function loadSkillsDir(dir: string, registry: SkillRegistry): Promise<void> {
  const entries = await fs.readdir(dir, { withFileTypes: true })
  for (const entry of entries) {
    if (!entry.isDirectory()) continue
    const skillDir = path.join(dir, entry.name)
    const skillJsonPath = path.join(skillDir, 'skill.json')
    const scriptPath = path.join(skillDir, 'skill.js')

    try {
      const manifest = await readJson<SkillManifest>(skillJsonPath)
      const script = await import(scriptPath)  // ES module
      registry.register(manifest, script.default)
    } catch (e) {
      logError(`Failed to load skill ${entry.name}:`, e)
    }
  }
}
```

### 4.2 Skill 工具执行 (`SkillTool`)

**结构**:
```typescript
const skillTool: Tool = {
  name: 'skill',
  description: 'Invoke a loaded skill by name with JSON arguments',
  inputSchema: z.object({
    skill: z.string(),
    arguments: z.record(z.any()),
  }),
  isEnabled: () => feature('EXPERIMENTAL_SKILL_SEARCH'),
  use: async (input, context) => {
    const skill = registry.get(input.skill)
    if (!skill) throw new Error(`Skill not found: ${input.skill}`)
    const result = await skill.execute(input.arguments, context)
    return { type: 'tool_result', content: [{ type: 'text', text: result }] }
  },
}
```

### 4.3 技能清单格式 (`skill.json`)

```json
{
  "name": "commit",
  "description": "Create a git commit with conventional format",
  "version": "1.0.0",
  "tags": ["git", "workflow"],
  "inputSchema": {
    "type": "object",
    "properties": {
      "message": { "type": "string", "description": "Commit message" },
      "all": { "type": "boolean", "default": false }
    },
    "required": ["message"]
  }
}
```

---

## 5. Bridge 模式: claude.ai 桥接

### 5.1 架构

**Bridge** (`src/bridge/`) 是 Claude Code 在 remote mode 下与 claude.ai 网页版通信的适配层。

**流程**:
```
Claude Code (local) → WebSocket → claude.ai (remote)
    ↑                                            ↑
 工具调用                                  网页版 Agent
    ↓                                            ↓
 结果返回                                     渲染
```

**实现** (`claudeai.ts`):
- 使用 `WebSocketTransport` (`utils/mcpWebSocketTransport.ts`)
- 消息格式: 适配 Anthropic SDK 的 `MessageParam`
- 认证: 从用户浏览器同步 session cookie (OAuth flow)

### 5.2 会话同步

**双向同步**:
- 本地 `messages[]` → remote 实时推送
- remote 流式响应 → 本地 `assistant` 消息增量更新
- 工具使用: remote 执行 → local `tool_result`

**优势**: 可使用 claude.ai 上的所有功能（Dream, Buddy, 实验性模型）而不需本地升级。

---

## 6. Query 引擎: 请求生命周期完整流

### 6.1 入口点 (`query.ts`)

**主函数**: `query(context, input, options)` → 返回 `AssistantMessage`

**步骤**:
1. **Pre-hooks**: `executePreQueryHooks()`
2. **Token counting**: `tokenCountWithEstimation()` 或精确 `tiktoken`
3. **Compaction**: `maybeCompact()` → 超过阈值触发
4. **Fetch system prompt parts**: `fetchSystemPromptParts()` (section memoization)
5. **Build messages**: `normalizeMessagesForAPI()` + `prependUserContext` + `appendSystemContext`
6. **Model selection**: `getMainLoopModel()` + effort override
7. **API call**: `queryModelWithStreaming()` → 流式响应
8. **Post-hooks**: `executePostQueryHooks()`
9. **Tool execution**: `runTools()` (如果存在 `tool_use`)
10. **Recursive**: 返回第 7 步，直到 max_turns 或无工具
11. **Finalize**: 生成最终 `AssistantMessage`，记录 `stop_reason`
12. **Save transcript**: `writeTranscript()`

### 6.2 PTL 重试逻辑

**触发点**: `parsePromptTooLongTokenCounts(response)` 检测

**策略** (`truncateHeadForPTLRetry`):
1. 按 API round 分组消息 (assistant + user + tool_result 循环)
2. 计算 token gap (超出窗口多少)
3. 估算每组平均 tokens → 需要丢弃的组数
4. 丢弃最旧的 N 组，保留至少 1 组
5. 插入 `PTL_RETRY_MARKER` 系统消息标记重试
6. 重试新消息列表，至多 `MAX_PTL_RETRIES = 3`

**反馈**: 记录 `tengu_ptl_retry` 事件用于监控。

### 6.3 工具执行循环 (`runTools`)

**流程**:
```typescript
while (true) {
  const { toolUseBlocks, stopReason } = extractToolUses(assistantMessage)
  if (!toolUseBlocks.length) break

  const results = await Promise.all(
    toolUseBlocks.map(block => runTool(block, context))
  )

  // 检查 permission 结果中的 stop flag
  if (results.some(r => r.stop)) return assistantMessage

  // 追加 tool_result 到历史
  messages.push(createToolResultMessage(block.id, results))

  // 再次调用模型
  assistantMessage = await queryModel(messages)
}
```

**并行度控制**: `maxConcurrentToolUses = 10` (configurable)

---

## 7. 错误分类与重试

### 7.1 Axios 错误五分类

```typescript
export function classifyAxiosError(e): {
  kind: 'auth' | 'timeout' | 'network' | 'http' | 'other'
  status?: number
  message: string
}
```

- **auth**: 401/403 → 不重试，提示用户重新认证
- **timeout**: ECONNABORTED (配置的 `timeout` 参数)
- **network**: ECONNREFUSED, ENOTFOUND → 指数退避重试
- **http**: 其他 5xx 或 4xx (429 单独处理)
- **other**: 非 axios 错误

### 7.2 通用重试封装 (`withRetry.ts`)

```typescript
export async function withRetry<T>(
  fn: () => Promise<T>,
  options: {
    maxRetries?: number
    retryable?: (e: unknown) => boolean
    onRetry?: (attempt: number, e: unknown) => void
  } = {},
): Promise<T> {
  const { maxRetries = 3, retryable = isRetryable, onRetry } = options
  let lastError
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn()
    } catch (e) {
      lastError = e
      if (!retryable(e) || attempt === maxRetries) throw e
      onRetry?.(attempt, e)
      await sleep(2 ** attempt * 1000)  // 指数退避
    }
  }
  throw lastError
}
```

**可重试判定**:
- `classifyAxiosError(e).kind in ['timeout', 'network', 'http']`
- 429 Rate Limit → 解析 `Retry-After` header 等待

---

## 8. Remote/CCR: Claude Code Remote

### 8.1 设计

**CCR** (Claude Code Remote) 允许将计算密集型 Agent 任务卸载到远程服务器。

**启用**: `CLAUDE_CODE_REMOTE_URL` 环境变量

**工作流**:
1. `AgentTool` 检测 `isolation: 'remote'`
2. `registerRemoteAgentTask()` 向 remote 服务注册
3. 返回 `sessionUrl` 用于进度轮询
4. 本地显示任务 pill，结果通过 `SendMessageTool` 转发

### 8.2 安全性

- Transport: HTTPS + Bearer token (从 `sessionIngressAuth` 获取)
- 沙箱: remote 服务器运行在隔离容器
- 权限: remote 继承调用者的 `permissionContext`

---

## 9. 工程化亮点 (续)

### 9.1 Lazy Schema 打破循环依赖

```typescript
const fullInputSchema = lazySchema(() => {
  return base.merge(extra).extend({ ... })
})
```

延迟执行，避免在模块顶层触发循环导入。

### 9.2 React Compiler 优化

**源码中** 大量 `_c(n)` hook ID 和 `$[]` 缓存数组:
```typescript
const $ = _c(31)  // 31 个记忆化槽
if ($[0] !== a || $[1] !== b) {
  $[0] = a; $[1] = b; $[2] = compute(a, b)
}
return $[2]
```

减少 React 渲染的重复计算（无需 `useMemo` 手写）。

### 9.3 Prompt Cache Sharing 实验

**Flag**: `tengu_compact_cache_prefix`

**实现**: `runForkedAgent()` 中设置 `forkContextMessages` 为固定系统提示 + 最近 N 条历史 → 与下一次压缩请求前缀相同 → 命中 shared cache。

**收益**: 每日 fleet-wide 节省 ~38B tokens

**关闭**: 3P 环境无 GrowthBook 时自动关闭。

### 9.4 Telemetry 安全

**标记类**:
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` → 确保日志不含代码/路径
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED` → PII 数据仅写入 `_PROTO_*` 字段

**双层路由**:
- `eventQueue` → `AnalyticsSink.logEvent()`
  - `Datadog` (stripped, no _PROTO)
  - `FirstPartyEventLogger` (BigQuery, keeps _PROTO)

---

## 10. OpenClaw 移植核心技术清单

| 功能 | 源码路径 | 难度 | 优先级 |
|------|---------|------|--------|
| Coordinator 多代理 | `coordinatorMode.ts` | ★★★ | P0 |
| Task 状态机 | `Task.js`, `tasks/*` | ★★ | P0 |
| Task 面板 UI | `screens/TasksScreen.tsx` | ★★ | P1 |
| MCP 工具聚合 | `services/mcp/client.ts` | ★★★ | P1 |
| MCP 大输出截断 | `utils/mcpValidation.ts` | ★ | P1 |
| Skill 加载器 | `skills/loadSkillsDir.ts` | ★ | P2 |
| Bridge WebSocket | `bridge/claudeai.ts` | ★★★ | P2 |
| Remote/CCR | `remote/` | ★★★ | P3 |
| Query PTL 重试 | `query.ts` `truncateHeadForPTLRetry` | ★★ | P1 |
| withRetry 通用 | `services/api/withRetry.ts` | ★ | P1 |
| Telemetry Sink | `services/analytics/index.ts` | ★ | P2 |
| GrowthBook 初始化 | `services/analytics/growthbook.ts` | ★★ | P2 |
| React Compiler 模板 | 全源码 `_c()` `$[]` | ★★★ | P3 |

---

## 11. 关键入口速查 (续)

| 功能 | 文件 | 函数 |
|------|------|------|
| 协调模式总控 | `coordinatorMode.ts` | `isCoordinatorMode()` |
| Worker 工具集 | `constants/tools.ts` | `ASYNC_AGENT_ALLOWED_TOOLS` |
| Worker 系统提示 | `coordinatorMode.ts` | `getCoordinatorUserContext()` |
| Task 注册 | `utils/task/framework.ts` | `registerTask()` |
| Task 更新 | `utils/task/framework.ts` | `updateTaskState()` |
| DreamTask | `tasks/DreamTask/DreamTask.ts` | `registerDreamTask()`, `addDreamTurn()` |
| MCP 连接管理 | `services/mcp/useManageMCPConnections.ts` | `useManageMCPConnections()` |
| MCP 工具转换 | `services/mcp/client.ts` | `convertToolToAnthropic()` |
| Skill 加载 | `skills/loadSkillsDir.ts` | `loadSkillsDir()` |
| Skill 执行 | `tools/SkillTool/SkillTool.ts` | `use()` |
| Bridge 适配 | `bridge/claudeai.ts` | `ClaudeAIBridge` class |
| Remote 注册 | `remote/registerRemoteAgentTask.ts` | `registerRemoteAgentTask()` |
| Query 主循环 | `query.ts` | `query()` |
| PTL 重试 | `query.ts` | `truncateHeadForPTLRetry()` |
| 重试封装 | `services/api/withRetry.ts` | `withRetry()` |
| Telemetry 初始化 | `services/analytics/index.ts` | `attachAnalyticsSink()` |
| GrowthBook 初始化 | `services/analytics/growthbook.ts` | `initGrowthBook()` |

---

报告完成于 2026-04-01 16:50 (Asia/Shanghai)  
总阅读量: 40+ 核心文件，约 6500 行代码  
状态: 准备发布