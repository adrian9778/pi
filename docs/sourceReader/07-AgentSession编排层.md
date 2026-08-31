[上一篇](06-函数级源码解析-Agent引擎与循环.md) · [总目录](README.md) · [下一篇](08-函数级源码解析-工具系统.md)

# 07-函数级源码解析-AgentSession编排层

> **场景**：AgentSession 的 prompt、事件处理、auto-retry、compaction 逻辑
> **版本**：0.0.3（commit `853a80d26`）
> **源码基准**：当前 `main` 分支源码

## 1. 类：AgentSession

**源码**：`packages/coding-agent/src/core/agent-session.ts`
**职责**：Agent 与运行模式之间的编排层。管理事件中继、会话持久化、模型/思考级别切换、工具注册、扩展钩子、自动重试、自动压缩。

### 1.1 构造函数

**方法**：`AgentSession.constructor`
**偏移**：+385 ～ +411

```
偏移 +386: this.agent = config.agent
偏移 +387: this.sessionManager = config.sessionManager
偏移 +389: this._scopedModels = config.scopedModels ?? []
偏移 +391: this._customTools = config.customTools ?? []
偏移 +393: this._modelRuntime = config.modelRuntime
偏移 +403: this._unsubscribeAgent = this.agent.subscribe(this._handleAgentEvent)
偏移 +404: this._installAgentToolHooks()
偏移 +405: this._installAgentNextTurnRefresh()
偏移 +407: this._buildRuntime({ activeToolNames: this._initialActiveToolNames, includeAllExtensionTools: true })
```

## 2. 方法：prompt

**源码**：`packages/coding-agent/src/core/agent-session.ts`
**方法**：`AgentSession.prompt`
**偏移**：+1160 ～ +1318
**参数**：`text: string`, `options?: PromptOptions`
**职责**：用户提示词的完整前置处理

### 执行流程

```
偏移 +1161: expandPromptTemplates = options?.expandPromptTemplates ?? true

偏移 +1168: if (expandPromptTemplates && text.startsWith("/"))
  → _tryExecuteExtensionCommand(text) → 如果 handled, return

偏移 +1177: if (this._compactionAbortController !== undefined)
  → throw Error("Cannot submit while compaction in progress")

偏移 +1186: if (this._extensionRunner.hasHandlers("input"))
  → emitInput(text, images, source, streamingBehavior)
  → 如果 action === "handled" → return
  → 如果 action === "transform" → currentText = result.text

偏移 +1206: expandedText = this._expandSkillCommand(expandedText)
偏移 +1207: expandedText = expandPromptTemplate(expandedText, [...this.promptTemplates])

偏移 +1211: if (this.isStreaming)
  → if (!options.streamingBehavior) throw Error
  → if (followUp) → _queueFollowUp(expandedText, images)
  → else → _queueSteer(expandedText, images)
  → return

偏移 +1227: this._flushPendingBashMessages()
偏移 +1228: this._flushPendingCustomMessages()

偏移 +1231: if (!this.model) → throw Error("No model selected")

偏移 +1235: hasConfiguredAuth = this._modelRuntime.hasConfiguredAuth(provider)
  || await this._modelRuntime.checkAuth(provider)
偏移 +1238: if (!hasConfiguredAuth) → throw auth error

偏移 +1252: lastAssistant = this._findLastAssistantMessage()
偏移 +1253: if (lastAssistant) → await this._checkCompaction(lastAssistant, false)

偏移 +1261: userContent = [{ type: "text", text: expandedText }]
偏移 +1262: if (currentImages) → userContent.push(...currentImages)
偏移 +1265: messages = [{ role: "user", content: userContent, timestamp: Date.now() }]

偏移 +1272: for (msg of this._pendingNextTurnMessages) → messages.push(msg)
偏移 +1275: this._pendingNextTurnMessages = []

偏移 +1278: result = await this._extensionRunner.emitBeforeAgentStart(
  expandedText, currentImages, this._baseSystemPrompt, this._baseSystemPromptOptions)
偏移 +1285: if (result?.messages) → push custom messages
偏移 +1299: if (result?.systemPrompt !== undefined)
  → this._systemPromptOverride = result.systemPrompt
  → this.agent.state.systemPrompt = result.systemPrompt
  else → reset to base

偏移 +1317: await this._runAgentPrompt(messages)
```

