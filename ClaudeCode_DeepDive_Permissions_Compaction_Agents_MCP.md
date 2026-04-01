# Claude Code Deep Dive: Permissions, Compaction, Agents, MCP

分析师: 哈基米 (Agent)  
执行人: junshunG  
日期: 2026-04-01  
源码仓库: https://github.com/canxiusi/ClaudeCode (v2.1.88, 1910 files)

---

## 1. YOLO Classifier: Auto Mode 的 LLM 安全网关

### 1.1 系统提示工程

**Prompt 模板结构** (编译时内联或外部加载):

```
BASE_PROMPT = load('./yolo-classifier-prompts/auto_mode_system_prompt.txt')
EXTERNAL_PERMISSIONS_TEMPLATE = load('./yolo-classifier-prompts/permissions_external.txt')
ANTHROPIC_PERMISSIONS_TEMPLATE = (USER_TYPE==='ant') ? load('./.../permissions_anthropic.txt') : ''
```

模板支持 **3 段用户自定义规则** (通过 `<user_*_to_replace>` 标签):

- `user_allow_rules_to_replace`: 无条件允许的规则
- `user_deny_rules_to_replace`: 软拒绝规则 (可被覆盖的高风险)
- `user_environment_to_replace`: 环境变量/白名单路径

**规则提取函数**:

```typescript
function extractTaggedBullets(tag: string): string[] {
  const match = EXTERNAL_TEMPLATE.match(new RegExp(`<${tag}>([\\s\\S]*?)</${tag}>`))
  return (match[1] ?? '')
    .split('\n')
    .map(l => l.trim())
    .filter(l => l.startsWith('- '))
    .map(l => l.slice(2))
}
```

输出作为 `AutoModeRules` 结构注入 prompt。

### 1.2 分类器 Schema (工具调用)

Claude 以 **Custom Tool** 形式返回判断:

```typescript
const YOLO_CLASSIFIER_TOOL_SCHEMA = {
  type: 'custom',
  name: 'classify_result',
  description: 'Report the security classification result for the agent action',
  input_schema: {
    type: 'object',
    properties: {
      thinking: { type: 'string', description: 'Brief step-by-step reasoning.' },
      shouldBlock: { type: 'boolean', description: 'Whether the action should be blocked (true) or allowed (false)' },
      reason: { type: 'string', description: 'Brief explanation of the classification decision' },
    },
    required: ['thinking', 'shouldBlock', 'reason'],
  },
}
```

**响应解析** (`parseClassifierResponse`):
```typescript
const result = schema.safeParse(toolUseBlock.input)
return result.success ? result.data : null
```

### 1.3 Fast-path 与 Denial Tracking

**Fast-path 1: acceptEdits**  
`tool.checkPermissions(input, {mode:'acceptEdits'})` → 允许的文件编辑直接跳过分类器。

**Fast-path 2: allowlist**  
`isAutoModeAllowlistedTool(toolName)` 白名单（如只读工具）直接放行。

**Denial 计数器** (`denialTracking.ts`):
- `maxTotal: 50`
- `maxConsecutive: 10`
- 超限 → 降级为 `/ask` 避免死循环

**Headless 代理**:  
无交互时，禁用自动通过 → 默认 `deny`。

### 1.4 错误转储与调试

环境变量 `CLAUDE_CODE_DUMP_AUTO_MODE=1` → 将请求/响应对写入:

```
%TEMP%/claude/auto-mode/{timestamp}.req.json
%TEMP%/claude/auto-mode/{timestamp}.res.json
```

会话级别错误转储: `getAutoModeClassifierErrorDumpPath()` → `/share` 可收集。

---

## 2. Compaction: 多策略上下文压缩

### 2.1 触发阈值

```typescript
const AUTOCOMPACT_BUFFER_TOKENS = 13_000       // 安全余量
const WARNING_THRESHOLD = 20_000              // 警告线
const MAX_CONSECUTIVE_FAILURES = 3            // 失败后停用
```

计算方式:

```typescript
function getAutoCompactThreshold(model: string): number {
  const window = getContextWindowForModel(model) - MAX_OUTPUT_TOKENS_FOR_SUMMARY
  return window - AUTOCOMPACT_BUFFER_TOKENS
}
```

