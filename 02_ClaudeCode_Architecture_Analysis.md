# Claude Code 2.1.88 完整源码分析报告

> 基于 https://github.com/canxiusi/ClaudeCode.git 还原的 TypeScript 源码  
> Clone 路径: `claude-code-canxiusi/` | 规模: 1902 文件, 1332 个 `.ts`  
> 分析日期: 2026-04-01 | 分析师: 哈基米 (Agent) 执行人：junshunG

---

## 📊 执行摘要

Claude Code 是 Anthropic 官方 CLI AI 编码助手。2026-03-31，其 npm 包因 `.map` 文件泄露暴露了完整的 TypeScript 源码（版本 2.1.88）。本报告对该源码进行架构级逆向分析，揭示其核心设计模式、安全模型与隐藏功能。

**关键发现**：
- ✅ 完整的 **40+ 工具系统**，支持 feature flag 裁剪与 MCP 集成
- ✅ 三层 **权限安全管道**：规则过滤 + 检查拦截 + YOLO 自动分类器
- ✅ 智能 **上下文压缩**：阈值触发、多策略（micro/auto/session）
- ✅ 实验性功能：**KAIROS** (always-on), **Coordinator** (多代理), **Buddy** (Tamagotchi), **Dream** (自省记忆整合)
- ✅ 系统提示分段缓存 + 动态装配，优化 prompt cache hit 率
- ✅ 工程化细节：dead-code elimination、sandbox、transcript classifier、denial tracking

---

## 1. 架构总览

### 1.1 模块分层

```
main.tsx ── 启动装配层
├── commands.ts ── 命令入口层 (207 文件)
├── tools.ts ── 工具池装配层 (40+ tools)
├── Tool.ts ── 工具抽象层 (契约)
├── QueryEngine.ts / query.ts ── 查询执行内核
├── REPL.tsx ── 交互态控制中心
├── ink/ ── 终端渲染 (96 文件)
├── components/ ── 终端 UI 组件 (389 文件)
├── services/
│   ├── compact/ ── 上下文压缩 (10 文件)
│   ├── autoDream/ ── 背景记忆整合
│   ├── mcp/ ── Model Context Protocol 集成
│   ├── analytics/ ── 统计分析 (GrowthBook)
│   └── ...
├── utils/
│   ├── permissions/ ── 权限管道 (18 文件)
│   ├── processUserInput/ ── 输入预处理
│   ├── queryContext/ ── 上下文构建
│   └── ...
└── 特殊模式 (feature-gated)
    ├── coordinator/ ── 多代理协调
    ├── assistant/ (KAIROS) ── 常驻助理
    ├── buddy/ ── 伴侣宠物
    └── bridge/ ── claude.ai 桥接
```

### 1.2 启动流程

**main.tsx 关键路径**:
```
1. 并行预热: profileCheckpoint, startMdmRawRead, startKeychainPrefetch
2. 解析 CLI 参数 (commander)
3. 初始化配置: applyConfigEnvironmentVariables, init()
4. 装配命令池: getCommands() → filterCommandsForRemoteMode()
5. 装配工具池: getTools(permissionContext) → getAllBaseTools() + feature gates
6. 选择运行模式:
   - REPL: launchRepl() → REPL.tsx → handlePromptSubmit() → processUserInput() → query()
   - Headless: QueryEngine.submitMessage() → query()
7. query() → runTools() → StreamingToolExecutor → tool.use()
```

**性能优化**：
- MDM & Keychain 并行预读（macOS ~65ms）
- Bun 打包 + dead-code elimination 移除未启用 feature 代码
- GrowthBook 特性门控缓存 (`getFeatureValue_CACHED_MAY_BE_STALE`)
- 系统提示分段缓存 (`systemPromptSection` + `DANGEROUS_uncachedSystemPromptSection`)

---

## 2. 工具系统 (Tools System)

### 2.1 工具池装配

**入口**: `src/tools.ts:getAllBaseTools()` → `getTools(permissionContext)`

