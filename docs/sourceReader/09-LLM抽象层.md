# 09-函数级源码解析-LLM抽象层

> **定位**：`packages/ai/`（npm 包 `@earendil-works/pi-ai`）
> **职责**：统一多供应商 LLM 流式调用、模型发现、认证解析、重试策略
> **上游消费者**：`packages/coding-agent/src/core/model-runtime.ts`（`ModelRuntime` 类）

---

## 1. 模块全景

```
packages/ai/src/
├── types.ts              # 核心类型：Model, Context, AssistantMessage, AssistantMessageEvent, ...
├── models.ts             # Models 接口 + ModelsImpl 实现 + createModels() + createProvider()
├── models-store.ts       # ModelsStore 持久化接口 + InMemoryModelsStore
├── model-catalog.ts      # flattenModelCatalog() 类型工具
├── models.generated.ts   # 代码生成的内置模型目录
├── providers/all.ts      # builtinProviders() — 39 个内置供应商工厂
├── api/
│   ├── lazy.ts           # lazyStream() + lazyApi() — 延迟加载核心
│   ├── simple-options.ts # buildBaseOptions(), clampMaxTokensToContext(), thinkingBudget 计算
│   ├── openai-completions.ts / .lazy.ts
│   ├── anthropic-messages.ts / .lazy.ts
│   ├── google-generative-ai.ts / .lazy.ts
│   ├── ...（每个 API 协议一对 .ts + .lazy.ts）
│   └── transform-messages.ts
├── auth/
│   ├── types.ts           # Credential, CredentialStore, AuthContext, ProviderAuth, ...
│   ├── context.ts         # defaultProviderAuthContext()
│   ├── credential-store.ts # InMemoryCredentialStore
│   ├── resolve.ts         # resolveProviderAuth() — 认证解析核心
│   ├── helpers.ts         # envApiKeyAuth(), lazyOAuth()
│   └── oauth/             # 各 OAuth 实现（anthropic, openai-codex, github-copilot, ...）
├── utils/
│   ├── event-stream.ts    # EventStream<T,R> + AssistantMessageEventStream
│   ├── retry.ts           # retryAssistantCall(), isRetryableAssistantError()
│   ├── abort.ts           # operationSignal(), raceWithAbortSignal()
│   └── ...
└── index.ts              # 公共导出汇总
```

### 架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                    coding-agent                             │
│  ┌─────────────────────────────────────────────────────┐     │
│  │  ModelRuntime (model-runtime.ts)                   │     │
│  │  - 包装 pi-ai Models，叠加 models.json 配置        │     │
│  │  - 注入 RuntimeCredentials + FileModelsStore        │     │
│  │  - streamSimple() → lazyStream → prepareRequest    │     │
│  └──────────────────────┬──────────────────────────────┘     │
└─────────────────────────┼───────────────────────────────────┘
                          │ @earendil-works/pi-ai
┌─────────────────────────▼───────────────────────────────────┐
│  Models (models.ts — ModelsImpl)                             │
│  - providers: Map<string, Provider>                         │
│  - credentials: CredentialStore                             │
│  - applyAuth() → resolveProviderAuth() → mergeHeaders()      │
│  - streamSimple() → lazyStream → provider.streamSimple()     │
└─────────┬───────────────────────────────────┬────────────────┘
          │                                   │
    ┌─────▼──────┐                    ┌───────▼────────┐
    │ Provider   │                    │ CredentialStore│
    │ (per-vendor)│                   │ (auth.json)    │
    │ stream()   │                    └────────────────┘
    │ streamSimple()
    └───┬────────┘
        │ lazyApi()
