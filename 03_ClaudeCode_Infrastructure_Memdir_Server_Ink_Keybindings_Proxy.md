# Claude Code 基础设施深度解析: Memdir, Server, Ink, Keybindings, UpstreamProxy, Context

分析师: 哈基米 (Agent)  
执行人: junshunG  
日期: 2026-04-01  
基于: canxiusi/ClaudeCode v2.1.88 (1910 files)

---

## 1. Memdir: 内存索引系统

### 1.1 设计目标

**Memdir** 是 Claude Code 的长期记忆文件系统，模拟 Unix maildir 风格：

```
~/.claude/projects/<project-slug>/memory/
├── MEMORY.md                    # 主索引（最多 200 行 / 25KB）
├── 2026-04-01.md                # 对话日志
├── decisions/my-first-pr.md     # 主题文件，带 frontmatter
├── learnings/react-hooks.md
└── people/john.md
```

### 1.2 核心约束

**MEMORY.md 限制** (`memdir.ts`):
```typescript
export const ENTRYPOINT_NAME = 'MEMORY.md'
export const MAX_ENTRYPOINT_LINES = 200        // ~200 条目
export const MAX_ENTRYPOINT_BYTES = 25_000     // ~25KB
```

**警告触发条件**:
- 超过 200 行 → 截断并追加警告
- 单行过长 → 字节截断到最后一个换行符

**警告信息示例**:
```
> WARNING: MEMORY.md is 350 lines. Only part of it was loaded. Keep index entries to one line under ~200 chars; move detail into topic files.
```

### 1.3 扫描流水线 (`memoryScan.ts`)

```typescript
export async function scanMemoryFiles(
  memoryDir: string,
  signal: AbortSignal,
): Promise<MemoryHeader[]> {
  const entries = await readdir(memoryDir, { recursive: true })
  const mdFiles = entries.filter(f => f.endsWith('.md') && basename(f) !== 'MEMORY.md')

  const headerResults = await Promise.allSettled(
    mdFiles.map(async (relativePath): Promise<MemoryHeader> => {
      const filePath = join(memoryDir, relativePath)
      const { content, mtimeMs } = await readFileInRange(
        filePath, 0, FRONTMATTER_MAX_LINES, undefined, signal,
      )
      const { frontmatter } = parseFrontmatter(content, filePath)
      return {
        filename: relativePath,
        filePath,
        mtimeMs,
        description: frontmatter.description || null,
        type: parseMemoryType(frontmatter.type),
      }
    }),
  )

  return headerResults
    .filter(r => r.status === 'fulfilled')
    .map(r => r.value)
    .sort((a, b) => b.mtimeMs - a.mtimeMs)  // 最新优先
    .slice(0, MAX_MEMORY_FILES)  // 最多 200 个条目
}
```

**性能优化**:
- 只读取前 `FRONTMATTER_MAX_LINES` (30 行) → 避免读大文件
- 并行读取 + `Promise.allSettled` → 错误隔离
- 一次 `readdir` 递归 → 避免多次 stat

### 1.4 清单格式化

```typescript
export function formatMemoryManifest(memories: MemoryHeader[]): string {
  return memories
    .map(m => {
      const tag = m.type ? `[${m.type}] ` : ''
      const ts = new Date(m.mtimeMs).toISOString()
      return m.description
        ? `- ${tag}${m.filename} (${ts}): ${m.description}`
        : `- ${tag}${m.filename} (${ts})`
    })
    .join('\n')
}
```

**用途**:
- Recall selector prompt (自动选择相关记忆)
- Extraction agent prompt (预注入清单，节省 `ls` 调用)

### 1.5 预创建目录优化

```typescript
export async function ensureMemoryDirExists(memoryDir: string): Promise<void> {
  const fs = getFsImplementation()
  try {
    await fs.mkdir(memoryDir)  // 递归创建，EEXIST 内部处理
  } catch (e) {
    // 真实错误才记录 (EACCES/EROFS)
    logForDebugging(`ensureMemoryDirExists failed: ${e.code}`, { level: 'debug' })
  }
}
```

**效益**: 模型可直接 `Write` 而无需先 `mkdir` (prompt 中有 `DIR_EXISTS_GUIDANCE`)。

### 1.6 记忆类型

**Type 字段** (`memoryTypes.ts`):
```typescript
export type MemoryType =
  | 'decision'
  | 'person'
  | 'project'
  | 'event'
  | 'document'
  | 'conversation'
  | 'learning'
  | 'preference'

export function parseMemoryType(typeStr: string | undefined): MemoryType | undefined {
  // 规范化：'DecisiOn' → 'decision'
}
```