```typescript
export function getAllBaseTools(): Tools {
  return [
    // 核心
    AgentTool, TaskOutputTool, BashTool,
    // 文件
    FileReadTool, FileEditTool, FileWriteTool, NotebookEditTool,
    // 搜索
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    // 网络
    WebFetchTool, WebSearchTool,
    // 交互
    AskUserQuestionTool, EnterPlanModeTool, ExitPlanModeV2Tool,
    // MCP
    ListMcpResourcesTool, ReadMcpResourceTool, MCPTool, McpAuthTool,
    // 任务管理 (todo v2)
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    // 扩展
    SkillTool, WebBrowserTool, LSPTool, WorkflowTool,
    // 后台/协调
    BriefTool, SendMessageTool, TaskStopTool,
    ...(SleepTool ? [SleepTool] : []),
    ...cronTools, RemoteTriggerTool, MonitorTool,
    // 内测专属
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool, REPLTool] : []),
    // KAIROS 特有
    ...(SendUserFileTool ? [SendUserFileTool] : []),
    ...(PushNotificationTool ? [PushNotificationTool] : []),
    ...(SubscribePRTool ? [SubscribePRTool] : []),
    // 调试
    ...(process.env.NODE_ENV === 'test' ? [TestingPermissionTool] : []),
    SnipTool, CtxInspectTool, TerminalCaptureTool, OverflowTestTool,
  ]
}
```

**过滤逻辑** (`getTools`):
- `CLAUDE_CODE_SIMPLE` → 仅 `[Bash, FileRead, FileEdit]`
- `REPL mode` → 隐藏底层工具，仅暴露 `REPLTool`
- `filterToolsByDenyRules` → 基于 permissionContext 移除被规则禁用的工具（包括 MCP server-level 规则）
- `tool.isEnabled()` → 运行时 feature gate（如 `isPowerShellToolEnabled`）

### 2.2 工具契约

**Tool 核心接口**:

```typescript
interface Tool {
  name: string
  description: SystemPrompt
  inputSchema: z.ZodObject<...>
  permissions: Permission[] // 或通过 checkPermissions 实现
  isEnabled(): boolean
  isHiddenFromSearch?.(): boolean
  isAvailableWhileInProgress?(): boolean
  requiresUserInteraction?(): boolean
  checkPermissions(input: z.infer<typeof this.inputSchema>, context: ToolUseContext): Promise<PermissionResult>
  use(input: ..., context: ToolUseContext): Promise<ToolResultBlockParam>
  // 可选进度回调
  onStreamProgressURN?(): string
}
```

**PermissionResult**:
```typescript
type PermissionResult =
  | { behavior: 'allow', updatedInput?, decisionReason?, suggestions? }
  | { behavior: 'deny', message?, decisionReason?, suggestions? }
  | { behavior: 'ask', message?, decisionReason?, suggestions? }  // 需要用户批准
  | { behavior: 'passthrough' }  // 继续检查
```

### 2.3 工具特性

| 工具 | 关键能力 | 安全特性 |
|------|---------|---------|
| **BashTool** | 支持 `&&`, `;`, `sudo` 检测, 日志捕获 | `sandbox` (bwrap), `subcommandRules` (deny/allow 前缀), `checkPathSafety` |
| **FileEditTool** | 机房编辑 (search/replace), `apply_patch` | `perDirectoryEdits` 限制, `.claude/` 屏蔽 |
| **AgentTool** | 启动子代理, 自定义 agents, 内存快照 | `allowedAgents` 过滤, `localDenialTracking` 隔离 |
| **WebFetchTool** | 内容提取, CSS 选择器, 截断 | 域名限制, `maxLength`, `rate limiting` |
| **Task\*** | 并发任务生命周期管理 | N/A |

---

## 3. 权限安全模型 (Permissions & Security)

### 3.1 检查管道

**主函数**: `src/utils/permissions/permissions.ts:hasPermissionsToUseToolInner()`

**8 步流程**:

```
Step 1a: Entire tool denied by rule? → deny
Step 1b: Entire tool always-ask rule? → ask (除非 sandbox auto-allow)
Step 1c: tool.checkPermissions() - tool-specific logic (e.g. Bash subcommand rules)
Step 1d: Tool implementation denied? → deny
Step 1e: requiresUserInteraction && ask? → ask
Step 1f: Content-specific ask rules (e.g. Bash(npm publish:*))? → ask
Step 1g: SafetyCheck bypass-immune? → ask (必须人工确认)
Step 2a: bypassPermissions mode or plan bypass? → allow
Step 2b: Entire tool allowed by rule? → allow
Step 3: Passthrough → ask
```

**注意**：SafetyCheck（如编辑 `.gitconfig`, `.bashrc`）在 bypass 模式下也必须 ask，不可绕过。

### 3.2 规则系统