┌───────▼──────────────────────────────┐
│  API Implementation Module            │
│  (openai-completions.ts, etc.)        │
│  - HTTP fetch → SSE/WebSocket parse   │
│  - Event stream → AssistantMessage    │
└───────────────────────────────────────┘
```

---

## 2. 核心类型定义

**源码**：`packages/ai/src/types.ts`

### 2.1 Model<TApi>

```
types.ts → interface Model
偏移：+821
```

```typescript
interface Model<TApi extends Api> {
  id: string;            // "claude-opus-4-7"
  name: string;          // "Claude Opus 4.7"
  api: TApi;             // "anthropic-messages" | "openai-completions" | ...
  provider: ProviderId;  // "anthropic" | "openai" | ...
  baseUrl: string;       // "https://api.anthropic.com"
  reasoning: boolean;    // 是否支持 thinking
  thinkingLevelMap?: ThinkingLevelMap;
  input: ("text" | "image")[];
  cost: ModelCost;       // 按百万 token 计价
  contextWindow: number; // 上下文窗口
  maxTokens: number;     // 最大输出
  samplingParams?: Record<string, unknown>;
  headers?: Record<string, string>;
  compat?: /* API 特定兼容性配置 */;
}
```

**关键设计**：`api` 字段是一个泛型参数，决定了该模型使用哪种 API 协议（`KnownApi`）。`Provider` 通过 `model.api` 分派到对应的 `ProviderStreams` 实现。

### 2.2 AssistantMessageEvent

```
types.ts → type AssistantMessageEvent
偏移：+535
```

流事件协议——所有 API 实现必须产出这个事件序列：

```
start → [text_start → text_delta* → text_end]*
      → [thinking_start → thinking_delta* → thinking_end]*
      → [toolcall_start → toolcall_delta* → toolcall_end]*
      → done | error
```

事件类型：
- `start`：流开始，携带初始 `partial` AssistantMessage
- `text_delta`：增量文本
- `thinking_delta`：增量思考
- `toolcall_delta`：增量工具调用参数
- `done`：成功终止，携带最终 `message`
- `error`：错误终止，携带 `error` AssistantMessage（`stopReason: "error" | "aborted"`）

### 2.3 Context

```
types.ts → interface Context
偏移：+521
```

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];    // UserMessage | AssistantMessage | ToolResultMessage
  tools?: Tool[];         // 工具定义（name + description + parameters schema）
}
```

这是发给供应商的统一请求上下文。各 API 实现模块负责将其转换为供应商特定格式。

### 2.4 Provider / ProviderStreams / ProviderAuth

```
models.ts → interface Provider<TApi>
偏移：+97

types.ts → interface ProviderStreams
偏移：+272

auth/types.ts → interface ProviderAuth
偏移：+237
```

```typescript
interface Provider<TApi extends Api = Api> {
  readonly id: string;
  readonly name: string;
  readonly baseUrl?: string;
  readonly headers?: ProviderHeaders;
  readonly auth: ProviderAuth;          // 至少 apiKey 或 oauth 之一
  getModels(): readonly Model<TApi>[];  // 同步返回已知模型
  refreshModels?(context): Promise<void>; // 动态供应商
  filterModels?(models, credential): readonly Model<TApi>[];
  stream<T>(model, context, options?): AssistantMessageEventStream;
  streamSimple(model, context, options?): AssistantMessageEventStream;
  fetchDeferred?(model, handle, options?): AssistantMessageEventStream;
  cancelDeferred?(model, handle, options?): Promise<void>;
}

interface ProviderAuth {
  apiKey?: ApiKeyAuth;    // 环境变量/存储 key 解析
  oauth?: OAuthAuth;     // OAuth login/refresh/toAuth
}
```

---

## 3. EventStream — 异步迭代器模式

**源码**：`packages/ai/src/utils/event-stream.ts`

### 3.1 EventStream<T, R> 泛型类

```
event-stream.ts → class EventStream<T, R>
偏移：+4（class 定义）
```

**职责**：一个生产者-消费者队列，同时提供 `AsyncIterable<T>` 和 `result(): Promise<R>`。

**内部状态**：
- `queue: T[]`：待消费的事件缓冲
- `waiting: ((value: IteratorResult<T>) => void)[]`：等待事件的消费者
- `done: boolean`：是否已终止
- `finalResultPromise: Promise<R>` + `resolveFinalResult`
- `isComplete: (event: T) => boolean`：判断事件是否是终止事件
- `extractResult: (event: T) => R`：从终止事件提取最终结果

**核心方法**：

#### push(event: T)

```
event-stream.ts → EventStream.push
偏移：+21
```

1. 如果 `done`，丢弃
2. 如果 `isComplete(event)`：设 `done = true`，解析 `finalResultPromise`
3. 如果有等待的消费者（`waiting` 队列），直接交付；否则入队 `queue`

#### end(result?: R)

```
event-stream.ts → EventStream.end
偏移：+38
```

强制终止流：
1. 设 `done = true`
2. 如果给了 `result`，解析 `finalResultPromise`
3. 通知所有等待中的消费者 `{ done: true }`

#### [Symbol.asyncIterator]()

```
event-stream.ts → EventStream.[Symbol.asyncIterator]
偏移：+50
```

循环：
1. 队列有事件 → `yield` 出来
2. 队列空 + `done` → `return`
3. 队列空 + 未 done → `await new Promise`，等 `push` 或 `end` 唤醒