**语义分组**:
- `decision`: 技术决策、PR 合并、架构选择
- `person`: 人物画像、联系人、角色
- `project`: 项目进展、里程碑
- `event`: 会议、计划、约见
- `document`: 文件、文档、笔记
- `conversation`: 对话摘要
- `learning`: 教训、新知
- `preference`: 喜好、设置

---

## 2. Server & REPL: 本地服务器架构

### 2.1 直接连接会话 (`createDirectConnectSession.ts`)

**通信协议**:
```
POST /sessions
Request: { cwd, dangerously_skip_permissions? }
Response: {
  session_id: string
  ws_url: string  // WebSocket endpoint
  work_dir?: string  // 可选的工作区路径
}
```

**错误封装**:
```typescript
export class DirectConnectError extends Error {
  constructor(message: string) {
    super(message)
    this.name = 'DirectConnectError'
  }
}
```

**所有失败路径**:
- 网络错误 (`fetch` 抛异常) → 包装 `DirectConnectError`
- HTTP 非 2xx → `DirectConnectError(status statusText)`
- 响应解析失败 (`connectResponseSchema` 验证) → `DirectConnectError(Invalid session response)`

**Schema 验证**:
```typescript
// types.ts
export const connectResponseSchema = object({
  session_id: string(),
  ws_url: string(),
  work_dir: optional(string()),
})
```

### 2.2 REPL 会话管理

**入口** (`src/screens/REPL.tsx`):
- 初始化 `DirectConnectConfig` 或 `stdio` 模式
- 创建 `QueryEngine` 实例
- 主循环: `await queryEngine.query(userInput)`

**主循环伪码**:
```typescript
while (true) {
  const input = await readUserInput()
  if (input === '/exit') break

  const response = await queryEngine.query(input)
  printAssistantMessage(response)
}
```

**会话恢复** (`ResumeConversation.tsx`):
- 从 `conversations/` 目录加载历史
- 重建 `messages[]` 数组
- 匹配 `coordinatorMode` 和环境变量
- 恢复 Task 状态 (`tasks` map)

---

## 3. Ink: 底层渲染引擎

### 3.1 架构

**Ink** 是 React 的终端渲染器，Claude Code 用它实现 TUI：

```typescript
import { render, Box, Text } from 'ink'

render(
  <Box flexDirection="column">
    <Text color="green">Hello Terminal</Text>
  </Box>
)
```

**渲染循环**:
- `FRAME_INTERVAL_MS = 16` (~60fps)
- React Reconciler 将 element tree 转换为 ANSI escape sequences
- `process.stdout.write()` 输出

### 3.2 核心组件

**Box** (`Box.tsx`):
- Flex 布局 (`flexDirection: 'row' | 'column'`)
- 内边距、边距
- 跨终端兼容的宽度计算

**Text** (`Text.tsx`):
- 颜色: `color`, `backgroundColor`
- 样式: `bold`, `italic`, `underline`, `strikethrough`
- 修饰: `inverse`

**静态导入优化**:
```typescript
import { colorScheme } from './colorScheme.js'  // 预计算 256/真彩色映射
```

### 3.3 ANSI 转义处理

```typescript
// Ansi.tsx
export const ANSI_ESC = '\x1b['
export const ANSI_RESET = '\x1b[0m'

export function ansiColorCode(color: AnsiColor): string {
  // 256 色: 38;5;<n>
  // 真彩色: 38;2;<r>;<g>;<b>
}
```

**终端检测**:
- `process.env.COLORTERM` (truecolor)
- `process.env.TERM` (xterm-256color, screen, etc.)

---

## 4. Keybindings: 快捷键系统

### 4.1 绑定结构

**KeybindingBlock**:
```typescript
type KeybindingBlock = {
  context: 'Global' | 'Chat' | 'Autocomplete' | 'Task' | ...
  bindings: Record<string, CommandName>
}
```

**上下文优先级**:
- Global → 总是生效
- Chat → 聊天模式有效
- Autocomplete → 自动完成弹窗时有效
- 更具体上下文覆盖通用

### 4.2 默认绑定 (`defaultBindings.ts`)

**平台适配**:

```typescript
const IMAGE_PASTE_KEY = getPlatform() === 'windows' ? 'alt+v' : 'ctrl+v'
const MODE_CYCLE_KEY = SUPPORTS_TERMINAL_VT_MODE ? 'shift+tab' : 'meta+m'
```