### 2.2 压缩策略

`compactConversation()` 支持两类压缩:

1. **全量压缩** (proactive/manual)
2. **部分压缩** (partial, direction='from'/'up_to')

**核心步骤**:
1. **Pre hooks**: `executePreCompactHooks()` → 允许扩展注入
2. **Generate summary**: 调用 `streamCompactSummary()`
   - **Cache-sharing path**: `runForkedAgent()` 复用主线程 prompt cache
   - **Fallback**: `queryModelWithStreaming()` + `FileReadTool` + optional `ToolSearchTool`
3. **Post hooks**: `processSessionStartHooks('compact')`
4. **重建附件**:
   - 最近访问的文件 (`createPostCompactFileAttachments`, 最多 5 个, 50K tokens)
   - 计划文件 (`createPlanAttachmentIfNeeded`)
   - 已调用技能 (`createSkillAttachmentIfNeeded`)
   - 异步代理状态 (`createAsyncAgentAttachmentsIfNeeded`)
   - MCP 增量指引 (`getMcpInstructionsDeltaAttachment`)
5. **创建边界消息**: `createCompactBoundaryMessage(...)` 含 preserved segment 元数据

### 2.3 预算与限制

| 常量 | 值 | 用途 |
|------|-----|------|
| `POST_COMPACT_MAX_FILES_TO_RESTORE` | 5 | 最多恢复的文件数 |
| `POST_COMPACT_TOKEN_BUDGET` | 50_000 | 文件附件总 token 上限 |
| `POST_COMPACT_MAX_TOKENS_PER_FILE` | 5_000 | 单个文件截断 |
| `POST_COMPACT_MAX_TOKENS_PER_SKILL` | 5_000 | 单个技能文件截断 |
| `POST_COMPACT_SKILLS_TOKEN_BUDGET` | 25_000 | 技能附件总上限 |

### 2.4 PTL 重试 (CC-1180)

当压缩请求本身触发 `prompt too long`:

```typescript
function truncateHeadForPTLRetry(messages, ptlResponse): Message[] | null {
  const groups = groupMessagesByApiRound(messages)
  if (groups.length < 2) return null

  const tokenGap = getPromptTooLongTokenGap(ptlResponse)
  let dropCount = tokenGap ? 估算需要删的组数 : Math.floor(groups.length * 0.2)
  dropCount = Math.min(dropCount, groups.length - 1)

  const sliced = groups.slice(dropCount).flat()
  if (sliced[0]?.type === 'assistant')
    return [createUserMessage({content: PTL_RETRY_MARKER, isMeta: true}), ...sliced]
  return sliced
}
```

最多重试 `MAX_PTL_RETRIES = 3` 次，失败则抛 `ERROR_MESSAGE_PROMPT_TOO_LONG`。

### 2.5 Cache Sharing 实验

Flag: `tengu_compact_cache_prefix` (default: true)

- **理论**: 使用 `runForkedAgent` 创建子进程，发送完全相同的 system/tools/messages 前缀 → 命中 shared prompt cache
- **收益**: 减少缓存失效，每天节省约 38B tokens (fleet-wide)
- **实现**: `streamCompactSummary()` 内优先尝试，失败 fallback 到普通 streaming

---

## 3. AgentTool 与 Subagent 隔离

### 3.1 工具定义

**输入 Schema** (feature-gated):

```typescript
const baseInputSchema = z.object({
  description: z.string(),
  prompt: z.string(),
  subagent_type: z.string().optional(),
  model: z.enum(['sonnet','opus','haiku']).optional(),
  run_in_background: z.boolean().optional(),
})

// Multi-agent extension
const multiAgentInputSchema = z.object({
  name: z.string().optional(),
  team_name: z.string().optional(),
  mode: permissionModeSchema().optional(),
})

// Isolation extension
const fullInputSchema = baseInputSchema.merge(multiAgentInputSchema).extend({
  isolation: z.enum(['worktree', 'remote']),   // ant only: remote CCR
  cwd: z.string().optional(),
})

// Gating
export const inputSchema = lazySchema(() => {
  let schema = feature('KAIROS') ? fullInputSchema() : fullInputSchema().omit({ cwd: true })
  return isBackgroundTasksDisabled || isForkSubagentEnabled() ? schema.omit({ run_in_background: true }) : schema
})
```