#### result(): Promise<R>

```
event-stream.ts → EventStream.result
偏移：+64
```

返回 `finalResultPromise`——在 `push(isComplete)` 或 `end(result)` 时解析。

### 3.2 AssistantMessageEventStream

```
event-stream.ts → class AssistantMessageEventStream
偏移：+69
```

预配置的特化：
- `isComplete`：`event.type === "done" || event.type === "error"`
- `extractResult`：从 `done` 事件取 `.message`，从 `error` 事件取 `.error`

```typescript
class AssistantMessageEventStream extends EventStream<AssistantMessageEvent, AssistantMessage> {
  constructor() {
    super(
      (event) => event.type === "done" || event.type === "error",
      (event) => event.type === "done" ? event.message : event.error,
    );
  }
}
```

**使用方式**：
- 生产者：`stream.push({ type: "text_delta", ... })` → 最终 `stream.push({ type: "done", message })` 或 `stream.push({ type: "error", error })`
- 消费者：`for await (const event of stream) { ... }` 或 `await stream.result()`

---

## 4. lazyStream + lazyApi — 延迟加载核心

**源码**：`packages/ai/src/api/lazy.ts`

### 4.1 lazyStream()

```
lazy.ts → function lazyStream
偏移：+46
```

**签名**：
```typescript
function lazyStream(
  model: Model<Api>,
  setup: () => Promise<AsyncIterable<AssistantMessageEvent>>,
): AssistantMessageEventStream
```

**行为**：
1. 同步创建一个 `AssistantMessageEventStream`（`outer`）
2. 异步启动 `setup()`——认证解析、动态模块加载
3. `setup` 成功 → `forwardStream(outer, inner)`：把内层流的事件转发到外层
4. `setup` 失败 → 构造一个 `stopReason: "error"` 的 `AssistantMessage`，push error 事件 + end

**关键设计**：返回值是同步的 `AssistantMessageEventStream`，但实际的异步 setup 在背后运行。这满足 `StreamFn` 的契约——"一旦调用，请求/模型/运行时失败应编码在返回的流中，不抛异常"。

```
lazy.ts → function forwardStream
偏移：+31
```

```typescript
async function forwardStream(target, source): Promise<void> {
  for await (const event of source) {
    target.push(event);  // 逐事件转发
  }
  target.end(hasResult(source) ? await source.result() : undefined);
}
```

### 4.2 lazyApi()

```
lazy.ts → function lazyApi
偏移：+73
```

**签名**：
```typescript
function lazyApi(
  load: () => Promise<ProviderStreams>,
  capabilities?: LazyApiCapabilities,
): ProviderStreams
```

**行为**：返回一个 `ProviderStreams` 对象，其 `stream`/`streamSimple` 方法在第一次调用时才 `await load()` 加载实际的 API 实现模块。

```typescript
const api: ProviderStreams = {
  stream: (model, context, options) =>
    lazyStream(model, async () => (await load()).stream(model, context, options)),
  streamSimple: (model, context, options) =>
    lazyStream(model, async () => (await load()).streamSimple(model, context, options)),
};
```

**用途**：每个 API 协议模块（如 `openai-completions.lazy.ts`）导出一个通过 `lazyApi()` 包装的 `ProviderStreams`，实现按需加载——只有实际用到该 API 时才 import 实现代码，减少启动开销。

### 4.3 createSetupErrorMessage()

```
lazy.ts → function createSetupErrorMessage
偏移：+4
```

构造一个 `stopReason: "error"` 的 `AssistantMessage`，在 `lazyStream` 的 setup 失败时使用。确保错误也以标准 `AssistantMessage` 形式呈现，不破坏下游消费者。

---

## 5. Models — 运行时供应商集合

**源码**：`packages/ai/src/models.ts`

### 5.1 createModels()

```
models.ts → function createModels
偏移：+735
```

```typescript
function createModels(options?: CreateModelsOptions): MutableModels {
  return new ModelsImpl(options);
}
```

`CreateModelsOptions`：`credentials`（默认 `InMemoryCredentialStore`）、`modelsStore`（默认 `InMemoryModelsStore`）、`authContext`（默认 `defaultProviderAuthContext()`）。

### 5.2 ModelsImpl 核心方法

```
models.ts → class ModelsImpl
偏移：+254
```

#### streamSimple()

```
models.ts → ModelsImpl.streamSimple
偏移：+690
```