**规则来源** (优先级从低到高):
- `policySettings` / `flagSettings` (只读, remote-managed)
- `userSettings`, `projectSettings`, `localSettings` (磁盘持久)
- `cliArg` (命令行 `--permission`, `--allow`)
- `session` (会话内临时更改)

**规则语法**:
```
允许:   Bash(prefix:*)       - 所有 Bash 命令
      FileEdit(*.py:*)      - 单一扩展名或全路径
      Git(*)               - Git 工具通配
拒绝:   Bash(rm -rf /*)     - 危险命令前缀
      FileEdit(.git/*)     - 保护目录
询问:   PowerShell(*)       - 高危工具默认询问
```

**匹配逻辑**:
- 整工具匹配（无 `ruleContent`）：`toolMatchesRule`
- 内容匹配（前缀/路径）：`ruleContent` 字段 + `tool.checkPermissions` 联合判定
- MCP server 级别：`mcp__server1` → 移除所有该 server 的工具

### 3.3 Auto Mode (YOLO Classifier)

**前提**:
- `feature('TRANSCRIPT_CLASSIFIER')` 启用
- 权限模式 `mode==='auto'`

**流程** (`yoloClassifier.ts`):

```
hasPermissionsToUseToolInner → behavior === 'ask' → mode==='auto'? 

Yes → 1. 检查 acceptEdits fast-path:
        tool.checkPermissions(input, {mode:'acceptEdits'})
        → 'allow' 则直接通过 (免分类器)

     2. 检查 allowlist fast-path:
        classifierDecisionModule.isAutoModeAllowlistedTool(tool.name)
        → 直接通过

     3. 调用 classifyYoloAction(messages, formatActionForClassifier(tool, input), ...)

        - 输入: 系统提示 (auto_mode_system_prompt.txt) + 动作字符串
        - 输出: { shouldBlock: boolean, reason?, usage, durationMs, ... }

        - 失败处理: 
          * transcriptTooLong → 抛 AbortError (headless) 或 降级为 ask
          * unavailable → 查 tengu_iron_gate_closed:
              - closed → deny (fail-closed)
              - open → 降级为 ask (fail-open)

     4. 统计 denial tracking:
        - shouldBlock → recordDenial()
        - 检查 DENIAL_LIMITS (maxTotal/consecutive) → 超限则降级 ask 让用户介入
        - allow → recordSuccess() 重置计数

返回: allow / deny (with reason: classifier)
```

**Denial Tracking** (`denialTracking.ts`):
```typescript
type DenialTrackingState = {
  consecutiveDenials: number
  totalDenials: number
  lastDenialAt?: number
}
const DENIAL_LIMITS = { maxTotal: 50, maxConsecutive: 10 }
// 超过限制 → 降级提示，break auto-mode 死循环
```

**Fast-Path 保护**:
- **acceptEdits**: 只允许 `FileWriteTool`, `FileEditTool` 在受控路径编辑；自动跳过分类器。
- **allowlist**: `readonly` 工具（如 `GlobTool`, `GrepTool`）直接放行。

**Headless 代理** (`shouldAvoidPermissionPrompts`):
- 先执行 `PermissionRequest` hooks → 有决策则采纳
- 无决策 → auto-deny (`AUTO_REJECT_MESSAGE`)

### 3.4 沙箱与安全

**BashTool 沙箱** (`shouldUseSandbox.ts`):
- 启用: `SandboxManager.isSandboxingEnabled()` + `bwrap` 可用
- 自动允许: `sandbox + autoAllowBashIfSandboxed` → ask 规则被 bypass
- 排除命令: `sudo`, `su`, `chmod`, 路径遍历等不进沙箱

**Path Validation**:
- URL-encoded traversal, Unicode normalization, backslash injection 全防护
- 敏感路径保护: `.git/`, `.claude/`, `.vscode/`, 环境变量路径

**Permission Explainer**:
`createPermissionRequestMessage()` 生成用户可读的解释，由 LLM 根据决策原因动态生成。

---

## 4. 上下文压缩 (Context Management & Compaction)

### 4.1 自动压缩触发

**配置**: `src/services/compact/autoCompact.ts`