### 关键分支

| 条件 | 行为 |
|------|------|
| `text.startsWith("/")` 且是扩展命令 | 立即执行扩展命令，不发 prompt |
| `this._compactionAbortController !== undefined` | 抛出错误 |
| `this.isStreaming` 且无 `streamingBehavior` | 抛出错误 |
| `this.isStreaming` 且 `streamingBehavior === "steer"` | `agent.steer(message)` |
| `this.isStreaming` 且 `streamingBehavior === "followUp"` | `agent.followUp(message)` |
| 无 model | 抛出错误 |
| 无 auth | 抛出错误 |

## 3. 方法：_runAgentPrompt

**源码**：同上
**方法**：`AgentSession._runAgentPrompt`
**偏移**：+1106 ～ +1119

```typescript
private async _runAgentPrompt(messages: AgentMessage | AgentMessage[]): Promise<void> {
  this._isAgentRunActive = true;
  try {
    await this.agent.prompt(messages);
    while (await this._handlePostAgentRun()) {
      await this.agent.continue();
    }
  } finally {
    this._systemPromptOverride = undefined;
    this._flushPendingBashMessages();
    this._flushPendingCustomMessages();
    await this._emitAgentSettled();
  }
}
```

**设计意图**：`agent.prompt()` 完成后，`_handlePostAgentRun()` 检查是否需要继续（auto-retry、compaction 恢复、followUp 队列）。如果需要，调用 `agent.continue()` 继续。循环直到不需要继续。

## 4. 方法：_handlePostAgentRun

**源码**：同上
**方法**：`AgentSession._handlePostAgentRun`
**偏移**：+1121 ～ +1149

```
偏移 +1122: msg = this._lastAssistantMessage
偏移 +1123: this._lastAssistantMessage = undefined
偏移 +1124: if (!msg) → return false

偏移 +1128: if (this._isRetryableError(msg) && await this._prepareRetry(msg))
  → return true  ← 触发 agent.continue() 重试

偏移 +1132: if (msg.stopReason === "error" && this._retryAttempt > 0)
  → emit("auto_retry_end", { success: false, attempt, finalError })
  → this._retryAttempt = 0

偏移 +1142: if (await this._checkCompaction(msg))
  → return true  ← 触发 agent.continue() 从 compaction 恢复

偏移 +1148: return this.agent.hasQueuedMessages()
  ← 检查 steering/followUp 队列
```

### 返回值决策表

| 条件 | 返回值 | 后续动作 |
|------|--------|---------|
| 无 lastAssistant | `false` | 结束 |
| 可重试错误且准备成功 | `true` | `agent.continue()` |
| 错误且已重试 | `false`（但先 emit retry_end） | 结束 |
| 需要压缩且成功 | `true` | `agent.continue()` |
| 有排队消息 | `true` | `agent.continue()` |
| 以上都不满足 | `false` | 结束 |

## 5. 方法：_handleAgentEvent

**源码**：同上
**方法**：`AgentSession._handleAgentEvent` (箭头函数)
**偏移**：+644 ～ +724
**职责**：Agent 事件的内部处理器 — 中继给外部 listener + 持久化

### 执行流程