```typescript
streamSimple(model, context, options?): AssistantMessageEventStream {
  return lazyStream(model, async () => {
    const provider = this.requireProvider(model);
    const { requestModel, requestOptions } = await this.applyAuth(model, options);
    return provider.streamSimple(requestModel, context, requestOptions);
  });
}
```

**链路**：
1. `lazyStream` 同步返回 `AssistantMessageEventStream`
2. 异步：`requireProvider(model)` → 查 Map 找 Provider
3. 异步：`applyAuth(model, options)` → 解析认证 + 合并 headers
4. 异步：`provider.streamSimple(requestModel, context, requestOptions)` → 返回内层流
5. `forwardStream` 把内层事件转发到外层

#### applyAuth()

```
models.ts → ModelsImpl.applyAuth
偏移：+636
```

**职责**：合并认证信息到请求选项。

1. `requireProvider(model)`：确保 Provider 存在
2. `getAuth(model, { apiKey, env, signal })`：解析认证
3. 如果 `resolution` 为 `undefined` → 抛 `ModelsError("auth", "Provider is not configured")`
4. 合并 headers：`mergeHeaders(auth.headers, options?.headers)`，再 `options?.transformHeaders` 可做最终变换
5. 合并 env：`{ ...resolution.env, ...options?.env }`
6. 如果 `auth.baseUrl` 有值，覆写 `model.baseUrl`
7. 返回 `{ requestModel, requestOptions }`

#### getAuth()

```
models.ts → ModelsImpl.getAuth
偏移：+546
```

委托到 `resolveProviderAuth()`（见 §6.1）。如果是按 Model 调用且 model 有 headers，合并到结果中。

#### refresh()

```
models.ts → ModelsImpl.refresh
偏移：+386
```

**两阶段刷新**：
1. **离线恢复阶段**：对每个动态 Provider，先从 `modelsStore` 读取缓存条目，通过 `provider.refreshModels()` 的 `context.publish({ update })` 恢复内存状态——`allowNetwork = false`
2. **网络刷新阶段**：如果 `allowNetwork`，解析 OAuth/apiKey 凭据，再调一次 `provider.refreshModels()` 的网络路径

**并发控制**：
- `supersedeProviderRefresh()`：每次开始时 abort 上一次同 Provider 的刷新
- `publishProviderModels()`：publication 操作串行化（`publicationChains` promise chain），且校验 generation 号防止过期写入

#### login() / logout()

```
models.ts → ModelsImpl.login
偏移：+565
```

**login 链路**：
1. 查 Provider，获取 `provider.auth.oauth` 或 `provider.auth.apiKey` 的 `login` 方法
2. 调 `method.login(interaction)` → 返回 `Credential`
3. `this.credentials.modify(providerId, async () => credential)` 串行写入
4. 中间处理 abort：如果 abort 发生在 modify 开始前，reject；开始后等 modify 完成

### 5.3 createProvider()

```
models.ts → function createProvider
偏移：+762
```

**职责**：从 `CreateProviderOptions` 构建一个 `Provider` 实例。

**模型合并**：`currentModels()` 合并 baseline + dynamic，同 id 的 dynamic 覆写 baseline。

**API 分派**：
- 单一 `api`（`ProviderStreams`）：所有模型用同一组 stream 函数
- `api` map（`Partial<Record<TApi, ProviderStreams>>`）：按 `model.api` 查找

```typescript
const dispatch = (model, run) => {
  const streams = apiFor(model);
  if (!streams) return lazyStream(model, async () => {
    throw new ModelsError("stream", `Provider ${id} has no API for "${model.api}"`);
  });
  return run(streams);
};
```

### 5.4 hasApi() — 类型收窄

```
models.ts → function hasApi
偏移：+874
```

```typescript
function hasApi<TApi extends Api>(model: Model<Api>, api: TApi): model is Model<TApi> {
  return model.api === api;
}
```

运行时类型守卫：从 `Models.getModel()` 返回的 `Model<Api>` 收窄到具体 `Model<TApi>`，使 stream options 类型安全。

### 5.5 calculateCost()

```
models.ts → function calculateCost
偏移：+878
```

按 `model.cost` 和 tier 计算 token 成本。Anthropic 1h cache write 按 2x input 计费。

### 5.6 getSupportedThinkingLevels() / clampThinkingLevel()

```
models.ts → getSupportedThinkingLevels
偏移：+902

models.ts → clampThinkingLevel
偏移：+913
```

查 `model.thinkingLevelMap` 确定支持的思考级别。`clamp` 在请求级别不直接支持时找最近的可用级别。

---

