[上一篇](04-核心模块与类关系.md) · [总目录](README.md) · [下一篇](06-函数级源码解析-Agent引擎与循环.md)

# 05-函数级源码解析-CLI入口与主流程

> **场景**：CLI 入口到运行时分发的函数级分析
> **版本**：0.0.3（commit `853a80d26`）
> **源码基准**：当前 `main` 分支源码

## 1. 函数：cli.ts 模块入口

**源码**：`packages/coding-agent/src/cli.ts`
**所属**：模块顶层
**职责**：设置进程元信息、配置 HTTP dispatcher、调用 main()
**调用方**：Node.js 运行时（`#!/usr/bin/env node`）
**被调用**：`configureHttpDispatcher`、`main`

### 执行流程

```
偏移 +12: process.title = APP_NAME ("pi")
偏移 +13: process.env.PI_CODING_AGENT = "true"
偏移 +14: process.env.AI_AGENT = "pi"
偏移 +15: process.emitWarning = (() => {})  ← 屏蔽 Node.js 警告噪音
偏移 +19: configureHttpDispatcher()        ← 配置 undici 全局 dispatcher
偏移 +21: main(process.argv.slice(2))       ← 调用主函数
```

### 设计意图

在导入任何 Provider SDK 前配置 HTTP dispatcher，确保全局 HTTP 代理设置生效。`process.emitWarning` 被屏蔽以避免 DeprecationWarning 污染 stdout（print/json 模式输出需要纯净 stdout）。

## 2. 函数：main

**源码**：`packages/coding-agent/src/main.ts`
**所属**：模块顶层
**职责**：CLI 参数解析、运行时创建、模式分发
**调用方**：`cli.ts`
**参数**：`args: string[]`, `options?: MainOptions`
**返回值**：`Promise<void>`

### 执行流程

```
偏移 +562: resetTimings()                    ← 重置计时器
偏移 +563: extensionFactories = [...]        ← 合并内置扩展
偏移 +564: offlineMode = args.includes("--offline") || env check
偏移 +570: if (await runAuthCommand(args)) return  ← auth 命令拦截
偏移 +579: cwd = process.cwd()
偏移 +580: agentDir = getAgentDir()
偏移 +581: bootstrapSettingsManager = SettingsManager.create(...)
偏移 +585: if (await handlePackageCommand(args)) return  ← package 命令拦截
偏移 +598: if (await handleConfigCommand(args)) return  ← config 命令拦截
偏移 +602: parsed = parseArgs(args)          ← 核心：解析 CLI 参数
偏移 +633: appMode = resolveAppMode(parsed, stdin, stdout)  ← 决定运行模式
偏移 +644: validateForkFlags(parsed)
偏移 +645: validateSessionIdFlags(parsed)
偏移 +648: runMigrations(cwd)                ← 运行数据迁移
偏移 +651: startupSettingsManager = SettingsManager.create(cwd, agentDir)
偏移 +670: sessionDir = resolveSessionDir(...)
偏移 +675: sessionManager = await createSessionManager(parsed, cwd, sessionDir, ...)
偏移 +699: trustStore = new ProjectTrustStore(agentDir)
偏移 +712: createRuntime = async ({ cwd, agentDir, sessionManager, ... }) => { ... }
偏移 +840: runtime = await createAgentSessionRuntime(createRuntime, { cwd, agentDir, sessionManager })
偏移 +846: { services, session, modelFallbackMessage } = runtime
偏移 +848: setCapabilityOverrides(...)
偏移 +849: applyHttpProxySettings(...)
偏移 +870: stdinContent = await readPipedStdin()  ← 读取管道输入
偏移 +878: { initialMessage, initialImages } = await prepareInitialMessage(...)
偏移 +884: initTheme(...)
偏移 +927: if (appMode === "rpc") await runRpcMode(runtime)
偏移 +930: else if (interactive) new InteractiveMode(runtime, ...).run()
偏移 +963: else await runPrintMode(runtime, { mode, messages, initialMessage, initialImages })
```

### 关键分支

| 条件 | 分支 |
|------|------|
| `parsed.version` | 输出版本号，`process.exit(0)` |
| `parsed.export` | 导出 HTML，`process.exit(0)` |
| `parsed.help` | 输出帮助，`process.exit(0)` |
| `parsed.listModels !== undefined` | 列出模型，`process.exit(0)` |
| `appMode === "rpc"` | `runRpcMode(runtime)` |
| `appMode === "interactive"` | `new InteractiveMode(runtime, opts).run()` |
| `appMode === "print"` 或 `"json"` | `runPrintMode(runtime, opts)` |

### 参数传递

```
args: string[]
  ↓ parseArgs
Args { print, mode, messages, model, provider, thinking, tools, ... }
  ↓ resolveAppMode
AppMode: "interactive" | "print" | "json" | "rpc"
  ↓ createRuntime closure
CreateAgentSessionRuntimeFactory({ cwd, agentDir, sessionManager, ... })
  ↓ createAgentSessionRuntime
AgentSessionRuntime { session, services, ... }
  ↓ runPrintMode / InteractiveMode.run / runRpcMode
```