```
偏移 +647: if (event.type === "message_start" && event.message.role === "user") {
  ← 从 steering/followUp 队列中移除已处理的消息
  messageText = contentText(event.message.content, "")
  → 查找 steeringMessages, 如果找到 → splice + emitQueueUpdate
  → 否则查找 followUpMessages, 如果找到 → splice + emitQueueUpdate
}

偏移 +668: await this._emitExtensionEvent(event)  ← 发射给扩展

偏移 +671: this._emit(event)  ← 发射给外部 listener
  (如果是 agent_end → 附加 willRetry 标志)

偏移 +674: if (event.type === "message_end") {
  if (message.role === "custom") → sessionManager.appendCustomMessageEntry(...)
  else if (user|assistant|toolResult) → sessionManager.appendMessage(event.message)  ← 持久化

  if (assistant) {
    this._lastAssistantMessage = event.message  ← 追踪用于 post-run
    if (stopReason !== "error" && "length") → this._overflowRecoveryAttempted = false
    if (stopReason !== "error" && this._retryAttempt > 0) {
      emit("auto_retry_end", { success: true, attempt: this._retryAttempt })
      this._retryAttempt = 0
    }
  }
}

偏移 +721: if (event.type === "turn_end") {
  this._flushPendingCustomMessages()  ← 刷新 context-only 消息
}
```

### 数据流

```
runAgentLoop emit(event)
  ↓
Agent.processEvents(event) → state 更新
  ↓
AgentSession._handleAgentEvent(event)
  ├── 移除已处理的 steering/followUp 消息
  ├── 发射给扩展
  ├── 发射给外部 listener (session.subscribe)
  └── 持久化到 SessionManager (message_end 时)
```

## 6. 方法：_installAgentToolHooks

**源码**：同上
**方法**：`AgentSession._installAgentToolHooks`
**偏移**：+487 ～ +541
**职责**：安装 `beforeToolCall` 和 `afterToolCall` 钩子到 Agent

### beforeToolCall（偏移 +488 ～ +507）

```typescript
this.agent.beforeToolCall = async ({ toolCall, args }) => {
  const runner = this._extensionRunner;
  if (!runner.hasHandlers("tool_call")) return undefined;
  return await runner.emitToolCall({
    type: "tool_call",
    toolName: toolCall.name,
    toolCallId: toolCall.id,
    input: args,
  });
};
```

### afterToolCall（偏移 +509 ～ +541）

```typescript
this.agent.afterToolCall = async ({ toolCall, args, result, isError }) => {
  const runner = this._extensionRunner;
  const hookResult = runner.hasHandlers("tool_result")
    ? await runner.emitToolResult({ type: "tool_result", toolName, toolCallId, input, content, details, isError, usage })
    : undefined;

  const content = hookResult?.content ?? result.content ?? [];
  const normalizedContent = await normalizeToolResultImages(content, { autoResizeImages });
  if (!hookResult && normalizedContent === content) return undefined;

  return {
    content: normalizedContent,
    details: hookResult?.details,
    isError: hookResult?.isError ?? isError,
    usage: hookResult?.usage,
  };
};
```

**设计意图**：钩子在构造时安装一次，但通过 `this._extensionRunner` 动态读取当前 runner，所以扩展重载时不需要重新安装钩子。

## 7. 方法：_installAgentNextTurnRefresh

**源码**：同上
**方法**：`AgentSession._installAgentNextTurnRefresh`
**偏移**：+562 ～ +584
**职责**：安装 `prepareNextTurnWithContext` 到 Agent，用于每轮前的 compaction 检查和 systemPrompt 刷新

```typescript
this.agent.prepareNextTurnWithContext = async (turn, signal) => {
  const context = await this._compactBeforeNextAssistantResponse(turn.context);
  const previousSnapshot = await previousPrepareNextTurnWithContext?.({ ...turn, context }, signal);

  return {
    ...previousSnapshot,
    context: {
      ...(previousSnapshot?.context ?? context),
      systemPrompt: this._systemPromptOverride ?? this._baseSystemPrompt,
      tools: this.agent.state.tools.slice(),
    },
    model: this.agent.state.model,
    thinkingLevel: this.agent.state.thinkingLevel,
  };
};
```

## 8. 方法：_compactBeforeNextAssistantResponse

**源码**：同上
**方法**：`AgentSession._compactBeforeNextAssistantResponse`
**偏移**：+543 ～ +560
**职责**：在下一轮 LLM 调用前检查是否需要自动压缩