```typescript
const AUTOCOMPACT_BUFFER_TOKENS = 13_000
const WARNING_THRESHOLD_BUFFER_TOKENS = 20_000
const ERROR_THRESHOLD_BUFFER_TOKENS = 20_000
const MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES = 3  // 失败 3 次后停用

export function getEffectiveContextWindowSize(model: string): number {
  const contextWindow = getContextWindowForModel(model, betas)
  return contextWindow - MAX_OUTPUT_TOKENS_FOR_SUMMARY  // 预留输出空间
}

export function getAutoCompactThreshold(model: string): number {
  return getEffectiveContextWindowSize(model) - AUTOCOMPACT_BUFFER_TOKENS
}
```

**状态追踪** (`autoCompact.ts:AutoCompactTrackingState`):
```typescript
{
  compacted: boolean,
  turnCounter: number,
  turnId: string,  // 每 turn UUID
  consecutiveFailures?: number
}
```

**触发点** (`query.ts` → `maybeCompact()`):
1. 每次模型响应后检测 token 使用量
2. 超过阈值 → `compactConversation()`
3. 失败超过 3 次 → 停用自动压缩，提示手动 `/compact`

### 4.2 压缩策略

**主流程**: `src/services/compact/compact.ts:compactConversation()`

三种策略自动选择：
1. **Microcompact** (区间截断) - 保留最近 N 条，删除旧消息
2. **Session Memory Compact** - 记忆文件摘要化 (`services/sessionMemoryCompact.ts`)
3. **Full Compaction** - 调用 `sideQuery` 生成摘要，插入 "compact boundary" 系统消息

**Hooks** (`executePreCompactHooks`, `executePostCompactHooks`):
- 允许扩展包注入自定义压缩逻辑（如插件清理）

**缓存控制**:
`notifyCompaction()` → 上报 server 的 `promptCacheBreakDetection`，invalidate 缓存键。

---

## 5. 系统提示装配 (System Prompt Assembly)

**设计**: 动态分段 + memoization + 按需 cache-break

**核心** (`src/constants/systemPromptSections.ts`):

```typescript
const section = systemPromptSection(name, compute)           // 稳定段（缓存）
const volatile = DANGEROUS_uncachedSystemPromptSection(name, compute, reason)  // 易变量

async function resolveSystemPromptSections(sections):
  for each section:
    if !cacheBreak && cache.has(name) → 复用
    else → await compute() → setCache(name, value)
  return values
```

**缓存失效**:
- `/clear` / `/compact` → `clearSystemPromptSections()`
- 带 `cacheBreak: true` 的 volatile sections（如当前目录、用户问题）每次 recompute

**段举例** (`src/constants/systemPromptSections.ts` + `src/utils/queryContext.ts`):
- `base`：基础身份与指令
- `permissions`：当前权限规则（可能变化）
- `memory`：记忆文件索引 + 告警相关条目
- `tools`：工具描述（排序后连续前缀以优化缓存）
- `attachments`：MCP 说明、技能模板等
- `ant-internal`：内部构建信息（仅 `USER_TYPE==='ant'`）

---

## 6. 特殊功能模式 (Feature-Gated)

### 6.1 KAIROS (Assistant Mode)

**路径**: `src/assistant/`

**触发器** (`config.ts`):
```typescript
const enabled = feature('KAIROS') && isEnvTruthy(CLAUDE_CODE_KAIROS)
// 三关卡:
// - 24h since last dream (autoDream)
// - >=5 sessions since last dream
// - consolidation lock (防并发)
```

**行为**:
- 持续运行（后台子进程），监听用户活动
- 工具集: `SendUserFileTool`, `PushNotificationTool`, `SubscribePRTool`
- **Brief Mode**：极简洁响应，避免刷屏
- **15s 阻塞预算**: 任何 >15s 操作自动 defer

**共享梦境**: 背景记忆整合 (`services/autoDream/`) 在 KAIROS 模式下由磁盘技能（disk-skill dream）接管。

### 6.2 Coordinator Mode (多代理协调)

**路径**: `src/coordinator/coordinatorMode.ts`

**启用**:
```bash
export CLAUDE_CODE_COORDINATOR_MODE=1
# 或 GrowthBook 控制
```

**架构**:
- **Coordinator** (主 LLM): 使用 `AgentTool` + `Task*` + `SendMessageTool`
- **Workers** (子代理): 受限工具集（`BASH`, `READ`, `EDIT`, `WEB_FETCH` 等，基于 `ASYNC_AGENT_ALLOWED_TOOLS`）
- **通信**: `<task-notification>` XML 消息 + scratchpad 共享目录 (`tengu_scratch`)
- **流程**:
  1. Research (workers 并行探索)
  2. Synthesis (coordinator 汇总)
  3. Implementation (workers 执行)
  4. Verification (workers 测试)

