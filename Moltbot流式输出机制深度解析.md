# Moltbot流式输出机制深度解析

> **目标**：深入解析Moltbot如何实现完整、精细的流式输出系统  
> **版本**: v1.0 | **日期**: 2026-01-28

---

## 🎯 核心架构概览

Moltbot的流式输出系统是一个**多层、多通道、智能分块**的复杂系统：

```
┌─────────────────────────────────────────────────────────┐
│              Moltbot流式输出架构                         │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. 事件源层（Agent Event Source）                      │
│     └─ Pi Agent Core → 原始事件流                       │
│                                                          │
│  2. 订阅处理层（Subscribe Handler）                     │
│     ├─ 事件分发与路由                                    │
│     ├─ 状态管理（40+字段）                              │
│     └─ 缓冲区管理                                        │
│                                                          │
│  3. 内容处理层（Content Processor）                     │
│     ├─ 思考链提取（<think>...</think>）                │
│     ├─ Markdown解析与保护                               │
│     ├─ 指令解析（Reply Directives）                     │
│     └─ 去重与规范化                                      │
│                                                          │
│  4. 分块策略层（Chunking Strategy）                     │
│     ├─ 智能分块（段落/换行/句子）                       │
│     ├─ Fence保护（代码块不截断）                        │
│     └─ 动态缓冲（min/max阈值）                          │
│                                                          │
│  5. 多通道输出层（Multi-Channel Output）                │
│     ├─ Reasoning Stream（推理过程）                     │
│     ├─ Assistant Stream（助手回复）                     │
│     ├─ Tool Result Stream（工具结果）                   │
│     ├─ Block Reply Stream（分块回复）                   │
│     └─ Partial Reply Stream（部分回复）                 │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

---

## 一、事件源层：Pi Agent Core事件流

### 1.1 核心事件类型

```typescript
// Agent执行过程中产生的事件流

type AgentEvent = 
  | { type: "message_start"; message: AgentMessage }
  | { type: "message_update"; message: AgentMessage; assistantMessageEvent: AssistantEvent }
  | { type: "message_end"; message: AgentMessage }
  | { type: "tool_call_begin"; toolCall: ToolCall }
  | { type: "tool_result"; toolCallId: string; result: unknown }
  | { type: "error"; error: Error };

type AssistantEvent =
  | { type: "text_start"; content: string }
  | { type: "text_delta"; delta: string }
  | { type: "text_end"; content: string };
```

**事件流示例**：

```
用户: "分析这个文件的内容"
    │
    ▼
┌─────────────────────────┐
│ message_start           │ ← Agent开始响应
│ role: "assistant"       │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ message_update          │ ← 流式文本开始
│ assistantEvent:         │
│   type: "text_start"    │
│   content: ""           │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ message_update          │ ← 逐字流式输出
│ assistantEvent:         │
│   type: "text_delta"    │
│   delta: "我"           │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ message_update          │
│ assistantEvent:         │
│   type: "text_delta"    │
│   delta: "正在"         │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ message_update          │
│ assistantEvent:         │
│   type: "text_delta"    │
│   delta: "分析"         │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ tool_call_begin         │ ← Agent决定调用工具
│ toolName: "read"        │
│ toolCallId: "call_123"  │
│ args: { path: "/..." } │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ tool_result             │ ← 工具执行完成
│ toolCallId: "call_123"  │
│ result: "文件内容..."   │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ message_update          │ ← 继续流式输出
│ assistantEvent:         │
│   type: "text_delta"    │
│   delta: "文件包含..."  │
└─────────────────────────┘
    │
    ▼
┌─────────────────────────┐
│ message_end             │ ← Agent完成响应
│ message: {              │
│   role: "assistant",    │
│   content: "完整文本"   │
│ }                       │
└─────────────────────────┘
```

---

## 二、订阅处理层：状态管理与事件分发

### 2.1 核心订阅函数

```typescript
// src/agents/pi-embedded-subscribe.ts

export function subscribeEmbeddedPiSession(params: SubscribeEmbeddedPiSessionParams) {
  // ===== 1. 初始化状态 =====
  const state: EmbeddedPiSubscribeState = {
    // 累积文本
    assistantTexts: [],                    // 最终助手文本数组
    
    // 工具元数据
    toolMetas: [],                         // 工具执行元数据
    toolMetaById: new Map(),               // 按ID索引的工具元数据
    toolSummaryById: new Set(),            // 已发送摘要的工具ID
    lastToolError: undefined,              // 最后的工具错误
    
    // 流式配置
    blockReplyBreak: "text_end",           // 分块回复边界（"text_end" | "message_end"）
    reasoningMode: "off",                  // 推理模式（"off" | "on" | "stream"）
    includeReasoning: false,               // 是否包含推理内容
    shouldEmitPartialReplies: true,        // 是否发送部分回复
    streamReasoning: false,                // 是否流式输出推理
    
    // 缓冲区
    deltaBuffer: "",                       // 文本增量缓冲区
    blockBuffer: "",                       // 块缓冲区
    
    // 块状态跟踪
    blockState: {
      thinking: false,                     // 是否在<think>块内
      final: false,                        // 是否在<final>块内
      inlineCode: createInlineCodeState(), // 行内代码状态
    },
    
    // 流式状态跟踪
    lastStreamedAssistant: undefined,      // 最后流式输出的助手文本
    lastStreamedReasoning: undefined,      // 最后流式输出的推理文本
    lastBlockReplyText: undefined,         // 最后的块回复文本
    
    // 消息索引
    assistantMessageIndex: 0,              // 当前助手消息索引
    lastAssistantTextMessageIndex: -1,     // 最后助手文本的消息索引
    lastAssistantTextNormalized: undefined,// 规范化后的最后文本
    lastAssistantTextTrimmed: undefined,   // 修剪后的最后文本
    assistantTextBaseline: 0,              // 助手文本基线
    suppressBlockChunks: false,            // 抑制块分片
    lastReasoningSent: undefined,          // 最后发送的推理内容
    
    // 压缩状态
    compactionInFlight: false,             // 是否正在压缩
    pendingCompactionRetry: 0,             // 待处理的压缩重试次数
    compactionRetryResolve: undefined,     // 压缩重试Promise的resolve
    compactionRetryPromise: null,          // 压缩重试Promise
    
    // 消息工具去重
    messagingToolSentTexts: [],            // 消息工具发送的文本
    messagingToolSentTextsNormalized: [], // 规范化后的消息工具文本
    messagingToolSentTargets: [],          // 消息工具发送目标
    pendingMessagingTexts: new Map(),      // 待处理的消息文本
    pendingMessagingTargets: new Map(),    // 待处理的消息目标
  };
  
  // ===== 2. 创建分块器（可选） =====
  const blockChunker = params.blockReplyChunking
    ? new EmbeddedBlockChunker(params.blockReplyChunking)
    : null;
  
  // ===== 3. 创建上下文 =====
  const ctx: EmbeddedPiSubscribeContext = {
    params,
    state,
    blockChunker,
    // ... 各种辅助函数
  };
  
  // ===== 4. 订阅事件流 =====
  const eventHandler = createEmbeddedPiSessionEventHandler(ctx);
  
  for await (const evt of params.session.stream()) {
    eventHandler(evt);
  }
  
  return {
    assistantTexts: state.assistantTexts,
    toolMetas: state.toolMetas,
    messagingToolSentTexts: state.messagingToolSentTexts,
    messagingToolSentTargets: state.messagingToolSentTargets,
  };
}
```

---

### 2.2 事件处理器分发

```typescript
// src/agents/pi-embedded-subscribe.handlers.ts