### 3.2 Fork Subagent 路径

**Feature**: `FORK_SUBAGENT` (与 coordinator 互斥)

触发条件：
- `subagent_type` 未指定
- 非 coordinator 模式
- 非交互式会话 (`!getIsNonInteractiveSession()`)

**代理定义** (`FORK_AGENT`):
```typescript
{
  agentType: 'fork',
  tools: ['*'],
  useExactTools: true,           // 继承父级完全相同的工具集
  permissionMode: 'bubble',      // 权限提示冒泡到父终端
  model: 'inherit',
}
```

**消息构建** (`buildForkedMessages`):
```typescript
// Result: [...history, assistant(all_tool_uses), user(placeholder_results..., directive)]
const fullAssistantMessage = { ...assistantMessage, uuid: randomUUID() }
const placeholderResults = tool_use_ids.map(id => ({
  type: 'tool_result' as const,
  tool_use_id: id,
  content: FORK_PLACEHOLDER_RESULT,
  is_error: false,
}))
return [fullAssistantMessage, createUserMessage({ content: [
  ...placeholderResults,
  { type: 'text', text: directive }
]})]
```

**目的**: 最大化 **prompt cache hit** — placeholder 结果固定，仅 directive 变化。

### 3.3 工作区隔离

**`isolation: 'worktree'`**:
- 创建临时 git worktree: `createAgentWorktree()`
- 复制当前 repo 到 `{tmp}/agent-worktrees/{agentId}`
- 代理的所有文件/ shell 操作限制在副本内
- 结束后 `removeAgentWorktree()` 清理

**`isolation: 'remote'`** (仅 ant):
- 启动远程 CCR (Claude Code Remote) 任务
- 注册: `registerRemoteAgentTask()`
- 提供 `sessionUrl` 用于监控

### 3.4 进度与通知

- `emitTaskProgress()`: 工具使用进度 (`tool_progress` 事件)
- `enqueueAgentNotification()`: 异步完成时通知父会话
- `background_hint`: 超过 `PROGRESS_THRESHOLD_MS = 2000` 显示后台任务提示

---

## 4. MCP (Model Context Protocol) 权限继承

### 4.1 MCP 集成点

**连接管理**: `src/services/mcp/MCPConnectionManager.tsx`
- 动态加载官方注册表: `prefetchOfficialMcpUrls()`
- 用户定义的 servers: `~/.claude/mcp.json`

**工具聚合**: `assembleToolPool()`
```typescript
const mcpTools = await fetchMcpTools()
const merged = uniqBy([...builtInTools, ...mcpTools], 'name')
```

### 4.2 权限规则作用域

**规则语法** 支持 `mcp__<serverName>` 前缀:

```
deny: mcp__filesystem    // 拒绝该 MCP server 的所有工具
ask: mcp__github         // 所有 GitHub MCP 工具需确认
allow: mcp__postgres    // 放行
```

**过滤逻辑** (`filterToolsByDenyRules`):
```typescript
if (tool.isMcp && tool.mcpServerName) {
  const rule = findRule(`mcp__${tool.mcpServerName}`)
  if (rule.behavior === 'deny') removeTool(tool)
  if (rule.behavior === 'ask') markAsAsk(tool)
}
```

### 4.3 通道权限 (Channel Permissions)

`channelPermissions.ts` 处理基于会话 channel 的权限 (如 Discord 频道限制)。

---

## 5. 关键代码片段库

### 5.1 YOLO Classifier 调用

```typescript
// permissions.ts:hasPermissionsToUseToolInner() 简化版
async function checkAutoMode(tool, input, context) {
  // 1. acceptEdits fast-path
  const fast = await tool.checkPermissions(input, { mode: 'acceptEdits' })
  if (fast.behavior === 'allow') return 'allow'

  // 2. allowlist
  if (isAutoModeAllowlistedTool(tool.name)) return 'allow'

  // 3. 构建 transcript
  const messages = formatTranscript(context.messages, tool, input)

  // 4. 调用分类器
  const result = await classifyYoloAction(messages, formatAction(tool, input), {
    model: 'claude-3-5-sonnet-...',
  })

  // 5. denial tracking
  if (result.shouldBlock) {
    recordDenial()
    if (exceedsDenialLimits()) return 'ask'  // 降级
    return 'deny'
  }
  recordSuccess()
  return 'allow'
}
```