## 3. 函数：createSessionManager

**源码**：`packages/coding-agent/src/main.ts`
**所属**：模块顶层
**职责**：根据 CLI 参数创建/打开/继续/恢复/fork 会话
**参数**：`parsed: Args`, `cwd: string`, `sessionDir: string | undefined`, `settingsManager: SettingsManager`
**返回值**：`Promise<SessionManager>`

### 执行流程

```
偏移 +358: if (parsed.noSession || help || listModels) → SessionManager.inMemory(cwd)
偏移 +362: if (parsed.fork) → resolveSessionPath → forkSessionOrExit
偏移 +385: if (parsed.session) → resolveSessionPath → openSessionOrExit
偏移 +409: if (parsed.resume) → selectSession → SessionManager.open
偏移 +426: if (parsed.continue) → SessionManager.continueRecent(cwd, sessionDir)
偏移 +430: if (parsed.sessionId) → findLocalSessionByExactId → open or create with id
偏移 +442: return SessionManager.create(cwd, sessionDir, { id: parsed.sessionId })
```

## 4. 函数：createAgentSession

**源码**：`packages/coding-agent/src/core/sdk.ts`
**所属**：模块顶层
**职责**：创建 Agent + AgentSession，注入所有依赖
**参数**：`options: CreateAgentSessionOptions = {}`
**返回值**：`Promise<CreateAgentSessionResult>`

### 执行流程

```
偏移 +174: cwd = resolvePath(options.cwd ?? process.cwd())
偏移 +175: agentDir = resolvePath(options.agentDir ?? getDefaultAgentDir())
偏移 +180: modelRuntime = options.modelRuntime ?? ModelRuntime.create({ authPath, modelsPath })
偏移 +182: settingsManager = options.settingsManager ?? SettingsManager.create(cwd, agentDir)
偏移 +183: sessionManager = options.sessionManager ?? SessionManager.create(cwd, getDefaultSessionDir(...))
偏移 +186: resourceLoader = new DefaultResourceLoader({ cwd, agentDir, settingsManager })
偏移 +187: await resourceLoader.reload()

偏移 +192: existingSession = sessionManager.buildSessionContext()
偏移 +193: hasExistingSession = existingSession.messages.length > 0

偏移 +196: model = options.model
偏移 +200: if (!model && hasExistingSession && existingSession.model) → 从 session 恢复 model
偏移 +212: if (!model) → findInitialModel({ ... })

偏移 +229: thinkingLevel = options.thinkingLevel
偏移 +232: if (undefined && hasExistingSession) → 从 session 恢复 thinkingLevel
偏移 +239: if (undefined && model) → 从 per-model settings 获取
偏移 +245: if (undefined) → 从 settings 获取或 DEFAULT_THINKING_LEVEL
偏移 +253: clampThinkingLevel(model, thinkingLevel)

偏移 +256: defaultActiveToolNames = ["read", "bash", "edit", "write"]
偏移 +261: initialActiveToolNames = (options.tools ?? configuredDefaultToolNames ?? defaultActiveToolNames)

偏移 +268: convertToLlmWithBlockImages = (messages) => { convertToLlm → filter images if blockImages }

偏移 +306: agent = new Agent({
  initialState: { systemPrompt: "", model, thinkingLevel, tools: [] },
  convertToLlm: convertToLlmWithBlockImages,
  streamFn: async (model, context, options) => { ... },  ← 偏移 +314
  onPayload, onResponse, transformContext,
  steeringMode, followUpMode, transport, thinkingBudgets, ...
})

偏移 +375: if (hasExistingSession) → agent.state.messages = existingSession.messages
偏移 +381: else → sessionManager.appendModelChange(...) + appendThinkingLevelChange(...)

偏移 +388: session = new AgentSession({ agent, sessionManager, settingsManager, cwd, ... })
偏移 +403: extensionsResult = resourceLoader.getExtensions()
偏移 +405: return { session, extensionsResult, modelFallbackMessage }
```

### streamFn 闭包（偏移 +314 ～ +342）

```typescript
async (model, context, options) => {
  const providerRetrySettings = settingsManager.getProviderRetrySettings();
  const timeoutMs = options?.timeoutMs ?? providerRetrySettings.timeoutMs ?? effectiveTimeoutMs;
  return modelRuntime.streamSimple(model, context, {
    ...options,
    timeoutMs,
    maxRetries: options?.maxRetries ?? providerRetrySettings.maxRetries,
    transformHeaders: async (requestHeaders) => {
      const headers = mergeProviderAttributionHeaders(model, settingsManager, options?.sessionId, requestHeaders);
      return headerRunner?.hasHandlers("before_provider_headers")
        ? headerRunner.emitBeforeProviderHeaders(headers ?? {})
        : (headers ?? {});
    },
  });
}
```

**数据流**：`Agent` 调用 `streamFn(model, context, options)` → 闭包添加 retry/timeout/headers → `modelRuntime.streamSimple(model, context, enhancedOptions)` → pi-ai

## 5. 函数：resolveAppMode

