# Claude Code Deep Dive: Buddy 与 Dream 的完整技术实现

> 基于 `claude-code-canxiusi/` 源码的模块级深度分析  
> 专题: Buddy (Tamagotchi Companion) + Dream (Auto Memory Consolidation)  
> 日期: 2026-04-01 | 分析师: 哈基米 🐱 执行人：junshunG

---

## 1. Buddy: 确定性 Gacha 算法与 ASCII 渲染引擎

### 1.1 核心算法 (`src/buddy/companion.ts`)

**设计目标**:
- 同一用户跨会话保持同一物种（基于 `userId` 哈希）
- 可预测的稀有度分布（60/25/10/4/1）
- 极简 PRNG（Mulberry32）避免平台依赖

#### 1.1.1 Mulberry32 PRNG

```typescript
function mulberry32(seed: number): () => number {
  let a = seed >>> 0
  return function () {
    a |= 0
    a = (a + 0x6d2b79f5) | 0
    let t = Math.imul(a ^ (a >>> 15), 1 | a)
    t = (t + Math.imul(t ^ (t >>> 7), 61 | t)) ^ t
    return ((t ^ (t >>> 14)) >>> 0) / 4294967296
  }
}
```

**特点**:
- 单状态 `a`（32 位无符号）
- 常数 `0x6d2b79f5` 为魔术数
- 每次调用产生 `[0, 1)` 均匀分布浮点数
- 周期 `2^32`，足够游戏用途

#### 1.1.2 哈希种子

```typescript
const SALT = 'friend-2026-401'

function hashString(s: string): number {
  if (typeof Bun !== 'undefined') {
    return Number(BigInt(Bun.hash(s)) & 0xffffffffn)  // Fast path: Bun 内置 xxhash
  }
  // Fallback: FNV-1a 32-bit
  let h = 2166136261
  for (let i = 0; i < s.length; i++) {
    h ^= s.charCodeAt(i)
    h = Math.imul(h, 16777619)
  }
  return h >>> 0
}

function rollCompanion(userId: string): Roll {
  const seed = hashString(userId + SALT) >>> 0  // 拼接 SALT 防彩虹表
  const rng = mulberry32(seed)
  // ... rarity/species/stats 由此 rng 顺序抽取
}
```

**关键**: `userId` 一致 → `seed` 一致 → 所有随机结果固定。

#### 1.1.3 稀有度权重系统

```typescript
export const RARITY_WEIGHTS = {
  common: 60,
  uncommon: 25,
  rare: 10,
  epic: 4,
  legendary: 1,
} as const

function rollRarity(rng: () => number): Rarity {
  const total = Object.values(RARITY_WEIGHTS).reduce((a, b) => a + b, 0)  // 100
  let roll = rng() * total
  for (const rarity of RARITIES) {  // [common, uncommon, rare, epic, legendary]
    roll -= RARITY_WEIGHTS[rarity]
    if (roll < 0) return rarity
  }
  return 'common'  // Fallback (理论上不会走到)
}
```

**分布**:
- Common: 60%
- Uncommon: 25%
- Rare: 10%
- Epic: 4%
- Legendary: 1%

#### 1.1.4 属性分配

```typescript
const RARITY_FLOOR: Record<Rarity, number> = {
  common: 5,
  uncommon: 15,
  rare: 25,
  epic: 35,
  legendary: 50,
}

function rollStats(rng: () => number, rarity: Rarity): Record<StatName, number> {
  const floor = RARITY_FLOOR[rarity]
  const peak = pick(rng, STAT_NAMES)  // 随机一项作为峰值属性
  let dump = pick(rng, STAT_NAMES)    // 随机一项作为垃圾属性
  while (dump === peak) dump = pick(rng, STAT_NAMES)

  const stats: Record<StatName, number> = {}
  for (const name of STAT_NAMES) {
    if (name === peak) {
      stats[name] = Math.min(100, floor + 50 + Math.floor(rng() * 30))  // 峰值: floor+50~80
    } else if (name === dump) {
      stats[name] = Math.max(1, floor - 10 + Math.floor(rng() * 15))    // 垃圾: floor-10~4
    } else {
      stats[name] = floor + Math.floor(rng() * 40)                      // 普通: floor~floor+39
    }
  }
  return stats
}
```