## 6. 认证系统

### 6.1 resolveProviderAuth() — 认证解析核心

**源码**：`packages/ai/src/auth/resolve.ts`

```
resolve.ts → function resolveProviderAuth
偏移：+50
```

**解析优先级**（高 → 低）：

1. **Overrides apiKey**：如果调用方显式传了 `apiKey` 且 provider 有 `apiKey` auth，直接用它构造 `AuthResult`
2. **存储的凭据**：从 `CredentialStore.read()` 读取
   - OAuth 凭据 → `resolveStoredOAuth()`（含自动刷新）
   - API Key 凭据 → `resolveApiKey()`
3. **环境凭据**（Ambient）：如果无存储凭据且 provider 有 `apiKey` auth，调 `resolveApiKey(undefined)` 从环境变量/AWS profile/ADC 文件解析

**返回 `undefined`**：Provider 未配置（没有凭据也没有环境变量）。

```
resolve.ts → function resolveStoredOAuth
偏移：+127
```

**OAuth 双检锁刷新**：
1. 乐观检查：`Date.now() + 5min >= credential.expires`？
2. 如果快过期：`credentials.modify()` 获取锁
3. 锁内再检查：另一个请求可能已经刷新了
4. 如果仍快过期：`oauth.refresh(current, signal)` 调供应商刷新端点
5. 15 秒超时保护
6. 刷新失败 → `ModelsError("oauth")`
7. 刷新成功 → 返回新凭据，`modify` 回调返回新值，存储层自动持久化

### 6.2 InMemoryCredentialStore

**源码**：`packages/ai/src/auth/credential-store.ts`

```
credential-store.ts → class InMemoryCredentialStore
偏移：+9
```

**序列化写**：通过 `enqueue()` 方法，每个 provider 的写操作串行排队（promise chain），防止并发竞态。

```typescript
private enqueue<T>(providerId, task, options): Promise<T> {
  const previous = this.chains.get(providerId) ?? Promise.resolve();
  const queued = (async () => {
    await previous.catch(() => {});
    signal.throwIfAborted();
    return task();
  })();
  // ... chain management
}
```

**关键方法**：
- `read()`：直接返回 Map 中的值，无锁
- `modify()`：通过 `enqueue` 串行化，执行 `fn(current)` → 写回
- `delete()`：同上

### 6.3 defaultProviderAuthContext()

**源码**：`packages/ai/src/auth/context.ts`

```
context.ts → function defaultProviderAuthContext
偏移：+23
```

创建 `AuthContext`：
- `env(name)`：从 `process.env` 读，trim 后非空才返回
- `fileExists(path)`：用 `node:fs/promises.access()` 检查，支持 `~` 展开

**浏览器兼容**：`process` 通过 `globalThis` 访问，不存在时返回 `undefined`。`node:fs` 通过变量 specifier 动态 import，浏览器打包工具不解析。

### 6.4 envApiKeyAuth() — 标准 API Key 认证

**源码**：`packages/ai/src/auth/helpers.ts`

```
helpers.ts → function envApiKeyAuth
偏移：+9
```

```typescript
function envApiKeyAuth(name: string, envVars: readonly string[]): ApiKeyAuth {
  return {
    name,
    login: async (interaction) => {
      const key = await interaction.prompt({ type: "secret", message: `Enter ${name}` });
      return { type: "api_key", key };
    },
    resolve: async ({ ctx, credential, signal }) => {
      if (credential?.key) return { auth: { apiKey: credential.key }, ... };
      for (const envVar of envVars) {
        const value = await ctx.env(envVar);
        if (value) return { auth: { apiKey: value }, source: envVar };
      }
      return undefined;
    },
  };
}
```

**解析顺序**：存储的 key > 环境变量（按 `envVars` 数组顺序尝试）

### 6.5 lazyOAuth()

```
helpers.ts → function lazyOAuth
偏移：+40
```

包装动态导入的 `OAuthAuth`：login/refresh/toAuth 均在首次调用时加载实现。模块级 `promise` 缓存避免重复加载。

---

## 7. SimpleStreamOptions 构建工具

**源码**：`packages/ai/src/api/simple-options.ts`

### 7.1 buildBaseOptions()

```
simple-options.ts → function buildBaseOptions
偏移：+21
```

**职责**：从 `SimpleStreamOptions` 提取公共字段，构建 `StreamOptions`。