export function createEmbeddedPiSessionEventHandler(
  ctx: EmbeddedPiSubscribeContext
) {
  return (evt: AgentEvent) => {
    const evtType = evt.type;
    
    // ===== 消息事件 =====
    if (evtType === "message_start") {
      handleMessageStart(ctx, evt);
    }
    else if (evtType === "message_update") {
      handleMessageUpdate(ctx, evt);
    }
    else if (evtType === "message_end") {
      handleMessageEnd(ctx, evt);
    }
    
    // ===== 工具事件 =====
    else if (evtType === "tool_call_begin") {
      handleToolCallBegin(ctx, evt);
    }
    else if (evtType === "tool_result") {
      handleToolResult(ctx, evt);
    }
    
    // ===== 生命周期事件 =====
    else if (evtType === "run_start") {
      handleRunStart(ctx, evt);
    }
    else if (evtType === "run_end") {
      handleRunEnd(ctx, evt);
    }
    
    // ===== 其他事件 =====
    else {
      // 转发未知事件
      void ctx.params.onAgentEvent?.(evt);
    }
  };
}
```

---

## 三、内容处理层：智能解析与提取

### 3.1 思考链提取（Thinking Extraction）

Moltbot支持**3种推理模式**：

```typescript
type ReasoningLevel = "off" | "on" | "stream";
```

**推理模式对比**：

| 模式 | 行为 | 用户体验 | 应用场景 |
|------|------|---------|----------|
| `off` | 不显示推理过程 | 仅看到最终答案 | 快速问答 |
| `on` | 推理完成后一次性显示 | 先看推理，再看答案 | 需要理解思考过程 |
| `stream` | 实时流式输出推理 | 实时看到Agent思考 | 复杂推理，增强可解释性 |

**思考链格式**：

```markdown
<think>
我需要先分析文件内容...
让我读取文件看看...
</think>

<!-- 提取后的推理内容 -->
💭 **推理过程**

我需要先分析文件内容...
让我读取文件看看...
```

**提取代码**：

```typescript
// src/agents/pi-embedded-utils.ts

const THINKING_TAG_RE = /<think(?:ing)?>([\s\S]*?)<\/think(?:ing)?>/gi;

export function extractThinkingFromTaggedText(text: string): string {
  const matches = Array.from(text.matchAll(THINKING_TAG_RE));
  return matches.map(m => m[1]?.trim()).filter(Boolean).join("\n\n");
}

export function extractThinkingFromTaggedStream(stream: string): string {
  // 处理部分标签：<think>部分内容（未闭合）
  // 支持流式输出过程中实时提取
  
  const openMatch = stream.match(/<think(?:ing)?>([^<]*?)$/i);
  if (openMatch) {
    return openMatch[1] || "";
  }
  
  // 完整标签
  const completeMatches = Array.from(stream.matchAll(THINKING_TAG_RE));
  return completeMatches.map(m => m[1]?.trim()).filter(Boolean).join("\n\n");
}

export function formatReasoningMessage(thinking: string): string {
  if (!thinking) return "";
  return `💭 **推理过程**\n\n${thinking.trim()}`;
}
```

---

### 3.2 标签剥离（Tag Stripping）

```typescript
const THINKING_TAG_SCAN_RE = /<\s*(\/?)\s*(?:think(?:ing)?|thought|antthinking)\s*>/gi;
const FINAL_TAG_SCAN_RE = /<\s*(\/?)\s*final\s*>/gi;