**属性** (5 项):
- `DEBUGGING` ( velocidad de debugging )
- `PATIENCE` ( tolerance )
- `CHAOS` ( randomness )
- `WISDOM` ( advice quality )
- `SNARK` ( sarcasm level )

**范围**: 1–100，稀有度提升基础下限。

#### 1.1.5 外观与闪亮

```typescript
const SPECIES = [duck, goose, blob, cat, dragon, octopus, owl, penguin,
                 turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit,
                 mushroom, chonk]  // 18 种，字符编码隐藏

const EYES = ['·', '✦', '×', '◉', '@', '°']
const HATS = ['none', 'crown', 'tophat', 'propeller', 'halo', 'wizard', 'beanie', 'tinyduck']

function rollFrom(rng: () => number): Roll {
  const rarity = rollRarity(rng)
  const bones: CompanionBones = {
    rarity,
    species: pick(rng, SPECIES),
    eye: pick(rng, EYES),
    hat: rarity === 'common' ? 'none' : pick(rng, HATS),  // common 无帽子
    shiny: rng() < 0.01,  // 1% 概率闪亮
    stats: rollStats(rng, rarity),
  }
  // ... 生成 soul 不在 rollFrom 内，首次孵化时由 LLM 填充
}
```

**闪亮**: 1% 概率，全局叠加（不影响稀有度）。

#### 1.1.6 物种隐藏技巧

```typescript
const c = String.fromCharCode
export const duck = c(0x64,0x75,0x63,0x6b) as 'duck'
export const goose = c(0x67, 0x6f, 0x6f, 0x73, 0x65) as 'goose'
// ... 均为 rune 编码，避免源码中明文出现敏感 species name
```

**作用**: `excluded-strings.txt` (构建检查) 不会匹配编译后代码中的物种名。

### 1.2 ASCII 渲染引擎 (`src/buddy/CompanionSprite.tsx`, `sprites.ts`)

#### 1.2.1 精灵帧设计

每物种 3 帧：
- Frame 0: 休息态（静态）
- Frame 1: 微动 (fidget)
- Frame 2: 眨眼 (-1 特殊标记：在帧 0 基础上眨眼)

**渲染函数** (`sprites.ts`):

```typescript
const SPRITE_WIDTH = 12
const SPRITE_HEIGHT = 5

export function renderSprite(
  species: Species,
  frame: number,
  bones: CompanionBones,
): string[] {
  const { eye, hat, shiny } = bones
  const base = BASE_SPRITES[species]  // 5×12 字符矩阵预存每物种休息态
  const eyes = isShiny ? SHINY_EYES : EYES[eye]
  const hatGlyph = HAT_GLYPHS[hat]
  // 逐字符替换 eyes + hat overlay
  // ...
  return lines  // 5 行字符串数组
}
```

**闪烁动画**:
- IDLE_SEQUENCE: `[0,0,0,0,1,0,0,0,-1,0,0,2,0,0,0]` (15 帧)
- 每帧 500ms (`TICK_MS`)
- `-1` 表示在帧 0 基础上绘制眨眼（快速闭眼）

#### 1.2.2 气泡系统

**布局** (`CompanionSprite.tsx`):
```
  Hearts (optional)
  ┌─────────────────────────────────┐
  │  Speech bubble (max 30 chars/line)│
  └─────────────────────────────────┘
  [Sprite Frame]
```

**生命周期**:
- 触发: `t0`（气泡内容创建时刻）
- 显示: `BUBBLE_SHOW = 20` ticks → 10s
- 渐隐: 最后 `FADE_WINDOW = 6` ticks (3s) 逐渐 `dimColor`
- After fade → 移除气泡状态

**Pet 反馈**:
- `/buddy pet` 触发 `PET_BURST_MS = 2500` 的心形飘动动画（5 行预渲染 `PET_HEARTS`）

#### 1.2.3 稀有度着色