**工具裁剪**:
- `getCoordinatorUserContext()` 注入 worker 可用工具列表到系统提示
- `filterToolsForAgent()` 根据代理类型过滤

### 6.3 Buddy (陪伴宠物)

**路径**: `src/buddy/`

**核心算法** (`companion.ts`):

```typescript
function mulberry32(seed: number): () => number  // PRNG

const SALT = 'friend-2026-401'
function rollCompanion(userId: string): Roll {
  const seed = hashString(userId + SALT) >>> 0
  const rng = mulberry32(seed)

  const rarity = rollRarity(rng)  // 权重: common 60, uncommon 25, rare 10, epic 4, legendary 1
  const shiny: shinyChance = rng() < 0.01  // 1% 闪亮
  const stats = rollStats(rng, rarity)    // peak/dump + 随机
}
```

**物种** (18 种，字符编码隐藏):
```typescript
const SPECIES = [
  duck, goose, blob, cat, dragon, octopus, owl, penguin,
  turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit,
  mushroom, chonk
]
```

**外观**:
- Eyes: `·`, `✦`, `×`, `◉`, `@`, `°`
- Hats: `none`, `crown`, `tophat`, `propeller`, `halo`, `wizard`, `beanie`, `tinyduck`
- 渲染: 5 行 × 12 列 ASCII art，多帧动画

**灵魂**:
首次孵化时，Claude 生成人格描述 → 存储在用户配置，系统提示注入 `companionIntroText()`，用户可直接与 Buddy 对话（Claude 自动退出）。

**注意**: 仅在 `feature('BUDDY')` 启用，存在 `tengu_buddy_launch` gate。

### 6.4 Dream (自省记忆整合)

**路径**: `src/services/autoDream/`

**三关卡** (`autoDream.ts`):
1. **Time gate**: `hoursSinceLastConsolidated >= config.minHours` (default 24)
2. **Session gate**: `numSessionsTouchedSince >= config.minSessions` (default 5)
3. **Lock gate**: `tryAcquireConsolidationLock()` 防并发

**流程** (`consolidationPrompt.ts`):
```
Phase 1 — Orient:
  ls memory/, read ENTRYPOINT_NAME (index), skim topic files

Phase 2 — Gather recent signal:
  - daily logs (logs/YYYY/MM/YYYY-MM-DD.md)
  - drifted memories (contradictions)
  - transcript search (grep narrow)

Phase 3 — Consolidate:
  - 更新/创建记忆文件（遵循 auto-memory 规范）
  - 转换相对日期 → 绝对日期
  - 删除矛盾的事实

Phase 4 — Prune and index:
  - 维护 ENTRYPOINT_NAME 索引（<200 lines, ~25KB）
  - 移除 stale 条目，resolve contradictions
  - 每个索引条目：`- [Title](file.md) — one-line hook`
```

**执行**: 通过 `runForkedAgent()` 启动独立子代理，只读 bash，结果通过 `FileWriteTool` 写回记忆文件。

---

## 7. 命令系统 (Commands)

**入口**: `src/commands.ts`

**分类** (`COMMANDS_ANALYSIS.md`):

| 类别 | 命令示例 | 功能 |
|------|---------|------|
| 会话控制 | resume, clear, exit, rewind, summary | REPL 状态管理 |
| 配置参数 | config, model, effort, fastMode, permission | 运行时设置 |
| 工具管理 | tasks, plugins, skills, mcp | 扩展系统 |
| 调试诊断 | doctor, debug, bug-report | 问题排查 |
| 工程操作 | review, plan, commit | 开发流程 |
| 内部 (ant) | /security-review, tungsten | 内部职能 |

**聚合策略**:
```
内置命令 (commands/) 
+ 技能命令 (skills/loadSkillsDir.ts)
+ 插件命令 (plugins/loadPluginCommands.ts)
+ MCP 命令 (mcp/getMcpToolsCommandsAndResources)
= 最终命令池
```

**筛选**: feature gate + remote mode + 用户类型 (`USER_TYPE==='ant'`)

---

## 8. MCP 与插件生态