**VT 模式检测**:
```typescript
const SUPPORTS_TERMINAL_VT_MODE =
  getPlatform() !== 'windows' ||
  (isRunningWithBun() ? satisfies(bunVersion, '>=1.2.23')
                      : satisfies(nodeVersion, '>=22.17.0 <23.0.0 || >=24.2.0'))
```

**Ctrl+C/D 特殊处理**:
- 保留但 **用户不可重绑定** (`reservedShortcuts.ts` 校验)
- `ctrl+c`: interrupt (时间窗口检测是否双击)
- `ctrl+d`: exit (连续两次才退出)

**Feature-flagged 绑定**:
```typescript
...(feature('KAIROS') || feature('KAIROS_BRIEF')
  ? { 'ctrl+shift+b': 'app:toggleBrief' as const }
  : {}),
...(feature('QUICK_SEARCH')
  ? { 'ctrl+shift+f': 'app:globalSearch' as const }
  : {}),
...(feature('TERMINAL_PANEL') ? { 'meta+j': 'app:toggleTerminal' } : {}),
```

### 4.3 绑定加载顺序

1. **内置默认** (`DEFAULT_BINDINGS`)
2. **用户配置** (`~/.claude/keybindings.json`) → 覆盖默认
3. **验证**: `reservedShortcuts.ts` 检查是否尝试重绑保留键 → 报错

### 4.4 绑定解析

**匹配算法** (`match.ts`):
```typescript
export function matchKeybinding(
  pressed: string,  // e.g. "ctrl+shift+k"
  blocks: KeybindingBlock[],
  currentContext: ContextName,
): CommandName | undefined {
  // 1. 收集当前上下文所有绑定的 block
  // 2. 精确匹配 → 命中则返回
  // 3. chord 前缀匹配（如 "ctrl+x" 绑定，用户按 "ctrl+x ctrl+k"）
  // 4. 若 chord 前缀匹配但后续不完整 → 进入等待状态
}
```

**Chord 支持**:
- `ctrl+x ctrl+k`: 先按 `ctrl+x`，再按 `ctrl+k` 触发
- 超时: `CHORD_TIMEOUT_MS = 2000` → 两秒内未完成则取消

---

## 5. Query Config: 运行时门控快照

### 5.1 设计动机

`feature()` 调用是 **tree-shaking 边界**，必须 inline 才能正确 dead-code elimination。

但某些运行时门控（ Statsig + Env ）需要 **一次性快照** 以避免每次工具调用都查 GB。

### 5.2 结构

```typescript
export type QueryConfig = {
  sessionId: SessionId
  gates: {
    streamingToolExecution: boolean
    emitToolUseSummaries: boolean
    isAnt: boolean
    fastModeEnabled: boolean
  }
}
```

**创建时机**: `query()` 入口一次生成，传给整个递归调用栈。

### 5.3 配置项说明

| 配置 | 来源 | 用途 |
|------|------|------|
| `streamingToolExecution` | `tengu_streaming_tool_execution2` (Statsig) | 工具结果增量显示 |
| `emitToolUseSummaries` | `CLAUDE_CODE_EMIT_TOOL_USE_SUMMARIES` (env) | 日志中输出工具摘要 |
| `isAnt` | `USER_TYPE === 'ant'` (env) | 内部测试功能开关 |
| `fastModeEnabled` | `!CLAUDE_CODE_DISABLE_FAST_MODE` (env) | 快速模式（少输出） |

**内联判断示例**:
```typescript
// fastMode.ts
export function isFastModeEnabled(): boolean {
  // 直接内联，避免 pull 整个模块到不相关的 test 包中
  return !isEnvTruthy(process.env.CLAUDE_CODE_DISABLE_FAST_MODE)
}
```

---

## 6. UpstreamProxy: CCR 容器代理桥

### 6.1 设计背景

**CCR** (Claude Code Remote) 在容器中运行，需要通过 upstreamproxy 将流量路由到远程服务器，同时拦截内网请求。

### 6.2 初始化流程 (`initUpstreamProxy`)

**触发条件**:
1. `CLAUDE_CODE_REMOTE` 存在
2. `CCR_UPSTREAM_PROXY_ENABLED` truthy (服务器端 GrowthBook 注入)

**步骤**:
1. **读取 session token** (`/run/ccr/session_token`)
2. **禁用 heap dump** (`prctl(PR_SET_DUMPABLE, 0)`) → 防止同 UID ptrace
3. **CA 证书拼接**:
   - 下载上游代理的 CA
   - 追加到系统 bundle `/etc/ssl/certs/ca-certificates.crt`