```typescript
export const RARITY_COLORS = {
  common: 'inactive',
  uncommon: 'success',
  rare: 'permission',
  epic: 'autoAccept',
  legendary: 'warning',
}
```

气泡边框颜色随稀有度变化。

### 1.3 人格生成 (`buddy/prompt.ts`)

首次孵化（`hatchedAt` 不存在）时：

```typescript
export function getCompanionPrompt(bones: CompanionBones): string {
  return `# Meet ${name} (${species}, ${rarity})

${species} with ${eye} eyes and ${hat} hat.

Stats:
${Object.entries(stats).map(([k,v]) => `- ${k}: ${v}/100`).join('\n')}

Generate:
1. Personality (3 adjectives)
2. Speaking style (tone, quirks)
3. One-line catchphrase`
}
```

Claude 在系统提示中注入此信息，让其即时扮演 companions。

---

## 2. Dream: 自省记忆整合的工程实践

### 2.1 三关卡触发器 (`src/services/autoDream/autoDream.ts`)

#### 2.1.1 时间关卡

```typescript
async function isTimeGateOpen(): boolean {
  const last = await readLastConsolidatedAt()  // 锁文件 mtime
  const hoursElapsed = (Date.now() - last) / (1000 * 60 * 60)
  const cfg = getConfig()  // { minHours: 24, minSessions: 5 }
  return hoursElapsed >= cfg.minHours
}
```

**锁文件**: `.consolidate-lock` 在内存目录中，其 `mtime` 即 `lastConsolidatedAt`。

#### 2.1.2 会话关卡

```typescript
async function isSessionGateOpen(): Promise<boolean> {
  const last = await readLastConsolidatedAt()
  const touched = await listSessionsTouchedSince(last)  // 扫描会话 transcript
  return touched.length >= cfg.minSessions
}
```

`listSessionsTouchedSince()` 排除当前会话，扫描 `sessions/` 目录下 `mtime` 之后的文件。

#### 2.1.3 锁关卡 (防并发)

```typescript
export async function tryAcquireConsolidationLock(): Promise<number | null> {
  const path = lockPath()

  let mtimeMs, holderPid
  try {
    const [s, raw] = await Promise.all([stat(path), readFile(path, 'utf8')])
    mtimeMs = s.mtimeMs
    holderPid = parseInt(raw.trim(), 10)
  } catch {
    // ENOENT → 无锁，继续
  }

  // 检查持有者是否存活 + 是否超时
  if (mtimeMs !== undefined && Date.now() - mtimeMs < HOLDER_STALE_MS) {
    if (holderPid !== undefined && isProcessRunning(holderPid)) {
      return null  // 竞争失败，锁被有效持有
    }
    // 死进程 / PID 重用 → 允许回收
  }

  // 争夺锁：写 PID + mtime = now
  await mkdir(getAutoMemPath(), { recursive: true })
  await writeFile(path, String(process.pid))

  // CAS: 再读一次，确认还是自己 PID
  let verify = await readFile(path, 'utf8')
  if (parseInt(verify.trim(), 10) !== process.pid) return null

  return mtimeMs ?? 0  // 返回旧 mtime 供失败时回滚
}
```

**回滚** (`rollbackConsolidationLock`):
- 成功完成：保留新 `mtime`
- 失败/崩溃：将 `mtime` 回退到 `priorMtime`，清空 PID 体（避免死锁）

#### 2.1.4 总控 (`initAutoDream`)

```typescript
export function initAutoDream(): void {
  if (!isGateOpen()) return  // 仅当 feature + env 开启

  // 每 turn 检查:
  registerPreQueryHook(async () => {
    if (await isGateOpen() && await isTimeGateOpen() && await isSessionGateOpen()) {
      const priorMtime = await tryAcquireConsolidationLock()
      if (priorMtime !== null) {
        try {
          await runDream()  // 子代理执行
        } catch (e) {
          await rollbackConsolidationLock(priorMtime)
        }
      }
    }
  })
}
```

### 2.2 Dream Prompt 工程 (`consolidationPrompt.ts`)