**源码**：`packages/coding-agent/src/main.ts`
**所属**：模块顶层
**职责**：根据 CLI 参数和 TTY 状态决定运行模式
**参数**：`parsed: Args`, `stdinIsTTY: boolean`, `stdoutIsTTY: boolean`
**返回值**：`AppMode`

### 执行流程

```
偏移 +111: if (parsed.mode === "rpc") → "rpc"
偏移 +114: if (parsed.mode === "json") → "json"
偏移 +117: if (parsed.print || !stdinIsTTY || !stdoutIsTTY) → "print"
偏移 +120: return "interactive"
```

## 6. 函数：buildSessionOptions

**源码**：`packages/coding-agent/src/main.ts`
**所属**：模块顶层
**职责**：从 CLI 参数构建 `CreateAgentSessionOptions`
**参数**：`parsed: Args`, `scopedModels: ScopedModel[]`, `hasExistingSession: boolean`, `modelRuntime`, `settingsManager`
**返回值**：`{ options, cliThinkingFromModel, diagnostics }`

### 关键逻辑

```
偏移 +463: if (parsed.model) → resolveCliModel({ cliProvider, cliModel, cliThinking, modelRuntime })
偏移 +487: if (!options.model && scopedModels.length > 0 && !hasExistingSession) → 使用 saved default 或 scopedModels[0]
偏移 +510: if (parsed.thinking) → options.thinkingLevel = parsed.thinking (覆盖)
偏移 +517: if (scopedModels.length > 0) → options.scopedModels = scopedModels.map(...)
偏移 +528: if (parsed.noTools) → options.noTools = "all"
偏移 +530: else if (parsed.noBuiltinTools) → options.noTools = "builtin"
偏移 +533: if (parsed.tools) → options.tools = [...parsed.tools]
偏移 +536: if (parsed.excludeTools) → options.excludeTools = [...parsed.excludeTools]
```

## 7. 函数：createAgentSessionRuntime

**源码**：`packages/coding-agent/src/core/agent-session-runtime.ts`
**所属**：模块顶层
**职责**：工厂编排器 — 调用 `createRuntime` 工厂、绑定扩展、设置重新绑定回调
**参数**：`factory: CreateAgentSessionRuntimeFactory`, `{ cwd, agentDir, sessionManager }`
**返回值**：`Promise<AgentSessionRuntime>`

### 执行流程

1. 调用 `factory({ cwd, agentDir, sessionManager })` → 返回 `{ session, services, diagnostics }`
2. 创建 `AgentSessionRuntime` 对象，持有 `session` 和 `services`
3. 设置 `runtime.setRebindSession` 回调（用于会话切换时重新绑定）
4. 返回 runtime

## 8. 函数：createAgentSessionServices

**源码**：`packages/coding-agent/src/core/agent-session-services.ts`
**所属**：模块顶层
**职责**：创建 `ModelRuntime`、`SettingsManager`、`ResourceLoader` 并应用扩展 flags
**参数**：`{ cwd, agentDir, settingsManager, ... }`
**返回值**：`Promise<{ settingsManager, modelRuntime, resourceLoader, diagnostics }>`

### 执行流程

```
1. modelRuntime = await ModelRuntime.create({ authPath, modelsPath, allowModelNetwork, signal })
2. resourceLoader = new DefaultResourceLoader({ cwd, agentDir, settingsManager })
3. await resourceLoader.reload()
4. 应用 extension flags (parsed.unknownFlags → extension flag values)
5. return { settingsManager, modelRuntime, resourceLoader, diagnostics }
```

## 9. 数据流图

```
CLI args (string[])
    │
    ▼ parseArgs
Args
    │
    ├── resolveAppMode → AppMode
    ├── createSessionManager → SessionManager
    ├── createAgentSessionServices → { ModelRuntime, SettingsManager, ResourceLoader }
    ├── resolveModelScope → ScopedModel[]
    ├── buildSessionOptions → CreateAgentSessionOptions
    │
    ▼ createAgentSession
Agent + AgentSession (注入 streamFn, convertToLlm, tools, model, thinkingLevel)
    │
    ▼ createAgentSessionRuntime
AgentSessionRuntime { session, services }
    │
    ▼ runPrintMode / InteractiveMode.run / runRpcMode
输出到 stdout / TUI / JSON-RPC
```

## 10. 错误处理

| 错误来源 | 处理方式 |
|---------|---------|
| `parseArgs` 诊断 | `chalk.red` 输出到 stderr，如果是 error 则 `process.exit(1)` |
| Session 创建失败 | `openSessionOrExit` / `forkSessionOrExit` → `process.exit(1)` |
| 扩展加载失败 | diagnostic `{ type: "error" }` → 输出错误 + 提示 `-ne` flag → `process.exit(1)` |
| 无可用 model | `formatNoModelsAvailableMessage()` → `process.exit(1)` (非 interactive 模式) |
| `--api-key` 无 model | diagnostic error → `process.exit(1)` |

---

[上一篇](04-核心模块与类关系.md) · [总目录](README.md) · [下一篇](06-函数级源码解析-Agent引擎与循环.md)