1. 合并 `model.samplingParams` 和 `options.samplingParams`（后者覆盖前者）
2. `maxTokens`：`clampMaxTokensToContext(model, context, options?.maxTokens ?? model.maxTokens)`
3. 透传 `temperature`, `signal`, `apiKey`, `fetch`, `transport`, `cacheRetention`, `sessionId`, `headers`, `onPayload`, `onResponse`, `timeoutMs`, ...

### 7.2 clampMaxTokensToContext()

```
simple-options.ts → function clampMaxTokensToContext
偏移：+15
```

```typescript
function clampMaxTokensToContext(model, context, maxTokens): number {
  const available = model.contextWindow - estimateContextTokens(context).tokens - 4096;
  return Math.min(maxTokens, Math.max(1, available));
}
```

**安全边距**：`CONTEXT_SAFETY_TOKENS = 4096`。确保 maxTokens 不超出上下文窗口减去输入 + 安全边距。

### 7.3 thinkingBudgetForLevel() / adjustMaxTokensForThinking()

```
simple-options.ts → thinkingBudgetForLevel
偏移：+68

simple-options.ts → adjustMaxTokensForThinking
偏移：+79
```

**默认思考预算**：
- minimal: 1024, low: 2048, medium: 8192, high: 16384

**调整逻辑**：
- 如果 caller 没给 `maxTokens`（`undefined`），用 `model.maxTokens` 并把思考预算放进去
- 如果 `maxTokens <= thinkingBudget`，压缩思考预算确保至少 `MIN_ANSWER_TOKENS = 1024` 给回答

---

## 8. 重试策略

**源码**：`packages/ai/src/utils/retry.ts`

### 8.1 isRetryableAssistantError()

```
retry.ts → function isRetryableAssistantError
偏移：+223
```

**分类逻辑**：
1. 如果 `stopReason !== "error"` 或无 `errorMessage` → 不可重试（非错误）
2. 如果匹配 `NON_RETRYABLE_PROVIDER_LIMIT_ERROR_PATTERN` → 不可重试（配额/计费耗尽）
3. 如果匹配 `RETRYABLE_PROVIDER_ERROR_PATTERN` → 可重试（瞬时错误）

**不可重试模式**（配额类）：
- `GoUsageLimitError`, `FreeUsageLimitError`
- `Monthly usage limit reached`, `available balance`
- `insufficient_quota`, `out of budget`, `quota exceeded`, `billing`

**可重试模式**（瞬时类）：
- HTTP: `429`, `500`, `502`, `503`, `504`, `524`, `overloaded`, `rate.?limit`, `service.?unavailable`
- 网络: `network.?error`, `connection.?refused`, `connection.?lost`, `fetch failed`, `ENOTFOUND`, `EAI_AGAIN`, `socket hang up`, `timed? out`
- 流: `stream ended before message_stop`, `ended without`, `http2 request did not get a response`
- WebSocket: `websocket.?closed`, `websocket.?error`
- 显式重试指引: `you can retry your request`, `try your request again`

### 8.2 retryAssistantCall()

```
retry.ts → function retryAssistantCall
偏移：+163
```

**签名**：
```typescript
async function retryAssistantCall(
  produce: () => Promise<AssistantMessage>,
  policy: RetryPolicy | undefined,
  signal: AbortSignal | undefined,
  callbacks?: RetryCallbacks,
): Promise<AssistantMessage>
```

**行为**：
1. `maxAttempts = policy?.enabled ? policy.maxRetries : 0`
2. 循环调 `produce()`
3. 成功（`stopReason !== "error"` 且 `!== "aborted"`）→ 立即返回
4. Abort → 终止，不重试
5. 不可重试错误 或 预算耗尽 → 返回最终错误
6. 可重试 → `attempt++`，计算 backoff `= baseDelayMs * 2^(attempt-1)`
7. `sleep(delayMs, signal)` 等待，可被 abort 中断
8. sleep 中 abort → 返回 `{ ...response, stopReason: "aborted" }`

**回调**：
- `onRetryScheduled(attempt, maxAttempts, delayMs, errorMessage)`：sleep 前
- `onRetryAttemptStart()`：sleep 后、重试调用前
- `onRetryFinished(success, attempt, finalError?)`：循环结束时

### 8.3 RetryPolicy

```
retry.ts → interface RetryPolicy
偏移：+98
```

```typescript
interface RetryPolicy {
  enabled: boolean;
  maxRetries: number;    // 0 = 不重试
  baseDelayMs: number;   // 基础延迟，实际 = base * 2^(attempt-1)
}
```

对应 coding-agent 的 `settings.retry`。

---

## 9. ModelRuntime — coding-agent 桥接层