**生成逻辑**:

```typescript
export function buildConsolidationPrompt(
  memoryRoot: string,
  transcriptDir: string,
  extra: string,
): string {
  return `# Dream: Memory Consolidation

You are performing a dream — a reflective pass over your memory files...

Memory directory: \`${memoryRoot}\`
${DIR_EXISTS_GUIDANCE}

Session transcripts: \`${transcriptDir}\`

---

## Phase 1 — Orient
  ls \`memoryRoot\`, read ENTRYPOINT_NAME, skim topics

## Phase 2 — Gather recent signal
  1. daily logs (logs/...)
  2. drifted memories
  3. transcript search (grep narrow)

## Phase 3 — Consolidate
  - 更新/创建记忆文件（遵循 auto-memory 规范）
  - 转换相对日期 → 绝对日期
  - 删除矛盾的事实

## Phase 4 — Prune and index
  - 维护 ENTRYPOINT_NAME（<200 lines, ~25KB）
  - 移除 stale，resolve contradictions

Return a brief summary of what you consolidated...`
}
```

**关键约束**:
- **索引大小**: `~25KB` 或 `200 lines` → 确保 prompt 不膨胀
- **相对日期**: 统一转绝对日期，防止信息随时间失效
- **转录搜索**: `grep -rn "<narrow term>" ${transcriptDir}/ --include="*.jsonl" | tail -50`
- **输出**: 仅 summary，实际修改通过 `FileWriteTool` 静默执行

### 2.3 执行上下文

**调用点** (`autoDream.ts`):

```typescript
async function runDream(): Promise<void> {
  const prompt = buildConsolidationPrompt(
    getAutoMemPath(),
    getTranscriptPath(),  // 会话 JSONL 目录
    '',
  )

  const params = createCacheSafeParams({
    prompt,
    tools: [FileWriteTool, FileEditTool, BashTool],  // 只读 bash (仅搜索 transcript)
    extraContext: getProjectDir(getOriginalCwd()),
  })

  // 启动 forked subagent
  const result = await runForkedAgent(params)

  // 记录完成时间 → 更新锁 mtime
  await recordConsolidation()
}
```

**子进程隔离**:
- `runForkedAgent()` → Node `child_process.fork()` (unix socket 通信)
- `localDenialTracking` 独立 → 不污染主会话
- 工具输入：仅文件编辑 + 只读 bash (用于转录搜索)

---

## 3. 技术细节对比与 OpenClaw 移植建议

### 3.1 Buddy 移植

| 要素 | Claude Code 实现 | OpenClaw 建议 |
|------|----------------|--------------|
| **RNG** | Mulberry32 (单函数) | 复制 `mulberry32` 即可 |
| **物种表** | 18 种 + Unicode 隐藏 | 保留 6 种常用 + 1 种传奇 |
| **稀有度** | 权重 60/25/10/4/1 | 简化: 常见 70%, 稀有 20%, 传奇 10% |
| **属性** | 5 项 (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK) | 可保留，或简化为 3 项: 速度/准确/幽默 |
| **渲染** | 5×12 ASCII + React ink | 直接用 ink 或 term 行绘制 |
| **人格** | LLM 首次生成 | 预置 3 种模板，避免首次 LLM 调用的延迟 |
| **气泡** | 10s 显示 + 渐隐 | 保留，简化至 5s |

**代码移植量**: <500 lines (重渲染层可复用 sprites.ts 的矩阵定义)

### 3.2 Dream 移植

| 要素 | Claude Code 实现 | OpenClaw 建议 |
|------|----------------|--------------|
| **锁** | `.consolidate-lock` (mtime + PID) | 同上，Windows 可用 `fs.promises.open` 独占锁 |
| **阈值** | 24h + 5 sessions | 可配置: 12h + 3 sessions |
| **索引约束** | <200 lines, ~25KB | 硬限制 100 lines |
| **四阶段** | Orient → Gather → Consolidate → Prune | 完全移植 |
| **子进程** | `runForkedAgent` (forked subagent) | 用 `child_process.fork` 或 OpenClaw subagents |
| **搜索** | grep JSONL transcripts | OpenClaw 用 `grep` 命令或内存内倒排索引 |

**关键简化点**:
1. **去掉 `extra` 参数支持**（原版用于内部调试）
2. **固定 prompt 模板**（去掉 `${DIR_EXISTS_GUIDANCE}` 等动态条件）
3. **省略 safety checks** (原版有严格 path 验证)

---

## 4. 关键实现片段 (代码片段库)

### 4.1 Buddy: 完整 rollCompanion 函数

```typescript
// 可直接复制
const SALT = 'your-salt-here'
function hashString(s: string): number { /* FNV-1a 32-bit */ }
function mulberry32(seed: number): () => number { /* as above */ }