**MCP 集成** (`src/services/mcp/`):
- 动态加载远程服务器 (`prefetchOfficialMcpUrls()`)
- 资源发现: `ListMcpResourcesTool`, `ReadMcpResourceTool`
- 工具合并: `assembleToolPool()` 将 MCP tools 与内置 tools 去重合并
- 权限控制: `mcp__serverName` 规则可整服务器 deny/ask

**插件系统** (`src/plugins/`):
- 内置插件: `initBundledPlugins()` (如 `@anthropic-ai/claude-code` 自带)
- 用户安装插件: `loadPluginsFromDir()` 热加载
- 命令/技能/工具注入: 统一聚合进命令池与工具池

**技能系统** (`src/skills/`):
- JSON 描述文件 (`skill.json`) + JS 脚本
- `/skill search` 命令发现安装 (`feature('EXPERIMENTAL_SKILL_SEARCH')`)
- `SkillTool` 负责执行

---

## 9. 统计与遥测

**Analytics** (`src/services/analytics/`):
- GrowthBook 特性门控 + Statsig 事件
- 关键事件: `tengu_auto_mode_decision`, `tengu_coordinator_mode_switched`, `tool_use_summary`
- 元数据标记: `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 防误传文件路径

**成本跟踪** (`cost-tracker.js`):
- 每请求 token 计数 → 金额 (调用 `calculateCostFromTokens`)
- 分类器独立计费 (`classifierCostUSD`) 统计 Auto Mode 开销

**性能分析**:
- `profileCheckpoint` + `headlessProfilerCheckpoint` 标记阶段耗时
- `sessionStatus` 上报至 claude.ai (remote mode)

---

## 10. 安全与加固

| 机制 | 实现文件 | 作用 |
|------|---------|-----|
| 权限模式 | `permissions.ts` | default / auto / bypass / dontAsk |
| 路径遍历防护 | `pathValidation.ts` | URL-encode, Unicode, backslash 归一化 |
| 敏感文件保护 | `filesystem.ts` | `.git/`, `.claude/`, `.vscode/`, shell rc 文件 read-only |
| 沙箱执行 | `shouldUseSandbox.ts` + `SandboxManager` | bwrap + seccomp 限制 |
| 分类器误判恢复 | `DENIAL_LIMITS` + 降级 prompt | 避免 auto-mode 死循环 |
| 权限 hook | `hooks.js` (PermissionRequest) | headless 代理前干预 |
| Undercover Mode | `utils/undercover.ts` (仅内部) | 防止内部信息泄露到开源仓库 |

---

## 11. 未发布功能 (Beta 表)

**`src/constants/betas.ts`**:

```typescript
'interleaved-thinking-2025-05-14'      // extended thinking
'context-1m-2025-08-07'                // 1M token context window
'structured-outputs-2025-12-15'        // JSON schema output
'web-search-2025-03-05'                // Web search (已发布)
'advanced-tool-use-2025-11-20'         // Advanced tool use
'effort-2025-11-24'                    // Effort level (low/medium/high)
'task-budgets-2026-03-13'              // 任务预算管理
'prompt-cache-scope-2026-01-05'        // 缓存作用域
'fast-mode-2026-02-01'                 // Fast mode (已发布 as Penguin)
'redact-thinking-2026-02-12'           // 思考内容脱敏
'token-efficient-tools-2026-03-28'     // 工具 schema 紧凑化
'afk-mode-2026-01-31'                  // away-from-keyboard 模式
'cli-internal-2026-02-09'              // 内部构建
'advisor-tool-2026-03-01'              // 顾问工具
'summarize-connector-text-2026-03-13'  // 连接文本摘要
```

**模型代号** (migrations 文件):
- `Fennec` → Opus 历史迁移
- `Capybara` 新家族: `capybara-v2-fast` (1M context)
- 已发布: `Opus 4.6`, `Sonnet 4.6`, `Sonnet 4.5`, `Sonnet 1m`

---

## 12. 工程化亮点

1. **Dead-code elimination**: Bun `feature()` + `process.env` 常量折叠，未启用 feature 代码完全从生产包移除。
2. **Module isolation**: 循环依赖通过 lazy `require()` 打破（如 `coordinatorModeModule`）。
3. **Type safety**: Zod 输入验证 + TypeScript 严格模式贯穿。
4. **Error classification**: `parsePromptTooLongTokenCounts` 等专用错误类型 + 重试 (`withRetry`)。
5. **Telemetry batching**: `logEvent` 异步写入，避免阻塞主线程。
6. **Config management`: 三层设置（全局/项目/本地）+ CLI 覆盖 + session 临时。