function stripBlockTags(
  text: string,
  state: { thinking: boolean; final: boolean; inlineCode: InlineCodeState }
): string {
  let result = "";
  let lastIdx = 0;
  
  // 构建代码块索引（保护行内代码和代码块）
  const codeSpanIndex = buildCodeSpanIndex(text);
  
  // 扫描思考标签
  for (const match of text.matchAll(THINKING_TAG_SCAN_RE)) {
    const idx = match.index ?? 0;
    
    // 检查是否在代码块内
    if (isInsideCodeSpan(codeSpanIndex, idx)) {
      continue;  // 跳过代码块内的标签
    }
    
    // 添加标签前的文本
    if (!state.thinking) {
      result += text.slice(lastIdx, idx);
    }
    
    // 更新状态
    const isClosing = match[1] === "/";
    state.thinking = !isClosing;
    
    lastIdx = idx + match[0].length;
  }
  
  // 添加剩余文本
  if (!state.thinking) {
    result += text.slice(lastIdx);
  }
  
  return result;
}
```

---

### 3.3 Reply Directives解析

Moltbot支持在回复中嵌入**指令**：

```markdown
<!-- 指令格式 -->
@media(https://example.com/image.png)
@audio-voice
@reply-to(message-id-123)
@reply-tag

<!-- 解析后 -->
{
  text: "这是回复内容",
  mediaUrls: ["https://example.com/image.png"],
  audioAsVoice: true,
  replyToId: "message-id-123",
  replyToTag: true,
}
```

**解析代码**：

```typescript
// src/auto-reply/reply/reply-directives.ts

const DIRECTIVE_RE = /@(media|audio-voice|reply-to|reply-tag)(?:\(([^)]+)\))?/g;

export function parseReplyDirectives(text: string): {
  text: string;
  mediaUrls?: string[];
  audioAsVoice?: boolean;
  replyToId?: string;
  replyToTag?: boolean;
} {
  const directives = {
    mediaUrls: [] as string[],
    audioAsVoice: false,
    replyToId: undefined as string | undefined,
    replyToTag: false,
  };
  
  let cleanedText = text;
  
  for (const match of text.matchAll(DIRECTIVE_RE)) {
    const directive = match[1];
    const arg = match[2];
    
    if (directive === "media" && arg) {
      directives.mediaUrls.push(arg);
    } else if (directive === "audio-voice") {
      directives.audioAsVoice = true;
    } else if (directive === "reply-to" && arg) {
      directives.replyToId = arg;
    } else if (directive === "reply-tag") {
      directives.replyToTag = true;
    }
    
    // 从文本中移除指令
    cleanedText = cleanedText.replace(match[0], "");
  }
  
  return {
    text: cleanedText.trim(),
    ...directives,
  };
}
```

---

### 3.4 文本去重与规范化

```typescript
// src/agents/pi-embedded-helpers.ts

export function normalizeTextForComparison(text: string): string {
  return text
    .trim()
    .toLowerCase()
    .replace(/\s+/g, " ")           // 多个空格合并为1个
    .replace(/[^\w\s]/g, "");       // 移除标点符号
}

// 去重逻辑
const shouldSkipAssistantText = (text: string): boolean => {
  // 检查是否是相同消息的重复
  if (state.lastAssistantTextMessageIndex !== state.assistantMessageIndex) {
    return false;
  }
  
  // 检查修剪后的文本是否相同
  const trimmed = text.trimEnd();
  if (trimmed && trimmed === state.lastAssistantTextTrimmed) {
    return true;
  }
  
  // 检查规范化后的文本是否相同
  const normalized = normalizeTextForComparison(text);
  if (normalized.length > 0 && normalized === state.lastAssistantTextNormalized) {
    return true;
  }
  
  return false;
};

const pushAssistantText = (text: string) => {
  if (!text) return;
  if (shouldSkipAssistantText(text)) return;  // 跳过重复
  
  assistantTexts.push(text);
  rememberAssistantText(text);
};
```

---

## 四、分块策略层：智能Chunking

### 4.1 EmbeddedBlockChunker核心

```typescript
// src/agents/pi-embedded-block-chunker.ts

export type BlockReplyChunking = {
  minChars: number;        // 最小字符数（触发分块）
  maxChars: number;        // 最大字符数（强制分块）
  breakPreference?: "paragraph" | "newline" | "sentence";  // 分块偏好
};

export class EmbeddedBlockChunker {
  #buffer = "";
  readonly #chunking: BlockReplyChunking;
  
  constructor(chunking: BlockReplyChunking) {
    this.#chunking = chunking;
  }
  
  append(text: string) {
    this.#buffer += text;
  }
  