### 5.2 Compaction 的 Fork 调用

```typescript
const result = await runForkedAgent({
  promptMessages: [summaryRequest],
  cacheSafeParams,  // 包含 forkContextMessages 用于 cache key
  canUseTool: createCompactCanUseTool(),
  querySource: 'compact',
  forkLabel: 'compact',
  maxTurns: 1,
  skipCacheWrite: true,
  overrides: { abortController: context.abortController },
})
```

### 5.3 AgentTool 的 Worktree 隔离

```typescript
async function runInWorktree(agentId, cwd, fn) {
  const worktree = await createAgentWorktree(agentId, cwd)
  try {
    return await runWithCwdOverride(worktree.path, fn)
  } finally {
    await removeAgentWorktree(agentId)
  }
}
```

---

## 6. 工程细节与最佳实践

1. **Dead-code elimination**: 所有 feature flag 分支在 Bun 打包时静态删除，减小生产包体积。
2. **Lazy schema**: `lazySchema(() => z.object(...))` 延迟编译，避免未用模块的解析开销。
3. **Prompt cache sharing**: 通过 fork 子进程 + 固定的 `forkContextMessages` 实现 byte-identical 前缀。
4. **Denial tracking**: 防止 auto-mode 死循环，触发降级。
5. **Hook system**: Pre/Post compaction hooks 允许插件扩展。
6. **PTL retry**: 压缩请求过长的自动截断重试，避免用户卡死。
7. **Attachment dedup**: `collectReadToolFilePaths()` 避免重复注入已可见的 Read 结果。

---

## 7. OpenClaw 移植建议

| 模块 | 可移植性 | 建议 |
|------|----------|------|
| YOLO Classifier | ★★★ | 可简化：用固定规则 + 轻量本地分类器，省去 LLM 调用开销 |
| Compaction (with fork cache sharing) | ★★ |  fork subagent 路径可保留，但需要子进程管理 |
| AgentTool worktree isolation | ★★ | 推荐实现，安全性高 |
| MCP permission scoping | ★★ | 规则引擎复用 permissions.ts，只需增加 `mcp__` 前缀匹配 |
| Denial tracking | ★ | 简单计数器，易实现 |

**Porting Order**:
1. `permissions` 管道 (8步流程)
2. `autoCompact` 的 micro + full 策略
3. `AgentTool` worktree 隔离
4. `MCP` permission scoping
5. 可选 YOLO classifier 替换为规则引擎

---

## 8. 未曝光但值得注意的设计

1. `tengu_compact_cache_prefix`: 默认 true，3P 环境关闭（GB 不可用）
2. `runForkedAgent()` 共享 prompt cache 依赖于 `forkContextMessages` 完全一致
3. `stripImagesFromMessages()` 将图片替换为 `[image]` 文本，防止压缩请求过长
4. `stripReinjectedAttachments()` 过滤已重新注入的技能列表，避免浪费
5. `createPostCompactFileAttachments()` 会排除 `claude.md` 和计划文件
6. `createSkillAttachmentIfNeeded()` 按 invoke 时间倒序，并截断到 5K tokens/技能
7. `partialCompactConversation()` 的 `up_to` 方向会失效缓存，`from` 方向保留前缀缓存
8. `forkSubagent` 递归检测: 通过 `<FORK_BOILERPLATE_TAG>` 文本标记防止无限 fork
9. `keepAlive` 机制: 压缩期间每 30s 发送一次 `sessionActivity` 保持 WebSocket
10. `autoCompact` 失败 3 次后自动停用，提示手动 `/compact`

---

报告完成于 2026-04-01 15:35 (Asia/Shanghai)  
总阅读: 14 核心文件，约 3000 行代码  
状态: 已准备发布