```
偏移 +544: model = this.model
偏移 +545: settings = this.settingsManager.getCompactionSettings()
偏移 +547: if (!model || model.contextWindow <= 0 ||
  !shouldCompact(estimateContextTokens(messages).tokens, model.contextWindow, settings))
  → return context  ← 不需要压缩

偏移 +555: await this._runAutoCompaction("threshold", false)
偏移 +556: return { ...context, messages: this.agent.state.messages.slice() }
  ← 返回压缩后的新消息列表
```

## 9. 方法：_isRetryableError

**源码**：同上
**方法**：`AgentSession._isRetryableError`
**职责**：判断 assistant 消息的错误是否可重试

调用 `isRetryableAssistantError(msg)` — 来自 `pi-ai/compat`。

## 10. 方法：_prepareRetry

**源码**：同上
**方法**：`AgentSession._prepareRetry`
**职责**：准备重试 — 发射事件、等待延迟

```
1. settings = this.settingsManager.getRetrySettings()
2. if (!settings.enabled || this._retryAttempt >= settings.maxRetries) → return false
3. this._retryAttempt++
4. delayMs = 计算指数退避延迟
5. emit("auto_retry_start", { attempt, maxAttempts, delayMs, errorMessage })
6. await sleep(delayMs)
7. return true
```

## 11. 方法：_checkCompaction

**源码**：同上
**方法**：`AgentSession._checkCompaction`
**职责**：检查是否需要执行 compaction

```
1. model = this.model
2. settings = this.settingsManager.getCompactionSettings()
3. if (isContextOverflow(msg) && !this._overflowRecoveryAttempted)
  → this._overflowRecoveryAttempted = true
  → await this._runAutoCompaction("overflow", false)
  → return true  ← 触发 continue
4. if (shouldCompact(estimateContextTokens(messages).tokens, model.contextWindow, settings))
  → await this._runAutoCompaction("threshold", false)
  → return true
5. return false
```

## 12. 方法：_runAutoCompaction

**源码**：同上
**方法**：`AgentSession._runAutoCompaction`
**职责**：执行自动上下文压缩

```
1. emit("compaction_start", { reason })
2. this._autoCompactionAbortController = new AbortController()
3. try:
   const result = await compact(messages, {
     model, streamFn, systemPrompt, context, signal, ...
   })
   if (result) → 替换 agent.state.messages
   emit("compaction_end", { reason, result, aborted: false })
4. catch (if aborted):
   emit("compaction_end", { reason, result: undefined, aborted: true })
5. finally:
   this._autoCompactionAbortController = undefined
```

## 13. 方法：subscribe

**源码**：同上
**方法**：`AgentSession.subscribe`
**职责**：外部事件订阅

```typescript
subscribe(listener: AgentSessionEventListener): () => void {
  this._eventListeners.push(listener);
  return () => {
    const index = this._eventListeners.indexOf(listener);
    if (index >= 0) this._eventListeners.splice(index, 1);
  };
}
```

## 14. 方法：setModel

**源码**：同上
**方法**：`AgentSession.setModel`
**职责**：切换当前模型

```
1. this.agent.state.model = model
2. sessionManager.appendModelChange(model.provider, model.id)  ← 持久化
3. clampThinkingLevel(model, this.agent.state.thinkingLevel)
4. emit("thinking_level_changed" if changed)
```

## 15. 事件传播总图

```
                    runAgentLoop emit()
                          │
                    Agent.processEvents()
                     ├── state 更新
                     │
                _handleAgentEvent()
                 ├── steering/followUp 移除
                 ├── _emitExtensionEvent()
                 ├── _emit() → 外部 listener
                 └── sessionManager 持久化 (message_end)
                          │
                    外部 listener
                 ├── InteractiveMode → TUI 渲染
                 ├── PrintMode → stdout 输出
                 └── RpcMode → JSON-RPC 输出
```

---

[上一篇](06-函数级源码解析-Agent引擎与循环.md) · [总目录](README.md) · [下一篇](08-函数级源码解析-工具系统.md)