**源码**：`packages/coding-agent/src/core/model-runtime.ts`

### 9.1 类结构

```
model-runtime.ts → class ModelRuntime
偏移：+130
```

`ModelRuntime` 实现了 `Models` 接口，包装 `pi-ai` 的 `MutableModels`，叠加：
- `RuntimeCredentials`：文件持久化 + 运行时 API Key 覆写
- `ModelConfig`：从 `models.json` 读取用户自定义 Provider
- `FileModelsStore`：文件持久化的模型目录缓存
- `builtinProviders()`：39 个内置供应商（通过 `withRemoteCatalog` 叠加远程目录）
- Provider 组合：`composeModelProvider()` 合并 builtin + config + extension

### 9.2 create() — 异步工厂

```
model-runtime.ts → ModelRuntime.create
偏移：+172
```

1. 创建 `RuntimeCredentials`（包装 `AuthStorage`）
2. 加载 `ModelConfig`（从 `~/.pi/models.json`）
3. 创建 `ModelsStore`（`FileModelsStore` 或 `InMemoryCodingAgentModelsStore`）
4. 获取 `builtinProviders()`，每个叠 `withRemoteCatalog`
5. 构造 `ModelRuntime` 实例
6. `configureRadiusProviders()`：根据 config 中的 Radius OAuth 配置重建 Radius Provider
7. `rebuildProviders()`：把所有 Provider 注入 pi-ai `Models`
8. 如果 `refreshOnCreate !== false`：调 `refresh()`（可能含网络刷新 + 超时控制）

### 9.3 streamSimple() — 调用链路

```
model-runtime.ts → ModelRuntime.streamSimple
偏移：+636
```

```typescript
streamSimple(model, context, options?): AssistantMessageEventStream {
  return lazyStream(model, async () => {
    const prepared = await this.prepareRequest(model, options);
    return prepared.provider.streamSimple(prepared.model, context, prepared.options);
  });
}
```

**prepareRequest()**（偏移 +573）：
1. 查 Provider
2. `getAuth(model, { apiKey, env, signal })` → 解析认证（含 OAuth 自动刷新）
3. 如果 `resolution` 为空 → `ModelsError("auth")`
4. `mergeHeaders(resolution.auth.headers, options.headers)` → `transformHeaders` 最终变换
5. 合并 env
6. 如果 `resolution.auth.baseUrl` → 覆写 `model.baseUrl`

### 9.4 可用性快照管理

`ModelRuntime` 维护一个 `ModelRuntimeSnapshot`：

```typescript
interface ModelRuntimeSnapshot {
  all: readonly Model<Api>[];
  available: readonly Model<Api>[];
  configuredProviders: ReadonlySet<string>;
  storedProviders: ReadonlySet<string>;
  auth: ReadonlyMap<string, AuthCheck | undefined>;
}
```

**刷新机制**：
- `queueAvailabilityRefresh()`：全量刷新——对所有 Provider 调 `getAvailable()` + `checkAuth()`，用 sequence number 防过期
- `refreshProviderAvailability(providerId)`：单 Provider 刷新——登录/登出后调
- `synchronizeCredentialState()`：凭据变更后重组 Provider + 刷新模型目录 + 更新可用性

### 9.5 login() / logout()

```
model-runtime.ts → ModelRuntime.login
偏移：+673
```

login 链路：
1. `enqueueCredentialOperation()` 串行化
2. `models.login(providerId, type, interaction)` → 拿到 `Credential`
3. `synchronizeCredentialState()` → 重组 Provider + 刷新快照
4. 返回 `Credential`

如果第 3 步失败，抛 `CredentialSynchronizationError`——凭据已存储但本地状态不同步，用户可手动 `pi refresh` 恢复。

---

## 10. 内置供应商注册表

**源码**：`packages/ai/src/providers/all.ts`

### 10.1 builtinProviders()

```
all.ts → function builtinProviders
偏移：+89
```

返回 39 个供应商实例数组，每个由对应的工厂函数创建：

| 供应商 | 工厂函数 | API 协议 |
|--------|----------|----------|
| amazon-bedrock | `amazonBedrockProvider()` | bedrock-converse-stream |
| anthropic | `anthropicProvider()` | anthropic-messages |
| openai | `openaiProvider()` | openai-responses, openai-completions |
| google | `googleProvider()` | google-generative-ai |
| google-vertex | `googleVertexProvider()` | google-vertex |
| openai-codex | `openaiCodexProvider()` | openai-codex-responses |
| azure-openai-responses | `azureOpenAIResponsesProvider()` | azure-openai-responses |
| github-copilot | `githubCopilotProvider()` | openai-responses |
| xai | `xaiProvider()` | openai-responses |
| groq | `groqProvider()` | openai-completions |
| mistral | `mistralProvider()` | mistral-conversations |
| openrouter | `openrouterProvider()` | openai-completions, openai-responses |
| deepseek | `deepseekProvider()` | openai-completions |
| ... | ... | ... |