4. **启动本地中继** (`startUpstreamProxyRelay()`):
   - 监听 `localhost:PORT`
   - 转发到 upstreamproxy
5. **删除 token 文件** (`unlink`) → token 仅驻留内存
6. **设置环境变量**:
   - `HTTPS_PROXY=http://localhost:PORT`
   - `SSL_CERT_FILE=/path/to/custom-ca-bundle.crt`

### 6.3 代理排除列表 (`NO_PROXY_LIST`)

**目的**: 信任的域名不走 MITM 代理

**包含**:
- 本地回环: `localhost`, `127.0.0.1`, `::1`
- 私有网络: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `169.254.0.0/16` (IMDS)
- Anthropic API 三形式: `anthropic.com`, `.anthropic.com`, `*.anthropic.com`
- GitHub/GitHubusercontent
- NPM/PyPI/Crates 包注册表（容器可直接访问）

**格式**: 逗号分隔字符串，跨运行时兼容（Bun/Node/ Go/ Python）。

### 6.4 安全假设

- **Token 仅内存**: 删除文件后，子进程继承环境变量但无法读文件
- **MITM 仅对出站 HTTPS**: 内网流量直连
- **容错**: 任何步骤失败 → `state.enabled = false` 并 warn，不断会话

---

## 7. Context: 应用上下文管理

### 7.1 模块概览

- `mailbox.tsx`: 异步任务消息队列（类似浏览器 event loop）
- `notifications.tsx`: 系统通知（Toast）
- `modalContext.tsx`: 模态框堆栈管理
- `overlayContext.tsx`: 覆盖层（如小键盘、帮助）
- `fpsMetrics.tsx`: 性能监控（帧率）

### 7.2 Mailbox 模式

**设计**:
```typescript
interface Mailbox {
  pendingUserMessages: UserMessage[]  // 来自用户交互
  backgroundTasks: Task[]             // 后台编译、索引
  systemNotifications: Notification[] // 错误、状态
}
```

**调度**: 每帧清空 `pendingUserMessages`，推送至 `queryEngine`

**用途**: 解耦 UI 输入与查询循环。

### 7.3 Notification 系统

**类型**:
- `error`: 错误通知（红色，可 dismiss）
- `info`: 信息（蓝色，自动消失）
- `success`: 成功（绿色）

**位置**: 右上角堆叠，5 秒后自动移除（可配置）

---

## 8. 其他亮点

### 8.1 Token Budget 管理 (`query/tokenBudget.ts`)

**算法**:
```typescript
const MAX_INPUT_TOKENS = 200_000
const MAX_OUTPUT_TOKENS = 8_000

function allocateTokenBudget(
  toolResults: ToolResult[],  // 需要压缩的历史
  systemPrompt: string,
): { messages: MessageParam[], remainingOutput: number } {
  // 二分搜索找到最多的历史组数，使得:
  //   sum(tokens(messages)) + tokens(system) <= MAX_INPUT_TOKENS
  // 剩余 tokens 给输出
}
```

### 8.2 Stop Hooks (`query/stopHooks.ts`)

**注册**:
```typescript
let stopHooks: StopHook[] = []

export function registerStopHook(hook: StopHook): void {
  stopHooks.push(hook)
}
```

**触发**: 用户中断 (Ctrl+C) 或 `TaskStopTool`
```typescript
export async function runStopHooks(reason: StopReason): Promise<void> {
  for (const hook of stopHooks) {
    await hook(reason)  // 允许异步清理
  }
}
```

**用例**:
- Dream 任务写入锁文件回滚
- Agent 子进程 kill
- 临时文件删除

### 8.3 Migrations 系统

**文件**: `src/migrations/`

**执行时机**: 启动时检查 `~/.claude/state/migrations/applied.json`

**示例**:
```typescript
export async function migrateFennecToOpus(): Promise<void> {
  // FENNEC_MODEL 环境变量重命名为 OPUS_MODEL
  // 旧值自动映射
}
```

**幂等**: 每个 migration 写入 hash，重复运行无副作用

---

## 9. OpenClaw 移植技术清单 (续)

