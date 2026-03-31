# Claude Code 状态管理与数据流架构分析

> 分析日期: 2026-03-31  
> 分析模块: context.ts & context/, state/, schemas/, history.ts, query.ts & query/ & QueryEngine.ts, cost-tracker.ts & costHook.ts

---

## 目录

1. [架构概述](#1-架构概述)
2. [状态管理实现](#2-状态管理实现)
3. [数据流设计](#3-数据流设计)
4. [Schema定义与验证](#4-schema定义与验证)
5. [核心模块详解](#5-核心模块详解)
6. [设计亮点](#6-设计亮点)
7. [关键文件清单](#7-关键文件清单)
8. [总结](#8-总结)

---

## 1. 架构概述

### 1.1 架构定位

Claude Code采用**自定义的本地状态管理体系**，而非Redux或Context API的标准实现。源码中大量使用模块级单例状态、getter/setter风格、以及订阅机制。

### 1.2 状态分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                    全局单例状态层                            │
│              (bootstrap/state.ts)                           │
│   - 会话ID、成本统计、模型使用、用户信息、插件状态           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    UI状态层                                  │
│              (state/AppStateStore.ts)                       │
│   - UI组件状态、工具权限上下文、任务状态、MCP状态            │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    局部状态层                                │
│              (state/store.ts)                               │
│   - 小型独立状态模块 (如compactWarningStore)                 │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 状态管理实现

### 2.1 全局单例状态 (bootstrap/state.ts)

```typescript
// bootstrap/state.ts
// 单例STATE对象，承载几乎所有系统级数据
const STATE: State = getInitialState()

// Getter/Setter风格方法
export function getSessionId(): SessionId {
  return STATE.sessionId
}

export function regenerateSessionId(options?: { setCurrentAsParent?: boolean }): SessionId {
  if (options?.setCurrentAsParent) {
    STATE.parentSessionId = STATE.sessionId
  }
  STATE.sessionId = randomUUID() as SessionId
  return STATE.sessionId
}

export function addToTotalCost(costUSD: number): void {
  STATE.totalCostUSD += costUSD
}

export function getTotalCostUSD(): number {
  return STATE.totalCostUSD
}
```

**State结构**:
```typescript
type State = {
  // 会话信息
  sessionId: SessionId
  parentSessionId: SessionId | undefined
  originalCwd: string
  projectRoot: string
  
  // 成本统计
  totalCostUSD: number
  totalAPIDuration: number
  totalAPIDurationWithoutRetries: number
  totalToolDuration: number
  turnHookDurationMs: number
  turnToolDurationMs: number
  turnClassifierDurationMs: number
  
  // 模型使用
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting
  
  // 遥测
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  locCounter: AttributedCounter | null
  prCounter: AttributedCounter | null
  commitCounter: AttributedCounter | null
  costCounter: AttributedCounter | null
  tokenCounter: AttributedCounter | null
  
  // 插件与扩展
  inlinePlugins: Array<string>
  useCoworkPlugins: boolean
  
  // 会话标志
  isInteractive: boolean
  kairosActive: boolean
  clientType: string
  sessionSource: string | undefined
  
  // ... 更多字段
}
```

### 2.2 UI状态层 (state/AppStateStore.ts)

```typescript
// state/AppStateStore.ts
export type AppState = DeepImmutable<{
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  mainLoopModelForSession: ModelSetting
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  showTeammateMessagePreview?: boolean
  selectedIPAgentIndex: number
  coordinatorTaskIndex: number
  viewSelectionMode: 'none' | 'selecting-agent' | 'viewing-agent'
  footerSelection: FooterItem | null
  toolPermissionContext: ToolPermissionContext
  spinnerTip?: string
  agent: string | undefined
  kairosEnabled: boolean
  remoteSessionUrl: string | undefined
  remoteConnectionStatus: 'connecting' | 'connected' | 'reconnecting' | 'disconnected'
  remoteBackgroundTaskCount: number
  replBridgeEnabled: boolean
  replBridgeExplicit: boolean
  replBridgeOutboundOnly: boolean
  replBridgeConnected: boolean
  replBridgeSessionActive: boolean
  replBridgeReconnecting: boolean
  replBridgeConnectUrl: string | undefined
  replBridgeSessionUrl: string | undefined
  replBridgeEnvironmentId: string | undefined
  replBridgeSessionId: string | undefined
  replBridgeError: string | undefined
  replBridgeInitialName: string | undefined
  showRemoteCallout: boolean
}> & {
  // 非DeepImmutable字段
  tasks: { [taskId: string]: TaskState }
  agentNameRegistry: Map<string, AgentId>
  foregroundedTaskId?: string
  viewingAgentTaskId?: string
  companionReaction?: string
  companionPetAt?: number
  mcp: {
    clients: MCPServerConnection[]
    tools: Tool[]
    commands: Command[]
    resources: Record<string, ServerResource[]>
    pluginReconnectKey: number
  }
  plugins: {
    enabled: LoadedPlugin[]
    disabled: LoadedPlugin[]
    commands: Command[]
    errors: PluginError[]
    installationStatus: {...}
    needsRefresh: boolean
  }
  agentDefinitions: AgentDefinitionsResult
  fileHistory: FileHistoryState
  attribution: AttributionState
  todos: { [agentId: string]: TodoList }
  remoteAgentTaskSuggestions: { summary: string; task: string }[]
  notifications: {
    current: Notification | null
    queue: Notification[]
  }
  elicitation: {
    queue: ElicitationRequestEvent[]
  }
  thinkingEnabled: boolean | undefined
  promptSuggestionEnabled: boolean
  sessionHooks: SessionHooksState
  // ... 更多字段
}
```

### 2.3 局部状态层 (state/store.ts)

```typescript
// state/store.ts - 极简实现 (仅34行)
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 引用相等检查
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

### 2.4 React集成 (state/AppState.tsx)

```typescript
// state/AppState.tsx
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()
  const getState = useCallback(() => selector(store.getState()), [selector, store])
  return useSyncExternalStore(store.subscribe, getState, getState)
}

export function useSetAppState() {
  return useAppStore().setState
}

export function AppStateProvider({ children, initialState, onChangeAppState }: Props) {
  const [store] = useState(() => createStore(initialState ?? getDefaultAppState(), onChangeAppState))
  
  return (
    <AppStoreContext.Provider value={store}>
      {children}
    </AppStoreContext.Provider>
  )
}
```

---

## 3. 数据流设计

### 3.1 核心数据流图

```
用户输入
    │
    ▼
┌─────────────────┐
│   QueryEngine   │ ◄─── 创建/管理对话生命周期
│   .submitMessage│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   query.ts      │ ◄─── 查询循环、状态机
│   queryLoop     │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌────────┐ ┌────────┐
│ Model  │ │ Tools  │ ◄─── 工具调用
│  API   │ │Execution
└───┬────┘ └───┬────┘
    │          │
    ▼          ▼
┌─────────────────┐
│  Message Stream │ ◄─── 流式输出
│  (AsyncGenerator)│
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│   UI/Consumer   │ ◄─── React组件或SDK消费者
└─────────────────┘
```

### 3.2 QueryEngine数据流

```typescript
// QueryEngine.ts
export class QueryEngine {
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean }
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // 1. 构建系统提示词
    const { defaultSystemPrompt, userContext, systemContext } = 
      await fetchSystemPromptParts({...})
    
    // 2. 处理用户输入 (slash命令展开)
    const { messages: messagesFromUserInput, shouldQuery } = 
      await processUserInput({...})
    
    // 3. 更新消息历史
    this.mutableMessages.push(...messagesFromUserInput)
    
    // 4. 调用query进行LLM交互
    for await (const message of query({...})) {
      // 5. 处理流式响应
      switch (message.type) {
        case 'assistant':
          // 处理助手消息
          yield message
          break
        case 'tool_use':
          // 执行工具调用
          const result = await executeTool(message, ...)
          yield result
          break
        case 'error':
          // 处理错误
          yield message
          break
      }
    }
    
    // 6. 返回最终结果
    yield {
      type: 'result',
      subtype: 'success',
      total_cost_usd: getTotalCost(),
      usage: this.totalUsage,
      // ...
    }
  }
}
```

### 3.3 上下文数据流

```typescript
// context.ts
// 系统上下文 - 包含Git状态等
export const getSystemContext = memoize(async (): Promise<{[k: string]: string}> => {
  const gitStatus = await getGitStatus()
  return {
    ...(gitStatus && { gitStatus }),
  }
})

// 用户上下文 - 包含CLAUDE.md等
export const getUserContext = memoize(async (): Promise<{[k: string]: string}> => {
  const claudeMd = await getClaudeMds(...)
  return {
    ...(claudeMd && { claudeMd }),
    currentDate: `Today's date is ${getLocalISODate()}.`,
  }
})
```

**上下文合并流程**:
```
getSystemContext() ──┐
                     ├──► fetchSystemPromptParts() ──► 系统提示词
getUserContext() ────┘
```

### 3.4 成本追踪数据流

```typescript
// cost-tracker.ts
let totalCostUSD = 0
let totalAPIDuration = 0
let modelUsage: { [modelName: string]: ModelUsage } = {}

export function addToTotalCost(costUSD: number): void {
  totalCostUSD += costUSD
  // 触发持久化
  saveCurrentSessionCosts()
}

export function addToTotalAPIDuration(durationMs: number): void {
  totalAPIDuration += durationMs
}

export function updateModelUsage(model: string, usage: ModelUsage): void {
  modelUsage[model] = usage
}

export function getTotalCost(): number {
  return totalCostUSD
}

export function getTotalAPIDuration(): number {
  return totalAPIDuration
}

export function getModelUsage(): { [modelName: string]: ModelUsage } {
  return modelUsage
}
```

### 3.5 历史记录数据流

```typescript
// history.ts
export async function addToHistory(entry: HistoryEntry): Promise<void> {
  const historyFile = getHistoryFilePath()
  const line = JSON.stringify(entry) + '\n'
  await fs.appendFile(historyFile, line)
}

export async function getHistory(limit?: number): Promise<HistoryEntry[]> {
  const historyFile = getHistoryFilePath()
  const content = await fs.readFile(historyFile, 'utf-8')
  const lines = content.trim().split('\n')
  const entries = lines.map(line => JSON.parse(line))
  return limit ? entries.slice(-limit) : entries
}
```

---

## 4. Schema定义与验证

### 4.1 Hooks Schema (schemas/hooks.ts)

```typescript
// schemas/hooks.ts
import { z } from 'zod/v4'

export const HookCommandSchema = z.union([
  z.object({
    type: z.literal('run_script'),
    script: z.string(),
    args: z.array(z.string()).optional(),
  }),
  z.object({
    type: z.literal('send_notification'),
    title: z.string(),
    message: z.string(),
  }),
  // ...
])

export const HookMatcherSchema = z.object({
  event: z.enum(['pre_tool_use', 'post_tool_use', 'session_start', 'session_end']),
  tool_name: z.string().optional(),
  condition: z.string().optional(),
})

export const HookSchema = z.object({
  name: z.string(),
  matcher: HookMatcherSchema,
  commands: z.array(HookCommandSchema),
  enabled: z.boolean().default(true),
})

export const HooksSchema = z.object({
  version: z.literal('1.0'),
  hooks: z.array(HookSchema),
})

export type HookCommand = z.infer<typeof HookCommandSchema>
export type HookMatcher = z.infer<typeof HookMatcherSchema>
export type Hook = z.infer<typeof HookSchema>
export type HooksConfig = z.infer<typeof HooksSchema>
```

### 4.2 验证使用示例

```typescript
// 验证hooks配置
const result = HooksSchema.safeParse(config)
if (!result.success) {
  console.error('Invalid hooks config:', result.error)
  return null
}
return result.data
```

---

## 5. 核心模块详解

### 5.1 QueryEngine.ts

**职责**: 对话生命周期管理、流式输出、token/成本跟踪、工具调用、错误处理

**关键方法**:
```typescript
export class QueryEngine {
  constructor(config: QueryEngineConfig) {
    this.config = config
    this.mutableMessages = config.initialMessages ?? []
    this.abortController = config.abortController ?? createAbortController()
    this.totalUsage = EMPTY_USAGE
  }

  async *submitMessage(prompt: string | ContentBlockParam[], options?: {...}) {
    // 对话核心逻辑
  }

  interrupt(): void {
    this.abortController.abort()
  }

  getMessages(): readonly Message[] {
    return this.mutableMessages
  }
}
```

### 5.2 query.ts

**职责**: 顶层查询流程、循环执行、事件和状态转移

**核心循环**:
```typescript
async function* queryLoop(params: QueryParams, consumedCommandUuids: string[]) {
  const deps = params.deps ?? productionDeps()
  
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    autoCompactTracking: undefined,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    maxOutputTokensOverride: params.maxOutputTokensOverride,
    pendingToolUseSummary: undefined,
    stopHookActive: undefined,
    turnCount: 1,
    transition: undefined,
  }

  while (true) {
    // 1. 应用压缩策略
    // 2. 调用模型API
    // 3. 处理流式响应
    // 4. 执行工具调用
    // 5. 检查终止条件
    // 6. 更新状态并继续或退出
  }
}
```

### 5.3 query/deps.ts

**职责**: 依赖注入，便于测试替换

```typescript
export type QueryDeps = {
  callModel: (params: CallModelParams) => AsyncGenerator<StreamEvent>
  autocompact: AutocompactFn
  microcompact: MicrocompactFn
  uuid: () => string
  // ...
}

export function productionDeps(): QueryDeps {
  return {
    callModel: queryModelWithStreaming,
    autocompact: autoCompact,
    microcompact: microCompact,
    uuid: randomUUID,
    // ...
  }
}
```

### 5.4 cost-tracker.ts

**职责**: 成本与使用量跟踪、持久化

```typescript
export function addToTotalCost(costUSD: number): void {
  const state = getBootstrapState()
  state.totalCostUSD += costUSD
  
  // 持久化到项目配置
  saveCurrentSessionCosts()
}

export function getTotalCost(): number {
  return getBootstrapState().totalCostUSD
}

export function getModelUsage(): { [modelName: string]: ModelUsage } {
  return getBootstrapState().modelUsage
}
```

### 5.5 costHook.ts

**职责**: React/CLI混合场景下的成本汇总钩子

```typescript
export function useCostTracking() {
  useEffect(() => {
    // 组件挂载时开始跟踪
    return () => {
      // 组件卸载时输出成本
      const totalCost = getTotalCost()
      if (totalCost > 0) {
        console.log(`Total cost: $${totalCost.toFixed(4)}`)
        saveCurrentSessionCosts()
      }
    }
  }, [])
}
```

### 5.6 history.ts

**职责**: 历史记录管理、JSONL格式持久化

```typescript
export async function addToHistory(entry: HistoryEntry): Promise<void> {
  const historyFile = getHistoryFilePath()
  const line = JSON.stringify({
    ...entry,
    timestamp: Date.now(),
  }) + '\n'
  await fs.appendFile(historyFile, line)
}

export async function searchHistory(query: string): Promise<HistoryEntry[]> {
  const history = await getHistory()
  return history.filter(entry => 
    entry.content.toLowerCase().includes(query.toLowerCase())
  )
}
```

---

## 6. 设计亮点

### 6.1 分层状态管理

- **全局单例状态**: bootstrap/state.ts提供会话级数据
- **UI状态**: state/AppStateStore.ts提供UI层状态
- **局部状态**: state/store.ts提供小型独立状态

**优势**: 职责清晰，避免单一全局仓库的臃肿

### 6.2 依赖注入与可测试性

```typescript
// QueryDeps接口允许测试时替换
export type QueryDeps = {
  callModel: (params: CallModelParams) => AsyncGenerator<StreamEvent>
  autocompact: AutocompactFn
  // ...
}

// 生产依赖
export function productionDeps(): QueryDeps {...}

// 测试时可以注入mock
const testDeps: QueryDeps = {
  callModel: mockCallModel,
  autocompact: mockAutocompact,
  // ...
}
```

### 6.3 上下文缓存策略

```typescript
// 使用memoize缓存，避免重复计算
export const getSystemContext = memoize(async () => {
  const gitStatus = await getGitStatus()
  return { gitStatus }
})

export const getUserContext = memoize(async () => {
  const claudeMd = await getClaudeMds(...)
  return { claudeMd, currentDate: ... }
})
```

### 6.4 流式数据处理

```typescript
// AsyncGenerator实现流式输出
async function* queryLoop(params: QueryParams) {
  for await (const message of callModel({...})) {
    yield message
    
    if (message.type === 'tool_use') {
      const result = await executeTool(message)
      yield result
    }
  }
}

// 消费者可以实时处理
for await (const message of queryEngine.submitMessage(prompt)) {
  if (message.type === 'assistant') {
    renderToUI(message)
  }
}
```

### 6.5 不可变状态更新

```typescript
// 使用函数式更新
setAppState(prev => ({
  ...prev,
  settings: {
    ...prev.settings,
    theme: 'dark'
  }
}))

// 引用相等检查避免不必要重渲染
if (Object.is(next, prev)) return
```

### 6.6 数据模式验证

```typescript
// Zod schema提供运行时类型安全
const result = HooksSchema.safeParse(config)
if (!result.success) {
  console.error('Invalid config:', result.error.format())
}
```

---

## 7. 关键文件清单

| 文件路径 | 职责 | 关键导出 |
|---------|------|---------|
| `/src/context.ts` | 系统/用户上下文聚合与缓存 | `getSystemContext`, `getUserContext` |
| `/src/utils/context.ts` | 上下文工具函数 | `getContextWindowForModel` |
| `/src/state/store.ts` | 通用Store实现 | `createStore`, `Store<T>` |
| `/src/state/AppStateStore.ts` | AppState定义与默认值 | `AppState`, `getDefaultAppState` |
| `/src/state/selectors.ts` | 派生数据选择器 | 状态选择函数 |
| `/src/state/teammateViewHelpers.ts` | Teammate视图辅助 | 视图切换辅助方法 |
| `/src/history.ts` | 历史记录系统 | `addToHistory`, `getHistory` |
| `/src/QueryEngine.ts` | 查询引擎核心 | `QueryEngine` |
| `/src/query.ts` | 查询流程入口 | `query`, `queryLoop` |
| `/src/query/deps.ts` | 依赖注入 | `QueryDeps`, `productionDeps` |
| `/src/query/config.ts` | 查询配置 | `QueryConfig` |
| `/src/query/stopHooks.ts` | Stop hooks实现 | `handleStopHooks` |
| `/src/schemas/hooks.ts` | Hooks schema定义 | `HooksSchema`, `HookSchema` |
| `/src/cost-tracker.ts` | 成本追踪核心 | `addToTotalCost`, `getTotalCost` |
| `/src/costHook.ts` | 成本钩子 | `useCostTracking` |
| `/src/bootstrap/state.ts` | 全局单例状态 | `STATE`, `getSessionId`, `addToTotalCost` |

---

## 8. 总结

### 8.1 架构特点

1. **自定义状态管理**: 非Redux/Context API，采用分层单例+订阅模式
2. **清晰的分层**: 全局状态、UI状态、局部状态三层分离
3. **依赖注入**: QueryDeps接口提供可测试性
4. **流式处理**: AsyncGenerator实现实时数据流
5. **缓存优化**: memoize缓存昂贵的上下文计算
6. **类型安全**: DeepImmutable+Zod schema提供编译时和运行时类型安全

### 8.2 可学习实践

- **分层状态管理**: 根据数据作用域选择合适的状态层
- **依赖注入**: 通过接口隔离实现可测试性
- **流式API**: 使用AsyncGenerator处理实时数据
- **缓存策略**: 对昂贵计算使用memoize缓存
- **不可变更新**: 函数式状态更新+引用相等检查

### 8.3 与Redux/Context API对比

| 特性 | Claude Code方案 | Redux | Context API |
|-----|----------------|-------|-------------|
| 学习成本 | 低 | 高 | 中 |
| 性能 | 高 (细粒度订阅) | 中 (需selector优化) | 低 (频繁重渲染) |
| 可测试性 | 高 (依赖注入) | 高 | 中 |
| 类型安全 | 高 | 中 | 中 |
| 适用场景 | 中型应用 | 大型应用 | 小型应用 |
| 调试工具 | 自定义日志 | Redux DevTools | React DevTools |

---

*报告完成*