---

## 13. 对 OpenClaw 的设计借鉴

### 13.1 可移植核心

| OpenClaw 需求 | Claude Code 实现 | 移植难度 |
|--------------|----------------|---------|
| 工具池装配 | `getAllBaseTools()` + `getTools(permissionContext)` | ★★ |
| 权限管道 | 8 步检查 + auto mode YOLO 分类器 | ★★★ (分类器可简化) |
| 自动压缩 | `autoCompact` + `compactConversation` (多策略) | ★★ |
| 动态系统提示 | `systemPromptSection` + memoization | ★ |
| 规则持久化 | `permissionsLoader` + `PermissionUpdate` | ★★ |
| 子代理能力隔离 | `AgentTool` + `localDenialTracking` | ★★ |

**简化建议**:
- **去掉 YOLO 分类器** → 使用固定 allowlist/denylist + 手动模式（或集成轻量本地分类器）
- **压缩策略** → 保留 auto + micro，去掉 session-memory 复杂分支
- **MCP** → 暂缓，优先实现本地技能系统

### 13.2 直接增强

1. **Dream 系统**: 四阶段记忆整理，直接可作为 `elite-longterm-memory` 的改进模板。
2. **Buddy gacha**: 确定性生成 + ASCII art 渲染，作为“趣味 Companion”功能移植。
3. **Coordinator 并行**: 借鉴消息协议 (`<task-notification>`) 实现 OpenClaw 多代理协作。
4. **AcceptEdits fast-path**: 在权限模型加入 "fast-track allow" 提升体验。

---

## 14. 文件索引 (关键)

| 路径 | 行数 | 作用 |
|------|------|------|
| `src/main.tsx` | ~1500 | 启动、总装配 |
| `src/tools.ts` | ~500 | 工具池装配 |
| `src/Tool.ts` | ~700 | 工具契约 |
| `src/query.ts` | ~1650 | 主执行内核 |
| `src/utils/permissions/permissions.ts` | ~1000 | 权限管道核心 |
| `src/utils/permissions/yoloClassifier.ts` | ~1400 | 自动分类器 |
| `src/services/compact/compact.ts` | ~1600 | 压缩逻辑 |
| `src/services/autoDream/autoDream.ts` | ~225 | 梦触发器 |
| `src/coordinator/coordinatorMode.ts` | ~270 | 协调模式 |
| `src/buddy/companion.ts` | ~350 | 伴侣生成 |
| `src/constants/systemPromptSections.ts` | ~200 | 系统提示缓存 |
| `SOURCE_INDEX.md` | - | 全量 1902 文件索引（含说明） |
| `COMMANDS_ANALYSIS.md` | - | 命令系统全解 |
| `TOOLS_ANALYSIS.md` | - | 工具系统能力矩阵 |
| `EXECUTION_FLOW.md` | - | 主调用链 diagrams |

---

## 15. 风险与限制

1. **法律边界**: 本分析基于公开泄露的源码，仅用于学术研究；不应分发完整代码。
2. **内部特征**: 大量 `ant`-only 代码在外部构建中被 dead-code eliminated，实际可复用有限。
3. **复杂度**: YOLO 分类器、reactive compact、KAIROS 等子系统和 OpenClaw 当前架构差异大，建议渐进移植。
4. **Rust 重写**: Anthropic 正在将部分核心（如 Buddy）迁移至 Rust（见 `src-rust/`），说明其工程化方向。

---

## 16. 结论

Claude Code 展现了一个**成熟的终端 Agent 平台**应有的深度：  
- **安全是第一位**（permissions 全链路检查 + sandbox + path 验证）
- **性能有优化**（缓存 + timeout + 并行 preload）
- **可扩展性强**（MCP、skills、plugins 三层扩展）
- **用户体验细腻**（fast-path、denial limits、buddy、dream）

对 OpenClaw 的价值：
1. **权限模型**可直接借鉴（8 步管道）
2. **自动压缩**策略值得全盘移植
3. **多代理架构**（coordinator）是未
4. **记忆维护**（dream）强化 elite-longterm-memory

**下一步**: 按模块产出详细技术文档 + 关键代码片段 + 移植方案。

---

报告生成时间: 2026-04-01 14:53 (Asia/Shanghai)  
分析师: 哈基米 🐱 (Agent)  
执行人：junshunG
版本: v0.9 (草案)