export function rollCompanion(userId: string): Companion {
  const seed = hashString(userId + SALT) >>> 0
  const rng = mulberry32(seed)
  const rarity = rollRarity(rng)
  const shiny = rng() < 0.01

  return {
    rarity,
    species: pick(rng, SPECIES),
    eye: pick(rng, EYES),
    hat: rarity === 'common' ? 'none' : pick(rng, HATS),
    shiny,
    stats: rollStats(rng, rarity),
    name: `Buddy-${userId.slice(0, 4)}`,  // 临时，首次 LLM 生成
    personality: '',
    hatchedAt: Date.now(),
  }
}
```

### 4.2 Dream: 锁文件管理

```typescript
const LOCK_FILE = path.join(memoryRoot, '.consolidate-lock')
const STALE_MS = 60 * 60 * 1000  // 1 小时

async function tryLock(): Promise<number | null> {
  try {
    const s = await stat(LOCK_FILE)
    const holder = await readFile(LOCK_FILE, 'utf8')
    if (Date.now() - s.mtimeMs < STALE_MS && isRunning(parseInt(holder))) {
      return null
    }
  } catch {
    // noop
  }

  await writeFile(LOCK_FILE, String(process.pid))
  const verify = await readFile(LOCK_FILE, 'utf8')
  return parseInt(verify) === process.pid ? (s?.mtimeMs ?? 0) : null
}
```

---

## 5. 性能与可靠性观察

### 5.1 Buddy

- **确定性**: 纯函数，无 I/O，毫秒级生成
- **随机性质量**: Mulberry32 非加密级，但对游戏足够
- **渲染**: React ink 组件，变化时才重绘（`React compiler runtime` 标记）

### 5.2 Dream

- **锁竞争**: 多进程安全（mtime + PID + stale timeout）
- **扫描开销**: `listSessionsTouchedSince()` 可能扫描数百 JSONL 文件，建议增量索引（Claude 用 `mtime` 优化）
- **子进程**: 独立内存空间，避免主进程阻塞
- **FAIL 处理**: 失败回滚锁 mtime → 下次触发时间延后 `minHours` (避免死锁)

---

## 6. 未见天日的细节

1. **Buddy 命令**: `/buddy` (hatch), `/buddy pet`, `/buddy info` (display stats)
2. **Dream 手动触发**: `/dream` → 立即记录 `mtime` 并清理
3. **Shiny 特效**: 渲染时眼睛用 `SHINY_EYES` (sparkle 符号)
4. **Hat rarity gating**: common 不能有帽子，其他按稀有度帽子池（8种）
5. **Entrypoint 索引**: Dream 维护的索引文件名为 `MEMORY_INDEX.md` (推测)

---

## 7. 结论

Buddy 与 Dream 代表了 **AI Agent 自省能力** 的两个方向：

- **Buddy**: 个性化、情感化，通过确定性 gacha 建立用户长期依恋
- **Dream**: 系统自省、自我优化，通过定期记忆整理提升长期记忆质量

两者都体现了 **Claude Code 工程化思维**：简单算法（PRNG、锁）、明确约束（索引大小、时间间隔）、优雅降级（lock stale reclaim、denial limits）。

**移植优先级**: Dream > Buddy  
- Dream 直接提升 `elite-longterm-memory` 自动化程度
- Buddy 增加趣味性，但可后续迭代

---
