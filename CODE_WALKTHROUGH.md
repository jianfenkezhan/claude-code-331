# Claude Code 关键代码走读

> 分析日期: 2026-03-31  
> 目标: 深入理解关键功能的实现细节

---

## 目录

1. [Tool系统完整走读](#1-tool系统完整走读)
2. [QueryEngine完整走读](#2-queryengine完整走读)
3. [状态管理完整走读](#3-状态管理完整走读)
4. [启动流程完整走读](#4-启动流程完整走读)
5. [权限系统完整走读](#5-权限系统完整走读)
6. [总结](#6-总结)

---

## 1. Tool系统完整走读

### 1.1 Tool类型定义详解

```typescript
// src/Tool.ts

// 1. 工具基础类型定义
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 基础信息
  name: string
  aliases?: string[]
  searchHint?: string  // 用于ToolSearch的关键词
  
  // 核心执行方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  
  // 描述与提示词生成
  description(
    input: z.infer<Input>,
    options: {
      isNonInteractiveSession: boolean
      toolPermissionContext: ToolPermissionContext
      tools: Tools
    },
  ): Promise<string>
  
  prompt(options: {
    getToolPermissionContext: () => Promise<ToolPermissionContext>
    tools: Tools
    agents: AgentDefinition[]
    allowedAgentTypes?: string[]
  }): Promise<string>
  
  // Schema定义
  readonly inputSchema: Input
  readonly inputJSONSchema?: ToolInputJSONSchema
  outputSchema?: z.ZodType<unknown>
  
  // 状态检查
  isEnabled(): boolean
  isReadOnly(input: z.infer<Input>): boolean
  isDestructive?(input: z.infer<Input>): boolean
  isConcurrencySafe(input: z.infer<Input>): boolean
  
  // 权限检查
  checkPermissions(
    input: z.infer<Input>,
    context: ToolUseContext,
  ): Promise<PermissionResult>
  
  // 渲染方法（UI层）
  renderToolUseMessage(
    input: Partial<z.infer<Input>>,
    options: { theme: ThemeName; verbose: boolean; commands?: Command[] },
  ): React.ReactNode
  
  renderToolResultMessage?(
    content: Output,
    progressMessagesForMessage: ProgressMessage<P>[],
    options: {
      style?: 'condensed'
      theme: ThemeName
      tools: Tools
      verbose: boolean
      isTranscriptMode?: boolean
      isBriefOnly?: boolean
      input?: unknown
    },
  ): React.ReactNode
  
  renderToolUseProgressMessage?(
    progressMessagesForMessage: ProgressMessage<P>[],
    options: {...},
  ): React.ReactNode
  
  // 工具结果转换
  mapToolResultToToolResultBlockParam(
    content: Output,
    toolUseID: string,
  ): ToolResultBlockParam
  
  // 自动分类器输入
  toAutoClassifierInput(input: z.infer<Input>): unknown
  
  // 其他元数据
  maxResultSizeChars: number
  interruptBehavior?(): 'cancel' | 'block'
  isSearchOrReadCommand?(input: z.infer<Input>): {...}
  shouldDefer?: boolean
  alwaysLoad?: boolean
  strict?: boolean
}
```

### 1.2 buildTool工厂函数详解

```typescript
// src/Tool.ts

// 1. 定义可默认化的方法键
 type DefaultableToolKeys =
  | 'isEnabled'
  | 'isConcurrencySafe'
  | 'isReadOnly'
  | 'isDestructive'
  | 'checkPermissions'
  | 'toAutoClassifierInput'
  | 'userFacingName'

// 2. 定义默认值对象
const TOOL_DEFAULTS = {
  // 默认启用
  isEnabled: () => true,
  
  // 默认不安全并发（保守策略）
  isConcurrencySafe: (_input?: unknown) => false,
  
  // 默认非只读
  isReadOnly: (_input?: unknown) => false,
  
  // 默认非破坏性
  isDestructive: (_input?: unknown) => false,
  
  // 默认允许（权限系统会进一步检查）
  checkPermissions: (
    input: { [key: string]: unknown },
    _ctx?: ToolUseContext,
  ): Promise<PermissionResult> =>
    Promise.resolve({ behavior: 'allow', updatedInput: input }),
  
  // 默认跳过分类器
  toAutoClassifierInput: (_input?: unknown) => '',
  
  // 默认使用工具名
  userFacingName: (_input?: unknown) => '',
}

// 3. buildTool工厂函数
export function buildTool<D extends AnyToolDef>(def: D): BuiltTool<D> {
  return {
    ...TOOL_DEFAULTS,                    // 1. 展开默认值
    userFacingName: () => def.name,      // 2. 设置默认userFacingName
    ...def,                              // 3. 展开自定义定义（覆盖默认值）
  } as BuiltTool<D>
}
```

**设计亮点**:
- 使用对象展开运算符实现默认值的优雅合并
- 类型级别的保证：BuiltTool<D>确保返回完整的Tool类型
- 开发者只需覆盖需要自定义的方法

### 1.3 BashTool完整实现走读

```typescript
// src/tools/BashTool/BashTool.ts

// 1. 输入Schema定义
const BashToolInputSchema = z.object({
  command: z.string().describe('The bash command to execute'),
  timeout: z.number().optional().describe('Timeout in milliseconds'),
  workdir: z.string().optional().describe('Working directory'),
})

// 2. 输出类型定义
type BashOutput = {
  stdout: string
  stderr: string
  exitCode: number
}

// 3. 使用buildTool创建工具
export const BashTool = buildTool({
  // 基础信息
  name: 'bash',
  maxResultSizeChars: 100000,
  
  // Schema绑定
  inputSchema: BashToolInputSchema,
  
  // 描述生成（动态，基于输入）
  async description(input) {
    return `Execute bash command: ${input.command}`
  },
  
  // 提示词生成（用于系统提示词）
  async prompt(options) {
    return `## Bash Tool
Execute bash commands in the user's environment.
Available commands: ls, cat, grep, find, etc.
Current directory: ${process.cwd()}`
  },
  
  // 用户显示名称
  userFacingName(input) {
    return input?.command?.split(' ')[0] || 'bash'
  },
  
  // 活动描述（用于加载指示器）
  getActivityDescription(input) {
    return `Running ${input?.command}`
  },
  
  // 权限检查
  async checkPermissions(input, context) {
    // 1. 检查是否允许执行bash
    const { mode } = context.getAppState().toolPermissionContext
    
    if (mode === 'bypassPermissions') {
      return { behavior: 'allow', updatedInput: input }
    }
    
    // 2. 检查命令是否在允许列表
    const allowedCommands = ['ls', 'cat', 'grep', 'find', 'pwd', 'echo']
    const command = input.command.trim().split(' ')[0]
    
    if (!allowedCommands.includes(command)) {
      return {
        behavior: 'ask',
        updatedInput: input,
        message: `Command "${command}" may be dangerous. Allow?`,
      }
    }
    
    return { behavior: 'allow', updatedInput: input }
  },
  
  // 核心执行逻辑
  async call(args, context, canUseTool, parentMessage, onProgress) {
    // 1. 权限检查
    const permission = await canUseTool(
      this,
      args,
      context,
      parentMessage,
      generateToolUseId(),
    )
    
    if (permission.behavior !== 'allow') {
      return {
        data: {
          stdout: '',
          stderr: 'Permission denied',
          exitCode: 1,
        },
      }
    }
    
    // 2. 执行命令
    const { command, timeout = 60000, workdir } = args
    
    onProgress?.({
      toolUseID: parentMessage.toolUseId,
      data: {
        type: 'bash_progress',
        status: 'running',
        command,
      },
    })
    
    try {
      const result = await execAsync(command, {
        cwd: workdir || context.getAppState().cwd,
        timeout,
        env: process.env,
      })
      
      onProgress?.({
        toolUseID: parentMessage.toolUseId,
        data: {
          type: 'bash_progress',
          status: 'completed',
          exitCode: 0,
        },
      })
      
      return {
        data: {
          stdout: result.stdout,
          stderr: result.stderr,
          exitCode: 0,
        },
      }
    } catch (error) {
      onProgress?.({
        toolUseID: parentMessage.toolUseId,
        data: {
          type: 'bash_progress',
          status: 'error',
          error: error.message,
        },
      })
      
      return {
        data: {
          stdout: '',
          stderr: error.message,
          exitCode: error.code || 1,
        },
      }
    }
  },
  
  // 渲染工具使用消息
  renderToolUseMessage(input, options) {
    return (
      <Box>
        <Text color="cyan">$</Text> <Text>{input.command}</Text>
      </Box>
    )
  },
  
  // 渲染工具结果
  renderToolResultMessage(content, progressMessages, options) {
    if (options.isBriefOnly) {
      return <Text dimColor>Exit code: {content.exitCode}</Text>
    }
    
    return (
      <Box>
        {content.stdout && <Text>{content.stdout}</Text>}
        {content.stderr && <Text color="red">{content.stderr}</Text>}
      </Box>
    )
  },
  
  // 渲染进度
  renderToolUseProgressMessage(progressMessages) {
    const latest = progressMessages[progressMessages.length - 1]
    if (latest?.data?.type === 'bash_progress') {
      return <Spinner text={latest.data.status} />
    }
    return null
  },
  
  // 转换结果为API格式
  mapToolResultToToolResultBlockParam(content, toolUseID) {
    return {
      type: 'tool_result',
      tool_use_id: toolUseID,
      content: [
        {
          type: 'text',
          text: content.stdout || content.stderr || '(no output)',
        },
      ],
      is_error: content.exitCode !== 0,
    }
  },
  
  // 自动分类器输入
  toAutoClassifierInput(input) {
    return input.command
  },
  
  // 并发安全：bash命令可以并发执行
  isConcurrencySafe() {
    return true
  },
  
  // 只读检查：bash可能修改文件系统
  isReadOnly(input) {
    const readOnlyCommands = ['ls', 'cat', 'pwd', 'echo', 'grep', 'find']
    const command = input.command.trim().split(' ')[0]
    return readOnlyCommands.includes(command)
  },
})
```

### 1.4 工具注册与使用流程

```typescript
// src/tools.ts

// 1. 获取所有基础工具
export function getAllBaseTools(): Tools {
  return [
    AgentTool,
    TaskOutputTool,
    BashTool,
    // 条件工具
    ...(hasEmbeddedSearchTools() ? [] : [GlobTool, GrepTool]),
    ExitPlanModeV2Tool,
    FileReadTool,
    FileEditTool,
    FileWriteTool,
    NotebookEditTool,
    WebFetchTool,
    TodoWriteTool,
    WebSearchTool,
    TaskStopTool,
    AskUserQuestionTool,
    SkillTool,
    EnterPlanModeTool,
    // 特性标志控制的工具
    ...(process.env.USER_TYPE === 'ant' ? [ConfigTool, TungstenTool] : []),
    ...(SuggestBackgroundPRTool ? [SuggestBackgroundPRTool] : []),
    ...(WebBrowserTool ? [WebBrowserTool] : []),
    ...(isTodoV2Enabled() ? [TaskCreateTool, TaskGetTool, TaskUpdateTool, TaskListTool] : []),
    // ... 更多条件工具
  ]
}

// 2. 获取过滤后的工具
export const getTools = (permissionContext: ToolPermissionContext): Tools => {
  // 简单模式：只返回基础工具
  if (isEnvTruthy(process.env.CLAUDE_CODE_SIMPLE)) {
    return [BashTool, FileReadTool, FileEditTool]
  }
  
  const tools = getAllBaseTools()
  
  // 根据deny规则过滤
  let allowedTools = filterToolsByDenyRules(tools, permissionContext)
  
  // 根据isEnabled过滤
  const isEnabled = allowedTools.map(_ => _.isEnabled())
  return allowedTools.filter((_, i) => isEnabled[i])
}

// 3. 组装工具池（包含MCP工具）
export function assembleToolPool(
  permissionContext: ToolPermissionContext,
  mcpTools: Tools,
): Tools {
  const builtInTools = getTools(permissionContext)
  
  // 过滤MCP工具
  const allowedMcpTools = filterToolsByDenyRules(mcpTools, permissionContext)
  
  // 合并并去重（内置工具优先）
  return uniqBy(
    [...builtInTools].sort(byName).concat(allowedMcpTools.sort(byName)),
    'name',
  )
}
```

---

## 2. QueryEngine完整走读

### 2.1 QueryEngine类结构

```typescript
// src/QueryEngine.ts

export class QueryEngine {
  // 配置（只读）
  private config: QueryEngineConfig
  
  // 可变状态
  private mutableMessages: Message[]
  private abortController: AbortController
  private permissionDenials: SDKPermissionDenial[]
  private totalUsage: NonNullableUsage
  private hasHandledOrphanedPermission = false
  private readFileState: FileStateCache
  
  // 技能发现跟踪
  private discoveredSkillNames = new Set<string>()
  private loadedNestedMemoryPaths = new Set<string>()

  constructor(config: QueryEngineConfig) {
    this.config = config
    this.mutableMessages = config.initialMessages ?? []
    this.abortController = config.abortController ?? createAbortController()
    this.permissionDenials = []
    this.readFileState = config.readFileCache
    this.totalUsage = EMPTY_USAGE
  }

  // 核心方法：提交消息
  async *submitMessage(
    prompt: string | ContentBlockParam[],
    options?: { uuid?: string; isMeta?: boolean },
  ): AsyncGenerator<SDKMessage, void, unknown> {
    // 实现详见下文
  }

  // 中断方法
  interrupt(): void {
    this.abortController.abort()
  }

  // 获取消息历史
  getMessages(): readonly Message[] {
    return this.mutableMessages
  }

  // 获取文件读取状态
  getReadFileState(): FileStateCache {
    return this.readFileState
  }
}
```

### 2.2 submitMessage方法详解

```typescript
async *submitMessage(prompt, options) {
  // ========== 1. 初始化阶段 ==========
  const {
    cwd,
    commands,
    tools,
    mcpClients,
    verbose = false,
    canUseTool,
    customSystemPrompt,
    appendSystemPrompt,
    userSpecifiedModel,
    fallbackModel,
    jsonSchema,
    getAppState,
    setAppState,
    // ... 更多配置
  } = this.config

  // 清理技能发现跟踪
  this.discoveredSkillNames.clear()
  setCwd(cwd)
  
  // 包装canUseTool以跟踪权限拒绝
  const wrappedCanUseTool: CanUseToolFn = async (...args) => {
    const result = await canUseTool(...args)
    if (result.behavior !== 'allow') {
      this.permissionDenials.push({
        tool_name: sdkCompatToolName(args[0].name),
        tool_use_id: args[4],
        tool_input: args[1],
      })
    }
    return result
  }

  // ========== 2. 系统提示词构建 ==========
  const initialAppState = getAppState()
  const initialMainLoopModel = userSpecifiedModel
    ? parseUserSpecifiedModel(userSpecifiedModel)
    : getMainLoopModel()

  // 获取系统提示词各部分
  const {
    defaultSystemPrompt,
    userContext: baseUserContext,
    systemContext,
  } = await fetchSystemPromptParts({
    tools,
    mainLoopModel: initialMainLoopModel,
    additionalWorkingDirectories: Array.from(
      initialAppState.toolPermissionContext.additionalWorkingDirectories.keys(),
    ),
    mcpClients,
    customSystemPrompt: typeof customSystemPrompt === 'string' ? customSystemPrompt : undefined,
  })

  // 合并用户上下文
  const userContext = {
    ...baseUserContext,
    ...getCoordinatorUserContext(mcpClients),
  }

  // 构建最终系统提示词
  const systemPrompt = asSystemPrompt([
    ...(customSystemPrompt !== undefined ? [customSystemPrompt] : defaultSystemPrompt),
    ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
    ...(appendSystemPrompt ? [appendSystemPrompt] : []),
  ])

  // ========== 3. 处理用户输入 ==========
  const processUserInputContext: ProcessUserInputContext = {
    messages: this.mutableMessages,
    setMessages: fn => { this.mutableMessages = fn(this.mutableMessages) },
    onChangeAPIKey: () => {},
    handleElicitation: this.config.handleElicitation,
    options: {
      commands,
      debug: false,
      tools,
      verbose,
      mainLoopModel: initialMainLoopModel,
      // ...
    },
    getAppState,
    setAppState,
    abortController: this.abortController,
    readFileState: this.readFileState,
    // ...
  }

  // 处理孤儿权限（如果有）
  if (orphanedPermission && !this.hasHandledOrphanedPermission) {
    this.hasHandledOrphanedPermission = true
    for await (const message of handleOrphanedPermission(...)) {
      yield message
    }
  }

  // 处理用户输入（展开slash命令等）
  const {
    messages: messagesFromUserInput,
    shouldQuery,
    allowedTools,
    model: modelFromUserInput,
    resultText,
  } = await processUserInput({
    input: prompt,
    mode: 'prompt',
    context: processUserInputContext,
    messages: this.mutableMessages,
    uuid: options?.uuid,
    isMeta: options?.isMeta,
    querySource: 'sdk',
  })

  // 推送新消息
  this.mutableMessages.push(...messagesFromUserInput)

  // 持久化用户消息
  if (persistSession && messagesFromUserInput.length > 0) {
    await recordTranscript(this.mutableMessages)
  }

  // 如果不是查询模式（如slash命令已处理），直接返回结果
  if (!shouldQuery) {
    yield {
      type: 'result',
      subtype: 'success',
      result: resultText ?? '',
      // ...
    }
    return
  }

  // ========== 4. 执行查询循环 ==========
  const mainLoopModel = modelFromUserInput ?? initialMainLoopModel
  
  // 更新工具权限上下文
  setAppState(prev => ({
    ...prev,
    toolPermissionContext: {
      ...prev.toolPermissionContext,
      alwaysAllowRules: {
        ...prev.toolPermissionContext.alwaysAllowRules,
        command: allowedTools,
      },
    },
  }))

  // 预加载技能和插件
  const [skills, { enabled: enabledPlugins }] = await Promise.all([
    getSlashCommandToolSkills(getCwd()),
    loadAllPluginsCacheOnly(),
  ])

  // 生成系统初始化消息
  yield buildSystemInitMessage({
    tools,
    mcpClients,
    model: mainLoopModel,
    permissionMode: initialAppState.toolPermissionContext.mode,
    commands,
    agents,
    skills,
    plugins: enabledPlugins,
    fastMode: initialAppState.fastMode,
  })

  // ========== 5. 查询循环 ==========
  let turnCount = 1
  let lastStopReason: string | null = null
  
  for await (const message of query({
    messages: this.mutableMessages,
    systemPrompt,
    userContext,
    systemContext,
    canUseTool: wrappedCanUseTool,
    toolUseContext: processUserInputContext,
    fallbackModel,
    querySource: 'sdk',
    maxTurns,
    taskBudget,
  })) {
    // 处理流式消息
    switch (message.type) {
      case 'assistant':
        // 处理助手消息
        this.mutableMessages.push(message)
        yield* normalizeMessage(message)
        
        // 检查是否有工具调用
        const toolUseBlocks = message.message.content.filter(
          content => content.type === 'tool_use'
        )
        if (toolUseBlocks.length > 0) {
          // 工具调用将在query内部处理
        }
        break
        
      case 'stream_event':
        // 处理流事件（用于token计数）
        if (message.event.type === 'message_stop') {
          this.totalUsage = accumulateUsage(
            this.totalUsage,
            currentMessageUsage,
          )
        }
        break
        
      case 'system':
        // 处理系统消息（如压缩边界）
        if (message.subtype === 'compact_boundary') {
          // 释放预压缩消息内存
          const mutableBoundaryIdx = this.mutableMessages.length - 1
          if (mutableBoundaryIdx > 0) {
            this.mutableMessages.splice(0, mutableBoundaryIdx)
          }
        }
        yield message
        break
        
      case 'tool_use_summary':
        // 工具使用摘要
        yield message
        break
    }

    // 检查预算限制
    if (maxBudgetUsd !== undefined && getTotalCost() >= maxBudgetUsd) {
      yield {
        type: 'result',
        subtype: 'error_max_budget_usd',
        is_error: true,
        errors: [`Reached maximum budget ($${maxBudgetUsd})`],
      }
      return
    }

    // 检查结构化输出重试限制
    if (message.type === 'user' && jsonSchema) {
      const currentCalls = countToolCalls(
        this.mutableMessages,
        SYNTHETIC_OUTPUT_TOOL_NAME,
      )
      if (currentCalls - initialStructuredOutputCalls >= maxRetries) {
        yield {
          type: 'result',
          subtype: 'error_max_structured_output_retries',
          is_error: true,
          errors: [`Failed to provide valid structured output after ${maxRetries} attempts`],
        }
        return
      }
    }
  }

  // ========== 6. 返回最终结果 ==========
  const result = messages.findLast(
    m => m.type === 'assistant' || m.type === 'user'
  )

  if (!isResultSuccessful(result, lastStopReason)) {
    yield {
      type: 'result',
      subtype: 'error_during_execution',
      is_error: true,
      errors: [...],
    }
    return
  }

  // 提取文本结果
  let textResult = ''
  if (result.type === 'assistant') {
    const lastContent = last(result.message.content)
    if (lastContent?.type === 'text') {
      textResult = lastContent.text
    }
  }

  yield {
    type: 'result',
    subtype: 'success',
    is_error: false,
    result: textResult,
    total_cost_usd: getTotalCost(),
    usage: this.totalUsage,
    modelUsage: getModelUsage(),
    permission_denials: this.permissionDenials,
    structured_output: structuredOutputFromTool,
  }
}
```

---

## 3. 状态管理完整走读

### 3.1 自定义Store实现

```typescript
// src/state/store.ts

// 1. 定义监听器类型
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

// 2. 定义Store接口
export type Store<T> = {
  getState: () => T                    // 获取当前状态
  setState: (updater: (prev: T) => T) => void  // 更新状态
  subscribe: (listener: Listener) => () => void  // 订阅变化
}

// 3. createStore工厂函数
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  // 内部状态
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    // 获取状态（只读）
    getState: () => state,

    // 更新状态
    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      
      // 引用相等检查 - 避免不必要更新
      if (Object.is(next, prev)) return
      
      // 更新状态
      state = next
      
      // 触发onChange回调
      onChange?.({ newState: next, oldState: prev })
      
      // 通知所有订阅者
      for (const listener of listeners) {
        listener()
      }
    },

    // 订阅状态变化
    subscribe: (listener: Listener) => {
      listeners.add(listener)
      // 返回取消订阅函数
      return () => listeners.delete(listener)
    },
  }
}
```

**设计亮点**:
- 使用`Object.is`进行引用相等检查，避免不必要重渲染
- 函数式更新：`setState(updater)`而非`setState(newState)`
- 订阅模式支持多个监听器
- 类型安全：泛型T确保类型一致性

### 3.2 React集成

```typescript
// src/state/AppState.tsx

// 1. 创建Context
export const AppStoreContext = React.createContext<AppStateStore | null>(null)

// 2. Provider组件
export function AppStateProvider({ 
  children, 
  initialState, 
  onChangeAppState 
}: Props) {
  // 使用useState创建store（只创建一次）
  const [store] = useState(() => 
    createStore(initialState ?? getDefaultAppState(), onChangeAppState)
  )
  
  return (
    <AppStoreContext.Provider value={store}>
      {children}
    </AppStoreContext.Provider>
  )
}

// 3. useAppState Hook
export function useAppState<T>(selector: (state: AppState) => T): T {
  const store = useAppStore()  // 获取store
  
  // 创建记忆化的getState函数
  const getState = useCallback(
    () => selector(store.getState()),
    [selector, store]
  )
  
  // 使用useSyncExternalStore订阅外部store
  return useSyncExternalStore(
    store.subscribe,    // 订阅函数
    getState,           // 客户端getState
    getState            // 服务端getState（同构）
  )
}

// 4. useSetAppState Hook
export function useSetAppState() {
  return useAppStore().setState
}

// 5. 内部辅助Hook
function useAppStore(): AppStateStore {
  const store = useContext(AppStoreContext)
  if (!store) {
    throw new ReferenceError(
      'useAppState/useSetAppState cannot be called outside of an <AppStateProvider />'
    )
  }
  return store
}
```

**设计亮点**:
- `useSyncExternalStore`: React 18官方推荐的外部store订阅方式
- 选择器模式：只订阅需要的部分状态，避免不必要重渲染
- 类型安全：泛型selector确保返回类型正确

### 3.3 全局状态 (bootstrap/state.ts)

```typescript
// src/bootstrap/state.ts

// 1. State类型定义（258行）
type State = {
  // 项目信息
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
  turnToolCount: number
  turnHookCount: number
  turnClassifierCount: number
  
  // 时间戳
  startTime: number
  lastInteractionTime: number
  
  // 代码统计
  totalLinesAdded: number
  totalLinesRemoved: number
  hasUnknownModelCost: boolean
  
  // 当前工作目录
  cwd: string
  
  // 模型使用
  modelUsage: { [modelName: string]: ModelUsage }
  mainLoopModelOverride: ModelSetting | undefined
  initialMainLoopModel: ModelSetting
  modelStrings: ModelStrings | null
  
  // 会话标志
  isInteractive: boolean
  kairosActive: boolean
  strictToolResultPairing: boolean
  sdkAgentProgressSummariesEnabled: boolean
  userMsgOptIn: boolean
  clientType: string
  sessionSource: string | undefined
  questionPreviewFormat: 'markdown' | 'html' | undefined
  
  // 设置相关
  flagSettingsPath: string | undefined
  flagSettingsInline: Record<string, unknown> | null
  allowedSettingSources: SettingSource[]
  sessionIngressToken: string | null | undefined
  oauthTokenFromFd: string | null | undefined
  apiKeyFromFd: string | null | undefined
  
  // 遥测
  meter: Meter | null
  sessionCounter: AttributedCounter | null
  locCounter: AttributedCounter | null
  prCounter: AttributedCounter | null
  commitCounter: AttributedCounter | null
  costCounter: AttributedCounter | null
  tokenCounter: AttributedCounter | null
  codeEditToolDecisionCounter: AttributedCounter | null
  activeTimeCounter: AttributedCounter | null
  statsStore: { observe(name: string, value: number): void } | null
  
  // 会话信息
  sessionId: SessionId
  parentSessionId: SessionId | undefined
  
  // 日志
  loggerProvider: LoggerProvider | null
  eventLogger: ReturnType<typeof logs.getLogger> | null
  meterProvider: MeterProvider | null
  tracerProvider: BasicTracerProvider | null
  
  // Agent颜色
  agentColorMap: Map<string, AgentColorName>
  agentColorIndex: number
  
  // API请求记录
  lastAPIRequest: Omit<BetaMessageStreamParams, 'messages'> | null
  lastAPIRequestMessages: BetaMessageStreamParams['messages'] | null
  lastClassifierRequests: unknown[] | null
  
  // 缓存内容
  cachedClaudeMdContent: string | null
  
  // 错误日志
  inMemoryErrorLog: Array<{ error: string; timestamp: string }>
  
  // 插件
  inlinePlugins: Array<string>
  chromeFlagOverride: boolean | undefined
  useCoworkPlugins: boolean
  
  // 权限模式
  sessionBypassPermissionsMode: boolean
  scheduledTasksEnabled: boolean
  sessionCronTasks: SessionCronTask[]
  sessionCreatedTeams: Set<string>
  sessionTrustAccepted: boolean
  sessionPersistenceDisabled: boolean
  
  // 计划模式
  hasExitedPlanMode: boolean
  needsPlanModeExitAttachment: boolean
  needsAutoModeExitAttachment: boolean
  lspRecommendationShownThisSession: boolean
  
  // SDK
  initJsonSchema: Record<string, unknown> | null
  registeredHooks: Partial<Record<HookEvent, RegisteredHookMatcher[]>> | null
  
  // 计划slug缓存
  planSlugCache: Map<string, string>
  
  // 远程会话
  teleportedSessionInfo: {...} | null
  
  // 调用的技能
  invokedSkills: Map<string, {...}>
  
  // 慢操作
  slowOperations: Array<{ operation: string; durationMs: number; timestamp: number }>
  
  // SDK betas
  sdkBetas: string[] | undefined
  
  // 主线程Agent类型
  mainThreadAgentType: string | undefined
  
  // 远程模式
  isRemoteMode: boolean
  directConnectServerUrl: string | undefined
  
  // 系统提示词缓存
  systemPromptSectionCache: Map<string, string | null>
  
  // 最后发出的日期
  lastEmittedDate: string | null
  
  // 额外目录
  additionalDirectoriesForClaudeMd: string[]
  
  // 允许的频道
  allowedChannels: ChannelEntry[]
  hasDevChannels: boolean
  
  // 会话项目目录
  sessionProjectDir: string | null
  
  // 提示缓存配置
  promptCache1hAllowlist: string[] | null
  promptCache1hEligible: boolean | null
  
  // Beta header锁存
  afkModeHeaderLatched: boolean | null
  fastModeHeaderLatched: boolean | null
  cacheEditingHeaderLatched: boolean | null
  thinkingClearLatched: boolean | null
  
  // 当前提示ID
  promptId: string | null
  
  // 最后API请求ID
  lastMainRequestId: string | undefined
  
  // 最后API完成时间戳
  lastApiCompletionTimestamp: number | null
  
  // 待处理的后压缩标志
  pendingPostCompaction: boolean
}

// 2. 单例STATE
const STATE: State = getInitialState()

// 3. Getter/Setter方法
export function getSessionId(): SessionId {
  return STATE.sessionId
}

export function regenerateSessionId(options?: { setCurrentAsParent?: boolean }): SessionId {
  if (options?.setCurrentAsParent) {
    STATE.parentSessionId = STATE.sessionId
  }
  STATE.planSlugCache.delete(STATE.sessionId)
  STATE.sessionId = randomUUID() as SessionId
  STATE.sessionProjectDir = null
  return STATE.sessionId
}

export function switchSession(sessionId: SessionId, projectDir: string | null = null): void {
  STATE.planSlugCache.delete(STATE.sessionId)
  STATE.sessionId = sessionId
  STATE.sessionProjectDir = projectDir
  sessionSwitched.emit(sessionId)
}

export function addToTotalCost(costUSD: number): void {
  STATE.totalCostUSD += costUSD
}

export function getTotalCostUSD(): number {
  return STATE.totalCostUSD
}

// ... 更多getter/setter
```

---

## 4. 启动流程完整走读

### 4.1 main.tsx启动流程

```typescript
// src/main.tsx

// 1. 启动性能分析
profileCheckpoint('main_tsx_entry')

// 2. 并行预取（在重模块加载前）
startMdmRawRead()        // MDM设置
startKeychainPrefetch()  // macOS钥匙串

// 3. 安全检查
process.env.NoDefaultCurrentDirectoryInExePath = '1'

// 4. 初始化警告处理器
initializeWarningHandler()

// 5. 处理信号
process.on('exit', () => { resetCursor() })
process.on('SIGINT', () => {
  if (process.argv.includes('-p') || process.argv.includes('--print')) {
    return  // 非交互模式不处理
  }
  process.exit(0)
})

// 6. 处理特殊URL协议
if (feature('DIRECT_CONNECT')) {
  // 处理cc://和cc+unix://URL
  // ...
}

// 7. 处理深度链接
if (feature('LODESTONE')) {
  // 处理--handle-uri参数
  // ...
}

// 8. 处理assistant命令
if (feature('KAIROS') && _pendingAssistantChat) {
  // 处理`claude assistant [sessionId]`
  // ...
}

// 9. 处理SSH远程
if (feature('SSH_REMOTE') && _pendingSSH) {
  // 处理`claude ssh <host> [dir]`
  // ...
}

// 10. 确定交互模式
const hasPrintFlag = cliArgs.includes('-p') || cliArgs.includes('--print')
const hasInitOnlyFlag = cliArgs.includes('--init-only')
const hasSdkUrl = cliArgs.some(arg => arg.startsWith('--sdk-url'))
const isNonInteractive = hasPrintFlag || hasInitOnlyFlag || hasSdkUrl || !process.stdout.isTTY

// 11. 设置交互标志
setIsInteractive(!isNonInteractive)

// 12. 确定客户端类型
const clientType = (() => {
  if (isEnvTruthy(process.env.GITHUB_ACTIONS)) return 'github-action'
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-ts') return 'sdk-typescript'
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-py') return 'sdk-python'
  if (process.env.CLAUDE_CODE_ENTRYPOINT === 'sdk-cli') return 'sdk-cli'
  // ... 更多类型
  return 'cli'
})()
setClientType(clientType)

// 13. 解析设置标志
eagerLoadSettings()

// 14. 运行主逻辑
await run()
```

### 4.2 init.ts初始化流程

```typescript
// src/entrypoints/init.ts

export const init = memoize(async (): Promise<void> => {
  const initStartTime = Date.now()
  
  // 1. 启用配置系统
  enableConfigs()
  
  // 2. 应用安全环境变量（信任对话框之前）
  applySafeConfigEnvironmentVariables()
  
  // 3. 应用CA证书
  applyExtraCACertsFromConfig()
  
  // 4. 设置优雅关闭
  setupGracefulShutdown()
  
  // 5. 初始化1P事件日志（延迟加载）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(([fp, gb]) => {
    fp.initialize1PEventLogging()
    gb.onGrowthBookRefresh(() => {
      void fp.reinitialize1PEventLoggingIfConfigChanged()
    })
  })
  
  // 6. 填充OAuth账户信息
  void populateOAuthAccountInfoIfNeeded()
  
  // 7. 初始化JetBrains检测
  void initJetBrainsDetection()
  
  // 8. 检测当前仓库
  void detectCurrentRepository()
  
  // 9. 初始化远程管理设置加载Promise
  if (isEligibleForRemoteManagedSettings()) {
    initializeRemoteManagedSettingsLoadingPromise()
  }
  if (isPolicyLimitsEligible()) {
    initializePolicyLimitsLoadingPromise()
  }
  
  // 10. 记录首次启动时间
  recordFirstStartTime()
  
  // 11. 配置全局mTLS
  configureGlobalMTLS()
  
  // 12. 配置全局HTTP代理
  configureGlobalAgents()
  
  // 13. 预连接Anthropic API
  preconnectAnthropicApi()
  
  // 14. 上游代理（如果是远程模式）
  if (isEnvTruthy(process.env.CLAUDE_CODE_REMOTE)) {
    try {
      const { initUpstreamProxy, getUpstreamProxyEnv } = await import('../upstreamproxy/upstreamproxy.js')
      const { registerUpstreamProxyEnvFn } = await import('../utils/subprocessEnv.js')
      registerUpstreamProxyEnvFn(getUpstreamProxyEnv)
      await initUpstreamProxy()
    } catch (err) {
      logForDebugging(`[init] upstreamproxy init failed: ${err}`)
    }
  }
  
  // 15. 设置Windows shell
  setShellIfWindows()
  
  // 16. 注册LSP管理器清理
  registerCleanup(shutdownLspServerManager)
  
  // 17. 注册团队清理
  registerCleanup(async () => {
    const { cleanupSessionTeams } = await import('../utils/swarm/teamHelpers.js')
    await cleanupSessionTeams()
  })
  
  // 18. 初始化scratchpad目录
  if (isScratchpadEnabled()) {
    await ensureScratchpadDir()
  }
})
```

---

## 5. 权限系统完整走读

### 5.1 权限上下文

```typescript
// src/Tool.ts

export type ToolPermissionContext = DeepImmutable<{
  // 当前权限模式
  mode: PermissionMode  // 'default' | 'plan' | 'bypassPermissions' | 'auto' | 'yolo'
  
  // 额外工作目录
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  
  // 规则定义
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  
  // 功能可用性
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  
  // 剥离的危险规则
  strippedDangerousRules?: ToolPermissionRulesBySource
  
  // 标志
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}>

export const getEmptyToolPermissionContext: () => ToolPermissionContext = () => ({
  mode: 'default',
  additionalWorkingDirectories: new Map(),
  alwaysAllowRules: {},
  alwaysDenyRules: {},
  alwaysAskRules: {},
  isBypassPermissionsModeAvailable: false,
})
```

### 5.2 权限检查流程

```typescript
// src/hooks/useCanUseTool.ts

export type PermissionResult =
  | { behavior: 'allow'; updatedInput: unknown }
  | { behavior: 'deny'; reason: string }
  | { behavior: 'ask'; message: string; updatedInput: unknown }

export type CanUseToolFn = (
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: 'allow' | 'deny',
) => Promise<PermissionResult>

export function useCanUseTool(): CanUseToolFn {
  return async (tool, input, context, assistantMessage, toolUseID, forceDecision) => {
    const appState = context.getAppState()
    const permissionContext = appState.toolPermissionContext
    
    // 1. 检查强制决策
    if (forceDecision) {
      return { behavior: forceDecision, updatedInput: input }
    }
    
    // 2. 检查deny规则
    const denyRule = getDenyRuleForTool(permissionContext, tool)
    if (denyRule) {
      return { behavior: 'deny', reason: `Tool ${tool.name} is blocked by policy` }
    }
    
    // 3. 检查always allow规则
    const allowRule = getAllowRuleForTool(permissionContext, tool, input)
    if (allowRule) {
      return { behavior: 'allow', updatedInput: input }
    }
    
    // 4. 根据模式处理
    switch (permissionContext.mode) {
      case 'bypassPermissions':
        return { behavior: 'allow', updatedInput: input }
        
      case 'yolo':
        // YOLO模式：自动允许，但记录
        logEvent('tengu_yolo_tool_allowed', { tool: tool.name })
        return { behavior: 'allow', updatedInput: input }
        
      case 'auto':
        // 自动模式：使用分类器
        const autoResult = await autoClassifier(tool, input, context)
        if (autoResult.confidence > 0.9) {
          return { behavior: autoResult.decision, updatedInput: input }
        }
        // 置信度低，转为询问
        break
        
      case 'plan':
        // 计划模式：检查是否在允许列表
        if (isToolInPlan(tool, input, context)) {
          return { behavior: 'allow', updatedInput: input }
        }
        break
    }
    
    // 5. 调用工具的checkPermissions方法
    const toolPermission = await tool.checkPermissions(input as any, context)
    if (toolPermission.behavior !== 'allow') {
      return toolPermission
    }
    
    // 6. 默认询问用户
    return {
      behavior: 'ask',
      message: `Allow ${tool.name} to execute?`,
      updatedInput: input,
    }
  }
}
```

---

## 6. 总结

### 6.1 关键设计模式

1. **工厂模式**: `buildTool`提供工具默认值
2. **依赖注入**: `QueryDeps`接口支持测试替换
3. **订阅模式**: `Store<T>`实现状态订阅
4. **单例模式**: `STATE`全局状态
5. **策略模式**: 权限模式（default/plan/bypassPermissions/auto/yolo）

### 6.2 性能优化

1. **并行启动**: MDM设置、钥匙串读取与模块加载并行
2. **延迟加载**: OpenTelemetry、gRPC等重型模块按需加载
3. **缓存策略**: `memoize`缓存昂贵的上下文计算
4. **引用相等检查**: `Object.is`避免不必要重渲染

### 6.3 可维护性

1. **类型安全**: 严格的TypeScript，复杂类型体操
2. **模块化**: 清晰的模块边界，职责单一
3. **可测试性**: 依赖注入、纯函数
4. **错误处理**: 分级错误、优雅降级

---

*文档完成*