  drain(params: { force: boolean; emit: (chunk: string) => void }) {
    const { force, emit } = params;
    const minChars = this.#chunking.minChars;
    const maxChars = this.#chunking.maxChars;
    
    // 1. 不足最小字符数，且非强制 → 不分块
    if (this.#buffer.length < minChars && !force) return;
    
    // 2. 强制且不超过最大字符数 → 全部输出
    if (force && this.#buffer.length <= maxChars) {
      if (this.#buffer.trim().length > 0) {
        emit(this.#buffer);
      }
      this.#buffer = "";
      return;
    }
    
    // 3. 循环分块
    while (this.#buffer.length >= minChars || (force && this.#buffer.length > 0)) {
      const breakResult = this.#pickBreakIndex(this.#buffer, force ? 1 : undefined);
      
      if (breakResult.index <= 0) {
        if (force) {
          emit(this.#buffer);
          this.#buffer = "";
        }
        return;
      }
      
      const breakIdx = breakResult.index;
      let chunk = this.#buffer.slice(0, breakIdx);
      
      // 跳过空白块
      if (chunk.trim().length === 0) {
        this.#buffer = this.#buffer.slice(breakIdx).trimStart();
        continue;
      }
      
      let nextBuffer = this.#buffer.slice(breakIdx);
      
      // 4. 处理代码块截断（Fence Split）
      const fenceSplit = breakResult.fenceSplit;
      if (fenceSplit) {
        // 关闭当前代码块
        chunk = `${chunk}\n${fenceSplit.closeFenceLine}\n`;
        
        // 下一个块重新打开代码块
        nextBuffer = `${fenceSplit.reopenFenceLine}\n${nextBuffer}`;
      }
      
      emit(chunk);
      this.#buffer = nextBuffer;
      
      // 5. 检查是否继续分块
      if (this.#buffer.length < minChars && !force) return;
      if (this.#buffer.length < maxChars && !force) return;
    }
  }
  
  #pickBreakIndex(buffer: string, minCharsOverride?: number): BreakResult {
    const minChars = minCharsOverride ?? this.#chunking.minChars;
    const maxChars = this.#chunking.maxChars;
    const preference = this.#chunking.breakPreference ?? "paragraph";
    
    // 解析代码块边界（保护代码块）
    const fenceSpans = parseFenceSpans(buffer);
    
    // ===== 策略1：段落边界（\n\n） =====
    if (preference === "paragraph") {
      let paragraphIdx = buffer.indexOf("\n\n");
      while (paragraphIdx !== -1) {
        if (paragraphIdx >= minChars && paragraphIdx <= maxChars) {
          // 检查是否在代码块内
          if (isSafeFenceBreak(fenceSpans, paragraphIdx)) {
            return { index: paragraphIdx };
          }
        }
        paragraphIdx = buffer.indexOf("\n\n", paragraphIdx + 2);
      }
    }
    
    // ===== 策略2：换行边界（\n） =====
    if (preference === "paragraph" || preference === "newline") {
      let newlineIdx = buffer.indexOf("\n");
      while (newlineIdx !== -1) {
        if (newlineIdx >= minChars && newlineIdx <= maxChars) {
          if (isSafeFenceBreak(fenceSpans, newlineIdx)) {
            return { index: newlineIdx };
          }
        }
        newlineIdx = buffer.indexOf("\n", newlineIdx + 1);
      }
    }
    
    // ===== 策略3：句子边界（.!?） =====
    if (preference !== "newline") {
      const sentenceRe = /[.!?](?=\s|$)/g;
      for (const match of buffer.matchAll(sentenceRe)) {
        const idx = match.index ?? -1;
        if (idx >= minChars && idx <= maxChars) {
          const candidate = idx + 1;
          if (isSafeFenceBreak(fenceSpans, candidate)) {
            return { index: candidate };
          }
        }
      }
    }
    
    // ===== 策略4：强制分块（超过maxChars） =====
    if (buffer.length > maxChars) {
      // 在maxChars处截断，但需要处理代码块
      const fenceSpan = findFenceSpanAt(fenceSpans, maxChars);
      if (fenceSpan) {
        // 在代码块内，需要闭合并重新打开
        return {
          index: maxChars,
          fenceSplit: {
            closeFenceLine: "```",
            reopenFenceLine: fenceSpan.openLine,  // 保留原始fence信息
          },
        };
      }
      return { index: maxChars };
    }
    
    return { index: -1 };
  }
}
```

---

### 4.2 Fence保护机制

**问题**：如果在代码块中间截断，Markdown渲染会出错：

```markdown
<!-- 错误的截断 -->
块1:
```python
def hello():
    print("Hello")

块2:
    print("World")
```

<!-- 正确的处理 -->
块1:
```python
def hello():
    print("Hello")
```

块2:
```python
    print("World")
```
```

**Fence解析代码**：

```typescript
// src/markdown/fences.ts

type FenceSpan = {
  start: number;
  end: number;
  openLine: string;  // "```python" 或 "~~~javascript"
};

export function parseFenceSpans(text: string): FenceSpan[] {
  const spans: FenceSpan[] = [];
  const lines = text.split("\n");
  let currentFence: { start: number; openLine: string } | null = null;
  let offset = 0;
  
  for (let i = 0; i < lines.length; i++) {
    const line = lines[i];
    const fenceMatch = line.match(/^(\s*)(```+|~~~+)(\w*)/);
    
    if (fenceMatch) {
      if (!currentFence) {
        // 打开代码块
        currentFence = {
          start: offset,
          openLine: line,
        };
      } else {
        // 关闭代码块
        const endOffset = offset + line.length;
        spans.push({
          start: currentFence.start,
          end: endOffset,
          openLine: currentFence.openLine,
        });
        currentFence = null;
      }
    }
    
    offset += line.length + 1;  // +1 for \n
  }
  
  return spans;
}

export function isSafeFenceBreak(spans: FenceSpan[], index: number): boolean {
  // 检查index是否在任何fence span内
  for (const span of spans) {
    if (index > span.start && index < span.end) {
      return false;  // 在代码块内，不安全
    }
  }
  return true;
}

export function findFenceSpanAt(spans: FenceSpan[], index: number): FenceSpan | null {
  for (const span of spans) {
    if (index > span.start && index < span.end) {
      return span;
    }
  }
  return null;
}
```

---

## 五、多通道输出层：5种流式输出

### 5.1 通道1：Reasoning Stream（推理流）

**触发条件**：`reasoningMode === "stream"`

```typescript
// 实时流式输出推理过程

if (ctx.state.streamReasoning) {
  // 从流式文本中提取思考内容
  const thinking = extractThinkingFromTaggedStream(ctx.state.deltaBuffer);
  
  if (thinking && thinking !== ctx.state.lastStreamedReasoning) {
    ctx.state.lastStreamedReasoning = thinking;
    
    // 发送推理流
    void ctx.params.onReasoningStream?.({
      text: formatReasoningMessage(thinking),
    });
  }
}
```

**用户体验**：

```
💭 **推理过程**

我需要先分析文件内容...
[实时更新]
让我读取文件看看...
[实时更新]
读取完成，开始分析...
[实时更新]
```

---

### 5.2 通道2：Assistant Stream（助手流）

**触发条件**：总是触发（去除<think>标签后的内容）

```typescript
// 去除思考标签，发送助手回复

const cleanText = stripBlockTags(ctx.state.deltaBuffer, {
  thinking: false,
  final: false,
  inlineCode: createInlineCodeState(),
}).trim();

if (cleanText && cleanText !== ctx.state.lastStreamedAssistant) {
  const { text: parsedText, mediaUrls } = parseReplyDirectives(cleanText);
  const previousText = ctx.state.lastStreamedAssistant ?? "";
  const { text: previousParsedText } = parseReplyDirectives(previousText);
  
  // 计算增量
  if (parsedText.startsWith(previousParsedText)) {
    const delta = parsedText.slice(previousParsedText.length);
    
    ctx.state.lastStreamedAssistant = cleanText;
    
    // 发送到前端（SSE/WebSocket）
    emitAgentEvent({
      runId: ctx.params.runId,
      stream: "assistant",
      data: {
        text: parsedText,         // 完整文本
        delta: delta,             // 增量文本
        mediaUrls: mediaUrls,     // 媒体URL
      },
    });
    
    // 调用回调
    void ctx.params.onAgentEvent?.({
      stream: "assistant",
      data: { text: parsedText, delta, mediaUrls },
    });
    
    // Partial Reply（可选）
    if (ctx.params.onPartialReply && ctx.state.shouldEmitPartialReplies) {
      void ctx.params.onPartialReply({
        text: parsedText,
        mediaUrls: mediaUrls,
      });
    }
  }
}
```

**前端渲染**：

```typescript
// 前端接收增量更新
eventSource.addEventListener("assistant", (event) => {
  const data = JSON.parse(event.data);
  
  // 方式1：累加增量
  assistantText += data.delta;
  
  // 方式2：直接使用完整文本
  assistantText = data.text;
  
  // 更新UI
  renderMarkdown(assistantText);
});
```

---

### 5.3 通道3：Tool Result Stream（工具结果流）

**触发时机**：工具执行完成

```typescript
// src/agents/pi-embedded-subscribe.handlers.tools.ts

export function handleToolResult(
  ctx: EmbeddedPiSubscribeContext,
  evt: AgentEvent & { toolCallId: string; result: unknown }
) {
  const toolCallId = evt.toolCallId;
  const result = evt.result;
  
  // 1. 提取工具元数据
  const toolMeta = extractToolMeta(result);
  const toolName = ctx.state.toolMetaById.get(toolCallId);
  
  // 2. 记录元数据
  if (toolMeta) {
    ctx.state.toolMetas.push({ toolName, meta: toolMeta });
    ctx.state.toolMetaById.set(toolCallId, toolMeta);
  }
  
  // 3. 检查是否发送工具结果
  if (ctx.shouldEmitToolResult()) {
    const formattedResult = formatToolResult(result, {
      format: ctx.params.toolResultFormat,  // "markdown" | "plain"
      toolName,
    });
    
    // 4. 发送工具结果
    void ctx.params.onToolResult?.({
      text: formattedResult,
    });
    
    // 5. 标记已发送
    ctx.state.toolSummaryById.add(toolCallId);
  }
  
  // 6. 刷新块回复缓冲（在工具执行前发送已缓冲内容）
  void ctx.params.onBlockReplyFlush?.();
}
```

**格式化示例**：

```markdown
<!-- Markdown格式 -->
🔧 **read** (`/path/to/file.txt`)
读取成功，1024字节

<!-- Plain格式 -->
[read] /path/to/file.txt: 读取成功，1024字节
```

---

### 5.4 通道4：Block Reply Stream（分块回复流）

**触发时机**：
- `blockReplyBreak === "text_end"`：每个text_end事件
- `blockReplyBreak === "message_end"`：message_end事件

```typescript
// 分块发送助手回复

const emitBlockChunk = (chunk: string) => {
  if (!chunk || chunk.trim().length === 0) return;
  if (ctx.state.suppressBlockChunks) return;
  
  // 解析指令
  const { text, ...directives } = parseReplyDirectives(chunk);
  
  // 检查是否重复（消息工具去重）
  const normalized = normalizeTextForComparison(text);
  if (isMessagingToolDuplicateNormalized(
    normalized,
    ctx.state.messagingToolSentTextsNormalized
  )) {
    return;  // 跳过重复内容
  }
  
  ctx.state.lastBlockReplyText = text;
  
  // 发送块回复
  void ctx.params.onBlockReply?.({
    text,
    ...directives,
  });
};

// 使用分块器
if (ctx.blockChunker) {
  // 追加文本
  ctx.blockChunker.append(chunk);
  
  // 排空缓冲区（根据策略）
  ctx.blockChunker.drain({
    force: false,       // 非强制，等待达到阈值
    emit: emitBlockChunk,
  });
}
```

**分块示例**：

```
配置: { minChars: 200, maxChars: 500, breakPreference: "paragraph" }

输入文本:
"这是第一段内容，包含一些描述。\n\n这是第二段内容，描述更多细节。\n\n这是第三段..."

输出块:
块1: "这是第一段内容，包含一些描述。"
块2: "这是第二段内容，描述更多细节。"
块3: "这是第三段..."
```

---

### 5.5 通道5：Partial Reply Stream（部分回复流）

**触发条件**：`onPartialReply` 回调存在 且 `shouldEmitPartialReplies === true`

```typescript
// 部分回复：去除思考标签的流式文本

if (ctx.params.onPartialReply && ctx.state.shouldEmitPartialReplies) {
  const cleanText = stripBlockTags(ctx.state.deltaBuffer, {
    thinking: false,
    final: false,
  }).trim();
  
  if (cleanText && cleanText !== ctx.state.lastStreamedAssistant) {
    const { text, mediaUrls } = parseReplyDirectives(cleanText);
    
    void ctx.params.onPartialReply({
      text,
      mediaUrls: mediaUrls?.length ? mediaUrls : undefined,
    });
  }
}
```

**与Assistant Stream的区别**：

| 特性 | Assistant Stream | Partial Reply Stream |
|------|-----------------|---------------------|
| **包含增量** | ✅ 包含delta字段 | ❌ 仅完整text |
| **事件类型** | `onAgentEvent` | `onPartialReply` |
| **用途** | 调试、监控 | 前端渲染 |

---

## 六、消息边界处理

### 6.1 消息边界事件

```typescript
// message_start: 新消息开始
export function handleMessageStart(ctx, evt) {
  // 重置所有状态
  ctx.resetAssistantMessageState(ctx.state.assistantTexts.length);
  
  // 触发typing indicator
  void ctx.params.onAssistantMessageStart?.();
}

// message_end: 消息完成
export function handleMessageEnd(ctx, evt) {
  const msg = evt.message;
  const rawText = extractAssistantText(msg);
  const rawThinking = extractAssistantThinking(msg);
  
  // 1. 剥离标签
  const cleanText = ctx.stripBlockTags(rawText, {
    thinking: false,
    final: false,
  });
  
  // 2. 格式化推理内容
  const formattedReasoning = rawThinking
    ? formatReasoningMessage(rawThinking)
    : "";
  
  // 3. 最终化助手文本
  const addedDuringMessage = ctx.state.assistantTexts.length > ctx.state.assistantTextBaseline;
  const chunkerHasBuffered = ctx.blockChunker?.hasBuffered() ?? false;
  
  ctx.finalizeAssistantTexts({
    text: cleanText,
    addedDuringMessage,
    chunkerHasBuffered,
  });
  
  // 4. 发送推理内容（如果需要）
  if (ctx.state.includeReasoning && formattedReasoning) {
    void ctx.params.onBlockReply?.({
      text: formattedReasoning,
    });
  }
  
  // 5. 排空分块缓冲区
  if (ctx.blockChunker?.hasBuffered()) {
    ctx.blockChunker.drain({
      force: true,
      emit: ctx.emitBlockChunk,
    });
  } else if (ctx.state.blockBuffer.length > 0) {
    ctx.emitBlockChunk(ctx.state.blockBuffer);
  }
  
  // 6. 发送最终文本块
  if (cleanText && ctx.state.blockReplyBreak === "message_end") {
    void ctx.params.onBlockReply?.({
      text: cleanText,
    });
  }
}
```

---

## 七、完整流式输出流程示例

### 7.1 场景：用户询问"分析这个文件"

```
┌─────────────────────────────────────────────────────────┐
│                   流式输出完整流程                       │
└─────────────────────────────────────────────────────────┘

1. message_start
   └─ resetAssistantMessageState()
   └─ onAssistantMessageStart() → 显示typing indicator

2. text_delta: "<think>"
   └─ deltaBuffer += "<think>"
   └─ blockState.thinking = true
   └─ (不发送，在<think>标签内)

3. text_delta: "我需要先读取文件"
   └─ deltaBuffer += "我需要先读取文件"
   └─ streamReasoning → onReasoningStream("💭 我需要先读取文件")
   └─ (前端显示推理过程)

4. text_delta: "</think>"
   └─ deltaBuffer += "</think>"
   └─ blockState.thinking = false

5. text_delta: "让我读取"
   └─ deltaBuffer += "让我读取"
   └─ stripBlockTags() → "让我读取"
   └─ Assistant Stream → { text: "让我读取", delta: "让我读取" }
   └─ onPartialReply({ text: "让我读取" })
   └─ blockChunker.append("让我读取")
   └─ (前端显示: "让我读取")

6. text_delta: "文件内容"
   └─ deltaBuffer += "文件内容"
   └─ stripBlockTags() → "让我读取文件内容"
   └─ Assistant Stream → { text: "让我读取文件内容", delta: "文件内容" }
   └─ onPartialReply({ text: "让我读取文件内容" })
   └─ blockChunker.append("文件内容")
   └─ (前端显示: "让我读取文件内容")

7. text_end
   └─ blockChunker.drain(force: true)
   └─ onBlockReply({ text: "让我读取文件内容" })
   └─ (Discord/Telegram发送消息)

8. tool_call_begin: { name: "read", args: { path: "/file.txt" } }
   └─ onBlockReplyFlush() → 刷新缓冲
   └─ 工具开始执行

9. tool_result: { result: "文件内容..." }
   └─ formatToolResult() → "🔧 read: 读取成功，1024字节"
   └─ onToolResult({ text: "🔧 read: 读取成功，1024字节" })
   └─ (前端显示工具结果)

10. text_delta: "文件包含以下内容："
    └─ Assistant Stream → { delta: "文件包含以下内容：" }
    └─ onPartialReply({ text: "文件包含以下内容：" })
    └─ blockChunker.append("文件包含以下内容：")

11. text_delta: "\n\n- 第一项\n- 第二项\n- 第三项"
    └─ blockChunker.append("\n\n- 第一项\n- 第二项\n- 第三项")
    └─ blockChunker.drain(force: false)
    └─ (达到minChars，检测到段落边界\n\n)
    └─ onBlockReply({ text: "文件包含以下内容：" })

12. message_end
    └─ extractAssistantText() → "让我读取文件内容\n\n文件包含以下内容：..."
    └─ extractThinking() → "我需要先读取文件"
    └─ formatReasoningMessage() → "💭 **推理过程**\n\n我需要先读取文件"
    └─ onBlockReply({ text: "💭 **推理过程**\n\n..." }) (如果includeReasoning)
    └─ blockChunker.drain(force: true)
    └─ onBlockReply({ text: "- 第一项\n- 第二项\n- 第三项" })
    └─ 流式输出完成
```

---

## 八、前端集成示例

### 8.1 SSE（Server-Sent Events）接收

```typescript
// 前端代码

const eventSource = new EventSource("/api/agent/stream");

let assistantText = "";
let reasoningText = "";

// 通道1：推理流
eventSource.addEventListener("reasoning", (event) => {
  const data = JSON.parse(event.data);
  reasoningText = data.text;
  
  // 更新推理区域
  document.getElementById("reasoning").innerHTML = 
    renderMarkdown(reasoningText);
});

// 通道2：助手流
eventSource.addEventListener("assistant", (event) => {
  const data = JSON.parse(event.data);
  
  // 方式1：累加增量
  assistantText += data.delta;
  
  // 方式2：直接使用完整文本
  // assistantText = data.text;
  
  // 更新助手回复区域
  document.getElementById("assistant").innerHTML = 
    renderMarkdown(assistantText);
});

// 通道3：工具结果
eventSource.addEventListener("tool_result", (event) => {
  const data = JSON.parse(event.data);
  
  // 添加工具结果卡片
  const toolCard = createToolResultCard(data.text);
  document.getElementById("tools").appendChild(toolCard);
});

// 通道4：块回复（用于Discord/Telegram）
eventSource.addEventListener("block_reply", (event) => {
  const data = JSON.parse(event.data);
  
  // 发送到Discord/Telegram
  await sendMessage(data.text);
});

// 完成
eventSource.addEventListener("done", () => {
  eventSource.close();
  document.getElementById("status").textContent = "完成";
});
```

---

### 8.2 WebSocket接收

```typescript
const ws = new WebSocket("ws://localhost:3000/agent/stream");

ws.onmessage = (event) => {
  const message = JSON.parse(event.data);
  
  switch (message.stream) {
    case "reasoning":
      handleReasoningStream(message.data);
      break;
    
    case "assistant":
      handleAssistantStream(message.data);
      break;
    
    case "tool_result":
      handleToolResult(message.data);
      break;
    
    case "block_reply":
      handleBlockReply(message.data);
      break;
  }
};

function handleAssistantStream(data: { text: string; delta: string }) {
  // 增量渲染
  const container = document.getElementById("assistant");
  const currentText = container.textContent || "";
  container.textContent = currentText + data.delta;
  
  // 或者使用完整文本
  // container.innerHTML = renderMarkdown(data.text);
}
```

---

## 九、对您的Agent的建议

### 9.1 当前问题分析

根据您的Agent文档（`myagent.MD`），您使用了**LangGraph + SSE**：

```python
# 您的当前实现
async for chunk in app.astream(
    state,
    config=config,
    stream_mode="updates",
):
    for node_name, node_output in chunk.items():
        # 发送节点输出
        writer({"type": "node_update", "node": node_name, "data": node_output})
```

**当前问题**：
1. ❌ **粒度太粗**：按节点输出，不是逐字流式
2. ❌ **没有思考链分离**：推理和回复混在一起
3. ❌ **没有智能分块**：长文本一次性发送
4. ❌ **没有去重机制**：可能重复发送
5. ❌ **单一通道**：所有内容都通过一个通道

---

### 9.2 改进方案：多通道流式输出

```python
from typing import AsyncIterator, Literal
from dataclasses import dataclass
import re

@dataclass
class StreamChunk:
    """流式输出块"""
    stream: Literal["reasoning", "assistant", "tool_result", "block_reply"]
    data: dict


class MultiChannelStreamer:
    """多通道流式输出管理器"""
    
    def __init__(
        self,
        min_chunk_chars: int = 200,
        max_chunk_chars: int = 500,
        break_preference: Literal["paragraph", "newline", "sentence"] = "paragraph",
    ):
        self.min_chunk_chars = min_chunk_chars
        self.max_chunk_chars = max_chunk_chars
        self.break_preference = break_preference
        
        # 缓冲区
        self.delta_buffer = ""
        self.block_buffer = ""
        
        # 状态跟踪
        self.last_streamed_text = ""
        self.in_thinking_block = False
        self.reasoning_buffer = ""
        
        # 去重
        self.sent_texts = []
    
    async def stream_node_output(
        self,
        node_name: str,
        output: dict,
    ) -> AsyncIterator[StreamChunk]:
        """流式输出节点结果"""
        
        # 根据节点类型路由
        if node_name == "reason_node":
            async for chunk in self._stream_reason_output(output):
                yield chunk
        
        elif node_name == "tool_node":
            async for chunk in self._stream_tool_output(output):
                yield chunk
        
        elif node_name == "synthesize_node":
            async for chunk in self._stream_synthesize_output(output):
                yield chunk
    
    async def _stream_reason_output(self, output: dict) -> AsyncIterator[StreamChunk]:
        """流式输出推理节点结果"""
        
        # 提取AI消息
        messages = output.get("messages", [])
        if not messages:
            return
        
        ai_message = messages[-1]
        if not hasattr(ai_message, "content"):
            return
        
        content = ai_message.content
        
        # 分离推理和回复
        reasoning, assistant_text = self._extract_thinking(content)
        
        # 1. 流式输出推理
        if reasoning:
            yield StreamChunk(
                stream="reasoning",
                data={
                    "text": f"💭 **推理过程**\n\n{reasoning}",
                },
            )
        
        # 2. 流式输出助手回复（逐字）
        async for chunk in self._stream_text_gradually(assistant_text):
            yield chunk
        
        # 3. 分块发送（如果需要）
        if len(assistant_text) > self.min_chunk_chars:
            chunks = self._chunk_text(assistant_text)
            for chunk_text in chunks:
                yield StreamChunk(
                    stream="block_reply",
                    data={"text": chunk_text},
                )
    
    async def _stream_tool_output(self, output: dict) -> AsyncIterator[StreamChunk]:
        """流式输出工具节点结果"""
        
        messages = output.get("messages", [])
        for message in messages:
            if not hasattr(message, "name"):
                continue
            
            tool_name = message.name
            tool_result = message.content
            
            # 格式化工具结果
            formatted = f"🔧 **{tool_name}**\n\n{tool_result[:500]}..."
            
            yield StreamChunk(
                stream="tool_result",
                data={
                    "tool_name": tool_name,
                    "text": formatted,
                },
            )
    
    def _extract_thinking(self, text: str) -> tuple[str, str]:
        """提取思考内容"""
        
        # 匹配<think>...</think>标签
        think_pattern = r'<think>(.*?)</think>'
        matches = re.findall(think_pattern, text, re.DOTALL)
        
        if not matches:
            return "", text
        
        reasoning = "\n\n".join(m.strip() for m in matches)
        assistant_text = re.sub(think_pattern, "", text, flags=re.DOTALL).strip()
        
        return reasoning, assistant_text
    
    async def _stream_text_gradually(
        self,
        text: str,
        chunk_size: int = 5,
    ) -> AsyncIterator[StreamChunk]:
        """逐字流式输出文本"""
        
        import asyncio
        
        for i in range(0, len(text), chunk_size):
            chunk = text[i:i+chunk_size]
            self.delta_buffer += chunk
            
            # 去除重复
            if self._is_duplicate(self.delta_buffer):
                continue
            
            yield StreamChunk(
                stream="assistant",
                data={
                    "text": self.delta_buffer,
                    "delta": chunk,
                },
            )
            
            # 模拟流式延迟
            await asyncio.sleep(0.05)
        
        self.last_streamed_text = self.delta_buffer
    
    def _chunk_text(self, text: str) -> list[str]:
        """智能分块文本"""
        
        chunks = []
        buffer = text
        
        while len(buffer) > self.min_chunk_chars:
            # 查找分块点
            break_idx = self._find_break_index(buffer)
            
            if break_idx <= 0:
                break
            
            chunk = buffer[:break_idx].strip()
            if chunk:
                chunks.append(chunk)
            
            buffer = buffer[break_idx:].lstrip()
        
        if buffer.strip():
            chunks.append(buffer.strip())
        
        return chunks
    
    def _find_break_index(self, text: str) -> int:
        """查找最佳分块点"""
        
        # 策略1：段落边界（\n\n）
        if self.break_preference == "paragraph":
            para_idx = text.find("\n\n")
            if self.min_chunk_chars <= para_idx <= self.max_chunk_chars:
                return para_idx
        
        # 策略2：换行边界（\n）
        if self.break_preference in ["paragraph", "newline"]:
            lines = text.split("\n")
            cumulative = 0
            for i, line in enumerate(lines):
                cumulative += len(line) + 1
                if self.min_chunk_chars <= cumulative <= self.max_chunk_chars:
                    return cumulative
        
        # 策略3：句子边界（.!?）
        sentence_re = r'[.!?](?=\s|$)'
        for match in re.finditer(sentence_re, text):
            idx = match.end()
            if self.min_chunk_chars <= idx <= self.max_chunk_chars:
                return idx
        
        # 策略4：强制分块
        if len(text) > self.max_chunk_chars:
            return self.max_chunk_chars
        
        return -1
    
    def _is_duplicate(self, text: str) -> bool:
        """检查是否重复"""
        
        normalized = self._normalize_text(text)
        
        if normalized in self.sent_texts:
            return True
        
        self.sent_texts.append(normalized)
        
        # 限制缓存大小
        if len(self.sent_texts) > 100:
            self.sent_texts.pop(0)
        
        return False
    
    @staticmethod
    def _normalize_text(text: str) -> str:
        """规范化文本（用于去重）"""
        return text.strip().lower().replace(" ", "")


# ===== 使用示例 =====

async def stream_agent_response(
    user_message: str,
    session_id: str,
):
    """流式输出Agent响应"""
    
    streamer = MultiChannelStreamer(
        min_chunk_chars=200,
        max_chunk_chars=500,
        break_preference="paragraph",
    )
    
    # 执行LangGraph
    async for chunk in app.astream(
        {"messages": [{"role": "user", "content": user_message}]},
        config={"configurable": {"session_id": session_id}},
        stream_mode="updates",
    ):
        for node_name, node_output in chunk.items():
            # 多通道流式输出
            async for stream_chunk in streamer.stream_node_output(node_name, node_output):
                # 发送到前端（SSE）
                yield f"event: {stream_chunk.stream}\n"
                yield f"data: {json.dumps(stream_chunk.data)}\n\n"
    
    # 完成信号
    yield "event: done\n"
    yield "data: {}\n\n"


# ===== FastAPI集成 =====

from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

@app.post("/api/agent/stream")
async def agent_stream_endpoint(request: dict):
    user_message = request["message"]
    session_id = request["session_id"]
    
    return StreamingResponse(
        stream_agent_response(user_message, session_id),
        media_type="text/event-stream",
    )
```

---

## 十、总结

### Moltbot流式输出的核心优势

1. **多层架构**
   - 事件源 → 订阅处理 → 内容处理 → 分块策略 → 多通道输出
   - 每层职责清晰，易于维护和扩展

2. **5种独立通道**
   - Reasoning Stream：推理过程
   - Assistant Stream：助手回复（含增量）
   - Tool Result Stream：工具结果
   - Block Reply Stream：分块回复（Discord/Telegram）
   - Partial Reply Stream：部分回复（简化版）

3. **智能分块**
   - 3种分块策略（段落/换行/句子）
   - Fence保护（代码块不截断）
   - 动态阈值（min/max字符数）

4. **精细状态管理**
   - 40+状态字段
   - 去重机制（规范化+缓存）
   - 消息边界处理

5. **完整的Markdown支持**
   - 思考链提取（`<think>...</think>`）
   - 代码块保护（Fence Spans）
   - Reply Directives（`@media`, `@reply-to`等）

---

**这份文档全面解析了Moltbot的流式输出机制，包括完整的Python实现示例，可直接应用到您的Agent改进中！**