### 10.2 getBuiltinModel() — 类型安全模型查找

```
all.ts → function getBuiltinModel
偏移：+61
```

```typescript
function getBuiltinModel<TProvider, TModelId>(provider, modelId): Model<BuiltinModelApi<...>> {
  const models = MODELS[provider];
  return models?.[modelId];
}
```

从 `models.generated.ts` 的代码生成目录中按类型安全方式查找模型。

### 10.3 getBuiltinModelDataGeneratedAt()

```
all.ts → function getBuiltinModelDataGeneratedAt
偏移：+74
```

从 `data/.manifest.json` 读取生成时间戳，用于 `withRemoteCatalog` 判断缓存是否需要更新。

---

## 11. 数据流总览

从 `Agent.runPromptMessages()` 到 LLM 响应返回的完整路径：

```
Agent.runPromptMessages()
  │
  ▼
streamFn(messages, model, options)          // sdk.ts 中创建的闭包
  │
  ▼
ModelRuntime.streamSimple(model, context, options)   // model-runtime.ts +636
  │
  ▼
lazyStream(model, async () => {             // lazy.ts +46
  │  prepareRequest(model, options)        // model-runtime.ts +573
  │    → getAuth(model)                    // models.ts +546
  │      → resolveProviderAuth()          // resolve.ts +50
  │        → CredentialStore.read()
  │        → resolveStoredOAuth()  (if needed)
  │        → resolveApiKey()
  │    → mergeHeaders(auth, options)
  │  provider.streamSimple(model, context, opts)  // models.ts +690 → provider dispatch
  │    → lazyApi().streamSimple()         // lazy.ts +77
  │      → (await load()).streamSimple()  // 实际 API 模块，如 openai-completions.ts
  │        → HTTP fetch → SSE parse
  │        → EventStream.push(events)
  │
  ▼
forwardStream(outer, inner)                 // lazy.ts +31
  │  for await (event of inner) outer.push(event)
  │  outer.end(await inner.result())
  ▼
Agent.streamAssistantResponse()            // agent-loop.ts +279
  │  for await (event of stream)
  │    → emit text_delta / thinking_delta / toolcall_end
  │    → accumulate into AssistantMessage
  ▼
runLoop() → executeToolCalls() or return
```

**错误传播**：
- 认证失败 → `lazyStream` catch → `push({ type: "error", error: createSetupErrorMessage() })` → 流以 error 终止
- HTTP 错误 → API 实现模块 push error event → `forwardStream` 转发 → Agent 看到 `stopReason: "error"`
- `retryAssistantCall` 包裹 `produce()` → 瞬时错误自动重试，预算耗尽直接返回

---

## 12. 关键设计决策

1. **同步返回流**：`streamSimple()` 同步返回 `AssistantMessageEventStream`，异步 setup（认证、模块加载）在背后通过 `lazyStream` 运行。消费者可以立即开始 `for await`，在 setup 完成前会自然等待。

2. **EventStream 双契约**：同时是 `AsyncIterable<T>`（供流式消费）和 `Promise<R>`（供 `result()` 等待最终结果）。无需额外包装。

3. **认证优先级链**：显式 override > 存储凭据 > 环境变量。无静默 env 回退——存储的凭据类型不匹配时返回 undefined 而非从 env 解析。

4. **OAuth 双检锁**：乐观检查 + `modify` 锁内再检查，防止并发请求重复刷新 token。15 秒超时保护防止刷新挂起。

5. **延迟加载**：`lazyApi()` + `lazyOAuth()` 确保只有实际用到的 API/OAuth 实现才会被 import，显著减少启动时间和内存。

6. **重试分类**：`isRetryableAssistantError` 用正则匹配 errorMessage 文本——覆盖了所有主流供应商的瞬时/永久错误模式，且不可重试的配额类错误优先检查。

7. **Provider 组合**：`ModelRuntime` 在 `pi-ai Models` 之上叠加 `composeModelProvider()`，让 `models.json` 用户配置可以覆写/叠加到内置 Provider 上，无需修改代码。