| 模块 | 核心文件 | 难度 | 优先级 | 移植建议 |
|------|---------|------|--------|----------|
| Memdir 索引 | `memdir/memdir.ts`, `scan.ts` | ★★ | P0 | 必须实现，长期记忆基础 |
| Memdir 类型 | `memdir/memoryTypes.ts` | ★ | P0 | 8 种类型即可 |
| Server Direct Connect | `server/createDirectConnectSession.ts` | ★★ | P1 | 用于远程模式桥接 |
| REPL 主循环 | `screens/REPL.tsx` | ★★ | P0 | 聊天主流程核心 |
| Ink 布局 | `ink/Box.tsx`, `Text.tsx` | ★★ | P1 | 已有 TUI 框架可借鉴 |
| ANSI 渲染 | `ink/constants.ts`, `Ansi.tsx` | ★ | P1 | 256/真彩色映射 |
| Keybindings 系统 | `keybindings/` | ★★ | P0 | 快捷键必备 |
| 默认绑定 | `defaultBindings.ts` | ★ | P0 | 直接抄作业 |
| 绑定匹配 | `match.ts` | ★★ | P1 | chord 前缀算法 |
| Query Config | `query/config.ts` | ★ | P1 | 运行时门控快照 |
| UpstreamProxy | `upstreamproxy/` | ★★★ | P2 | CCR 容器特有，可选 |
| Context 系统 | `context/*` | ★★ | P1 | 消息队列 + 通知 |
| Token Budget | `query/tokenBudget.ts` | ★★ | P1 | 二分搜索分配 |
| Stop Hooks | `query/stopHooks.ts` | ★ | P1 | 中断清理支持 |
| Migrations | `migrations/*` | ★ | P2 | 版本迁移框架 |

---

## 10. 关键入口速查 (续)

| 功能 | 文件 | 函数 |
|------|------|------|
| Memdir 扫描 | `memdir/memoryScan.ts` | `scanMemoryFiles()` |
| 清单格式化 | `memdir/memoryScan.ts` | `formatMemoryManifest()` |
| 目录预创建 | `memdir/memdir.ts` | `ensureMemoryDirExists()` |
| 截断索引 | `memdir/memdir.ts` | `truncateEntrypointContent()` |
| 会话创建 | `server/createDirectConnectSession.ts` | `createDirectConnectSession()` |
| 错误类 | `server/createDirectConnectSession.ts` | `DirectConnectError` |
| REPL 屏幕 | `screens/REPL.tsx` | `REPL()` component |
| 会话恢复 | `screens/ResumeConversation.tsx` | `ResumeConversation()` |
| Box 组件 | `ink/Box.tsx` | `<Box>` |
| Text 组件 | `ink/Text.tsx` | `<Text>` |
| 颜色映射 | `ink/constants.ts` | `ansiColorCode()` |
| 默认绑定 | `keybindings/defaultBindings.ts` | `DEFAULT_BINDINGS` |
| 绑定匹配 | `keybindings/match.ts` | `matchKeybinding()` |
| 用户绑定加载 | `keybindings/loadUserBindings.ts` | `loadUserBindings()` |
| Query 配置 | `query/config.ts` | `buildQueryConfig()` |
| Token 预算 | `query/tokenBudget.ts` | `allocateTokenBudget()` |
| 停止钩子 | `query/stopHooks.ts` | `registerStopHook()`, `runStopHooks()` |
| 代理初始化 | `upstreamproxy/upstreamproxy.ts` | `initUpstreamProxy()` |
| 中继启动 | `upstreamproxy/relay.ts` | `startUpstreamProxyRelay()` |
| 通知上下文 | `context/notifications.tsx` | `useNotifications()` |
| 邮箱上下文 | `context/mailbox.tsx` | `useMailbox()` |

---

## 11. 设计与实现模式总结

| 模式 | 应用场景 | 示例 |
|-----|---------|------|
| **Config Snapshot** | 避免运行时门控散落各处 | `QueryConfig` |
| **Truncate with Warning** | 保护入口文件过大 | `truncateEntrypointContent` |
| **Two-Phase Init** | 容错依赖（代理、MCP） | `initUpstreamProxy` 失败 open |
| **Mailbox Queue** | 解耦输入/输出循环 | `mailbox.tsx` |
| **Stop Hooks** | 中断清理扩展点 | `registerStopHook` |
| **Migration Registry** | 配置 schema 演进 | `migrations/` |
| **Context + Provider** | React 全局状态 | `KeybindingProviderSetup` |
| **Feature Flag Granularity** | 死代码消除边界 | `feature()` inline |
| **Compiled Hook IDs** | React Forget 生成 | `_c(n)` `$[]` |
| **Lazy Schema** | 打破循环依赖 | `lazySchema(() => ...)` |

---

报告完成于 2026-04-01 17:40 (Asia/Shanghai)  
总阅读量: 20+ 核心文件，约 3500 行代码  
状态: 准备发布
