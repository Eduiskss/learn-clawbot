# Moltbot 技术架构深度剖析

> 本文档深入剖析Moltbot的技术实现细节,适合希望深度理解或复现该项目的开发者

## 目录

1. [数据流分析](#1-数据流分析)
2. [消息处理流程](#2-消息处理流程)
3. [Gateway协议详解](#3-gateway协议详解)
4. [Agent执行机制](#4-agent执行机制)
5. [会话持久化](#5-会话持久化)
6. [工具调用链路](#6-工具调用链路)
7. [安全机制](#7-安全机制)
8. [性能优化策略](#8-性能优化策略)

---

## 1. 数据流分析

### 1.1 完整数据流向图

```
┌──────────────────────────────────────────────────────────────┐
│                      外部消息平台                              │
│         WhatsApp / Telegram / Discord / Slack                │
└──────────────────┬───────────────────────────────────────────┘
                   │ HTTP/WebSocket
                   ▼
┌──────────────────────────────────────────────────────────────┐
│                    Channel Adapter                            │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐             │
│  │  WhatsApp  │  │  Telegram  │  │   Discord  │  ...        │
│  │   监听器   │  │   Bot      │  │    Bot     │             │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘             │
│        │               │               │                      │
│        └───────────────┴───────────────┘                      │
└────────────────────────┬─────────────────────────────────────┘
                         │ IncomingMessage
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                  Gateway Router                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ 1. 消息验证 (来源、权限、配对)                          │   │
│  │ 2. 会话解析 (生成SessionKey)                          │   │
│  │ 3. 路由决策 (选择Agent)                               │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬─────────────────────────────────────┘
                         │ RoutedMessage
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                 Session Manager                               │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ • 加载会话历史                                          │   │
│  │ • 合并新消息                                            │   │
│  │ • 应用上下文窗口限制                                     │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬─────────────────────────────────────┘
                         │ SessionContext
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                   Pi Agent Runner                             │
│  ┌────────────────────┐                                      │
│  │ 1. 构建系统提示词   │                                      │
│  │ 2. 加载可用工具     │                                      │
│  │ 3. 调用LLM         │ ◄──┐                                │
│  │ 4. 解析工具调用     │    │                                │
│  └─────────┬──────────┘    │                                │
│            │              工具循环                            │
│            ▼               │                                 │
│  ┌────────────────────┐    │                                │
│  │   Tool Executor    │────┘                                │
│  │  • bash            │                                      │
│  │  • read/write      │                                      │
│  │  • browser         │                                      │
│  │  • sessions        │                                      │
│  └────────────────────┘                                      │
└────────────────────────┬─────────────────────────────────────┘
                         │ AgentResponse
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                Response Formatter                             │
│  • Markdown → 平台格式转换                                     │
│  • 消息分块(长消息)                                            │
│  • 附件处理                                                   │
└────────────────────────┬─────────────────────────────────────┘
                         │ FormattedResponse
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                 Channel Sender                                │
│  发送回复到原始消息平台                                         │
└──────────────────────────────────────────────────────────────┘
```

### 1.2 关键数据结构

```typescript
// 输入消息结构
interface IncomingMessage {
  channel: 'whatsapp' | 'telegram' | 'discord' | 'slack';
  from: string;              // 发送者ID
  chatId?: string;           // 群组ID(可选)
  text: string;              // 消息文本
  attachments?: Attachment[]; // 附件
  timestamp: number;
  metadata: Record<string, any>;
}

// 路由后的消息
interface RoutedMessage extends IncomingMessage {
  sessionKey: string;        // 会话Key
  agentId: string;           // 目标Agent
  allowedTools: string[];    // 可用工具
}

// 会话上下文
interface SessionContext {
  id: string;
  agentId: string;
  messages: ConversationMessage[];
  metadata: {
    model?: string;
    thinkingLevel?: 'off' | 'low' | 'medium' | 'high';
    verboseLevel?: number;
    toolPolicy?: ToolPolicy;
  };
}

// Agent响应
interface AgentResponse {
  content: string;           // 文本回复
  toolCalls: ToolCall[];     // 工具调用列表
  toolResults: ToolResult[]; // 工具结果
  usage?: {
    inputTokens: number;
    outputTokens: number;
    cost?: number;
  };
}
```

---

## 2. 消息处理流程

### 2.1 接收消息的完整流程

```typescript
// src/channels/telegram/bot.ts (示例)
async function handleIncomingMessage(update: Update) {
  // 步骤1: 提取消息信息
  const message = {
    channel: 'telegram',
    from: update.message.from.id.toString(),
    chatId: update.message.chat.type !== 'private' 
      ? update.message.chat.id.toString() 
      : undefined,
    text: update.message.text,
    timestamp: update.message.date * 1000,
  };
  
  // 步骤2: 安全检查
  const isAllowed = await checkAccess(message);
  if (!isAllowed) {
    await sendPairingCode(message);
    return;
  }
  
  // 步骤3: 路由到Gateway
  const response = await gateway.handleMessage(message);
  
  // 步骤4: 发送回复
  await sendResponse(update.message.chat.id, response);
}
```

### 2.2 Gateway处理流程

```typescript
// src/gateway/server-chat.ts (简化版)
export async function handleChatMessage(
  ws: WebSocket,
  message: IncomingMessage
): Promise<void> {
  // 1. 生成会话Key
  const sessionKey = deriveSessionKey(
    message.channel,
    message.from,
    message.chatId
  );
  
  // 2. 获取或创建会话
  let session = await loadSession(sessionKey);
  if (!session) {
    session = createNewSession(sessionKey, message);
  }
  
  // 3. 检查消息队列
  const queueStatus = await checkMessageQueue(sessionKey);
  if (queueStatus === 'busy') {
    await queueMessage(sessionKey, message);
    return;
  }
  
  // 4. 执行Agent
  const response = await runAgent(session, message);
  
  // 5. 保存会话
  await saveSession(sessionKey, response);
  
  // 6. 发送响应
  await sendResponseToChannel(message.channel, response);
}
```

### 2.3 消息队列机制

```typescript
// 队列状态管理
class MessageQueue {
  private queues = new Map<string, Message[]>();
  private processing = new Set<string>();
  
  async enqueue(sessionKey: string, message: Message) {
    if (!this.queues.has(sessionKey)) {
      this.queues.set(sessionKey, []);
    }
    this.queues.get(sessionKey)!.push(message);
    
    // 如果当前没有在处理,立即开始
    if (!this.processing.has(sessionKey)) {
      await this.processQueue(sessionKey);
    }
  }
  
  async processQueue(sessionKey: string) {
    this.processing.add(sessionKey);
    
    try {
      while (true) {
        const queue = this.queues.get(sessionKey);
        if (!queue || queue.length === 0) break;
        
        const message = queue.shift()!;
        await this.processMessage(sessionKey, message);
      }
    } finally {
      this.processing.delete(sessionKey);
    }
  }
}
```

---

## 3. Gateway协议详解

### 3.1 WebSocket消息格式

```typescript
// 客户端 → Gateway
interface GatewayRequest {
  id: string;                    // 请求ID
  method: string;                // 方法名
  params: Record<string, any>;   // 参数
  metadata?: {
    clientId?: string;
    token?: string;
  };
}

// Gateway → 客户端
interface GatewayResponse {
  id: string;                    // 对应请求ID
  result?: any;                  // 成功结果
  error?: {                      // 错误信息
    code: string;
    message: string;
    details?: any;
  };
}

// 推送事件
interface GatewayEvent {
  type: string;                  // 事件类型
  payload: any;                  // 事件数据
  timestamp: number;
}
```

### 3.2 核心RPC方法

```typescript
// 1. 发送聊天消息
{
  "method": "chat.send",
  "params": {
    "sessionKey": "telegram:main:123456",
    "message": "Hello, AI!",
    "options": {
      "thinking": "high",
      "model": "anthropic/claude-opus-4-5"
    }
  }
}

// 2. 列出会话
{
  "method": "sessions.list",
  "params": {
    "filter": "active",
    "limit": 50
  }
}

// 3. 应用配置
{
  "method": "config.apply",
  "params": {
    "config": {
      "agent": {
        "model": "anthropic/claude-opus-4-5"
      }
    }
  }
}

// 4. 调用工具
{
  "method": "tools.invoke",
  "params": {
    "tool": "bash",
    "args": {
      "command": "ls -la"
    }
  }
}
```

### 3.3 事件推送

```typescript
// Agent开始思考
{
  "type": "agent.thinking.start",
  "payload": {
    "sessionKey": "telegram:main:123456",
    "timestamp": 1234567890000
  }
}

// 工具调用
{
  "type": "tool.call",
  "payload": {
    "toolName": "bash",
    "args": { "command": "ls" }
  }
}

// 流式响应
{
  "type": "agent.response.chunk",
  "payload": {
    "sessionKey": "telegram:main:123456",
    "chunk": "这是一段回复文本...",
    "index": 0
  }
}

// Agent完成
{
  "type": "agent.response.complete",
  "payload": {
    "sessionKey": "telegram:main:123456",
    "fullResponse": "完整回复内容",
    "usage": {
      "inputTokens": 1234,
      "outputTokens": 567
    }
  }
}
```

---

## 4. Agent执行机制

### 4.1 Pi Agent集成架构

```typescript
// src/agents/pi-embedded.ts
import { PiAgent } from '@mariozechner/pi-agent-core';

class MoltbotAgentRunner {
  private agent: PiAgent;
  
  constructor(config: AgentConfig) {
    this.agent = new PiAgent({
      model: config.model,
      systemPrompt: this.buildSystemPrompt(config),
      tools: this.loadTools(config),
      maxIterations: 50,
    });
  }
  
  async run(session: SessionContext, message: string) {
    // 1. 构建上下文
    const context = {
      messages: session.messages,
      newMessage: message,
    };
    
    // 2. 执行Agent
    const stream = await this.agent.stream(context);
    
    // 3. 处理流式输出
    const chunks: string[] = [];
    for await (const chunk of stream) {
      if (chunk.type === 'text') {
        chunks.push(chunk.content);
        await this.emitChunk(chunk.content);
      } else if (chunk.type === 'tool_call') {
        await this.executeTool(chunk);
      }
    }
    
    return {
      content: chunks.join(''),
      toolCalls: stream.toolCalls,
      usage: stream.usage,
    };
  }
}
```

### 4.2 系统提示词生成

```typescript
// src/agents/system-prompt.ts
function buildSystemPrompt(context: PromptContext): string {
  const parts: string[] = [];
  
  // 1. 基础身份
  parts.push(`You are ${context.identity.name}, ${context.identity.role}`);
  
  // 2. 当前时间
  parts.push(`Current time: ${new Date().toISOString()}`);
  
  // 3. 工作空间路径
  parts.push(`Workspace: ${context.workspace}`);
  
  // 4. 可用工具列表
  parts.push(`Available tools:\n${formatTools(context.tools)}`);
  
  // 5. 注入AGENTS.md
  if (context.files.agents) {
    parts.push(`\n## Instructions\n${context.files.agents}`);
  }
  
  // 6. 注入SOUL.md
  if (context.files.soul) {
    parts.push(`\n## Personality\n${context.files.soul}`);
  }
  
  // 7. Skills注入
  if (context.skills.length > 0) {
    parts.push(`\n## Available Skills\n${formatSkills(context.skills)}`);
  }
  
  return parts.join('\n\n');
}
```

### 4.3 工具执行流程

```typescript
// 工具调用的生命周期
async function executeToolCall(toolCall: ToolCall): Promise<ToolResult> {
  // 1. 权限检查
  const isAllowed = await checkToolPolicy(toolCall.name);
  if (!isAllowed) {
    return {
      success: false,
      error: 'Tool not allowed in current context',
    };
  }
  
  // 2. 沙箱检查
  const needsSandbox = shouldRunInSandbox(toolCall);
  if (needsSandbox) {
    return await executInSandbox(toolCall);
  }
  
  // 3. 执行工具
  try {
    const tool = getToolImpl(toolCall.name);
    const result = await tool.execute(toolCall.args);
    
    return {
      success: true,
      data: result,
    };
  } catch (error) {
    return {
      success: false,
      error: error.message,
    };
  }
}
```

---

## 5. 会话持久化

### 5.1 会话存储结构

```bash
~/.clawdbot/
├── sessions/
│   ├── telegram:main:123456.json       # Telegram私聊会话
│   ├── telegram:group:789012.json      # Telegram群组会话
│   ├── whatsapp:main:+1234567890.json  # WhatsApp私聊
│   └── discord:guild:456789.json       # Discord服务器
└── agents/
    └── default/
        ├── sessions/
        │   └── session-abc123.jsonl    # Pi会话日志
        └── workspace/
            └── ...
```

### 5.2 会话文件格式

```json
// telegram:main:123456.json
{
  "id": "telegram:main:123456",
  "agentId": "default",
  "createdAt": 1234567890000,
  "updatedAt": 1234567890000,
  "messages": [
    {
      "role": "user",
      "content": "Hello",
      "timestamp": 1234567890000
    },
    {
      "role": "assistant",
      "content": "Hi! How can I help?",
      "timestamp": 1234567890001,
      "usage": {
        "inputTokens": 10,
        "outputTokens": 8
      }
    }
  ],
  "metadata": {
    "model": "anthropic/claude-opus-4-5",
    "thinkingLevel": "medium",
    "totalMessages": 2,
    "totalTokens": 18
  }
}
```

### 5.3 会话管理策略

```typescript
class SessionManager {
  // 会话修剪策略
  async pruneSession(session: Session): Promise<Session> {
    const maxMessages = 100;
    const maxTokens = 100000;
    
    // 1. 消息数量限制
    if (session.messages.length > maxMessages) {
      // 保留最近的消息
      session.messages = session.messages.slice(-maxMessages);
    }
    
    // 2. Token数量限制
    const totalTokens = this.countTokens(session.messages);
    if (totalTokens > maxTokens) {
      // 使用摘要压缩
      await this.compactSession(session);
    }
    
    return session;
  }
  
  // 会话压缩
  async compactSession(session: Session): Promise<void> {
    // 提取前半部分消息
    const oldMessages = session.messages.slice(0, session.messages.length / 2);
    
    // 生成摘要
    const summary = await this.generateSummary(oldMessages);
    
    // 替换为摘要
    session.messages = [
      {
        role: 'system',
        content: `Previous conversation summary:\n${summary}`,
      },
      ...session.messages.slice(session.messages.length / 2),
    ];
  }
}
```

---

## 6. 工具调用链路

### 6.1 工具注册机制

```typescript
// src/agents/pi-tools.ts
interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: ZodSchema;
  execute: (params: any, context: ToolContext) => Promise<ToolResult>;
  policy?: ToolPolicy;
}

// 工具注册表
class ToolRegistry {
  private tools = new Map<string, ToolDefinition>();
  
  register(tool: ToolDefinition) {
    this.tools.set(tool.name, tool);
  }
  
  get(name: string): ToolDefinition | undefined {
    return this.tools.get(name);
  }
  
  // 根据策略过滤工具
  filterByPolicy(policy: ToolPolicy): ToolDefinition[] {
    return Array.from(this.tools.values()).filter(tool => {
      if (policy.allow && !policy.allow.includes(tool.name)) {
        return false;
      }
      if (policy.deny && policy.deny.includes(tool.name)) {
        return false;
      }
      return true;
    });
  }
}
```

### 6.2 Bash工具实现深度

```typescript
// src/agents/bash-tools.exec.ts
export const bashTool: ToolDefinition = {
  name: 'bash',
  description: 'Execute bash commands',
  
  inputSchema: z.object({
    command: z.string().describe('The bash command to execute'),
    cwd: z.string().optional().describe('Working directory'),
    env: z.record(z.string()).optional().describe('Environment variables'),
    timeout: z.number().optional().describe('Timeout in milliseconds'),
  }),
  
  async execute(params, context) {
    const { command, cwd, env, timeout = 30000 } = params;
    
    // 1. 安全检查
    if (context.sandbox && !context.sandbox.allowBash) {
      throw new Error('Bash execution not allowed in this context');
    }
    
    // 2. 命令黑名单检查
    const blockedCommands = ['rm -rf /', 'mkfs', 'dd if=/dev/zero'];
    if (blockedCommands.some(cmd => command.includes(cmd))) {
      throw new Error('Dangerous command blocked');
    }
    
    // 3. 执行命令
    const subprocess = spawn('bash', ['-c', command], {
      cwd: cwd || context.workspace,
      env: { ...process.env, ...env },
      timeout,
    });
    
    // 4. 收集输出
    let stdout = '';
    let stderr = '';
    
    subprocess.stdout.on('data', data => stdout += data);
    subprocess.stderr.on('data', data => stderr += data);
    
    // 5. 等待完成
    const exitCode = await new Promise<number>((resolve) => {
      subprocess.on('exit', code => resolve(code || 0));
    });
    
    return {
      stdout,
      stderr,
      exitCode,
      command,
    };
  },
};
```

### 6.3 Browser工具架构

```typescript
// src/browser/ 模块
class BrowserTool {
  private browser?: Browser;
  private page?: Page;
  
  async initialize() {
    // 启动Chromium
    this.browser = await chromium.launch({
      headless: true,
      args: ['--no-sandbox'],
    });
    
    this.page = await this.browser.newPage();
  }
  
  // browser_navigate工具
  async navigate(url: string) {
    await this.page!.goto(url);
    return {
      url: this.page!.url(),
      title: await this.page!.title(),
    };
  }
  
  // browser_snapshot工具
  async snapshot() {
    // 获取可访问性树
    const snapshot = await this.page!.accessibility.snapshot();
    
    // 转换为文本表示
    return this.formatSnapshot(snapshot);
  }
  
  // browser_click工具
  async click(selector: string) {
    await this.page!.click(selector);
  }
  
  // browser_type工具
  async type(selector: string, text: string) {
    await this.page!.type(selector, text);
  }
}
```

---

## 7. 安全机制

### 7.1 DM配对机制

```typescript
// src/pairing/pairing.ts
class PairingManager {
  private pendingPairs = new Map<string, PairingRequest>();
  
  async requestPairing(channel: string, userId: string): Promise<string> {
    // 生成6位配对码
    const code = this.generateCode();
    
    // 保存配对请求
    this.pendingPairs.set(code, {
      channel,
      userId,
      timestamp: Date.now(),
      expiresAt: Date.now() + 5 * 60 * 1000, // 5分钟过期
    });
    
    return code;
  }
  
  async approvePairing(code: string): Promise<boolean> {
    const request = this.pendingPairs.get(code);
    if (!request) return false;
    
    // 检查过期
    if (Date.now() > request.expiresAt) {
      this.pendingPairs.delete(code);
      return false;
    }
    
    // 添加到白名单
    await this.addToAllowlist(request.channel, request.userId);
    
    // 删除配对请求
    this.pendingPairs.delete(code);
    
    return true;
  }
  
  private generateCode(): string {
    return Math.random().toString(36).substr(2, 6).toUpperCase();
  }
}
```

### 7.2 沙箱隔离

```typescript
// src/agents/sandbox/docker-sandbox.ts
class DockerSandbox {
  async createContainer(config: SandboxConfig) {
    // 创建Docker容器
    const container = await docker.createContainer({
      Image: config.image || 'moltbot-sandbox:latest',
      
      // 资源限制
      HostConfig: {
        Memory: 512 * 1024 * 1024,  // 512MB
        CpuShares: 1024,
        PidsLimit: 100,
        
        // 挂载工作空间(只读)
        Binds: [
          `${config.workspaceRoot}:/workspace:ro`,
        ],
      },
      
      // 网络隔离
      NetworkMode: 'none',
      
      // 用户隔离
      User: 'sandbox',
    });
    
    await container.start();
    return container;
  }
  
  async executeInContainer(
    container: Container,
    command: string
  ): Promise<ExecutionResult> {
    const exec = await container.exec({
      Cmd: ['bash', '-c', command],
      AttachStdout: true,
      AttachStderr: true,
    });
    
    const stream = await exec.start();
    
    // 收集输出
    const output = await this.collectOutput(stream);
    
    return output;
  }
}
```

### 7.3 工具权限策略

```typescript
// 工具策略定义
interface ToolPolicy {
  mode: 'allowlist' | 'denylist';
  allow?: string[];
  deny?: string[];
  
  // 特殊规则
  sandbox?: {
    requireSandbox: boolean;
    allowedTools: string[];
  };
}

// 策略应用
class ToolPolicyEnforcer {
  checkPermission(
    toolName: string,
    policy: ToolPolicy,
    context: ExecutionContext
  ): boolean {
    // 1. 沙箱检查
    if (context.isSandboxed && policy.sandbox) {
      if (policy.sandbox.requireSandbox) {
        return policy.sandbox.allowedTools.includes(toolName);
      }
    }
    
    // 2. 模式检查
    if (policy.mode === 'allowlist') {
      return policy.allow?.includes(toolName) ?? false;
    } else {
      return !(policy.deny?.includes(toolName) ?? false);
    }
  }
}
```

---

## 8. 性能优化策略

### 8.1 连接池管理

```typescript
// 数据库连接池
class ConnectionPool {
  private pool: Connection[] = [];
  private maxSize = 10;
  
  async getConnection(): Promise<Connection> {
    // 从池中获取
    if (this.pool.length > 0) {
      return this.pool.pop()!;
    }
    
    // 创建新连接
    if (this.pool.length < this.maxSize) {
      return await this.createConnection();
    }
    
    // 等待可用连接
    return await this.waitForConnection();
  }
  
  releaseConnection(conn: Connection) {
    this.pool.push(conn);
  }
}
```

### 8.2 消息批处理

```typescript
// 批量发送消息
class BatchSender {
  private batch: Message[] = [];
  private batchSize = 10;
  private batchTimeout = 1000; // 1秒
  private timer?: NodeJS.Timeout;
  
  enqueue(message: Message) {
    this.batch.push(message);
    
    // 达到批量大小,立即发送
    if (this.batch.length >= this.batchSize) {
      this.flush();
    } else {
      // 设置超时
      this.resetTimer();
    }
  }
  
  private resetTimer() {
    if (this.timer) clearTimeout(this.timer);
    
    this.timer = setTimeout(() => {
      this.flush();
    }, this.batchTimeout);
  }
  
  private async flush() {
    if (this.batch.length === 0) return;
    
    const messages = this.batch.splice(0);
    await this.sendBatch(messages);
  }
}
```

### 8.3 缓存策略

```typescript
// LRU缓存实现
class LRUCache<K, V> {
  private cache = new Map<K, V>();
  private maxSize: number;
  
  constructor(maxSize: number) {
    this.maxSize = maxSize;
  }
  
  get(key: K): V | undefined {
    const value = this.cache.get(key);
    
    if (value !== undefined) {
      // 移到最前面(LRU)
      this.cache.delete(key);
      this.cache.set(key, value);
    }
    
    return value;
  }
  
  set(key: K, value: V) {
    // 删除旧的
    if (this.cache.has(key)) {
      this.cache.delete(key);
    }
    
    // 检查容量
    if (this.cache.size >= this.maxSize) {
      // 删除最旧的(第一个)
      const firstKey = this.cache.keys().next().value;
      this.cache.delete(firstKey);
    }
    
    // 添加新的
    this.cache.set(key, value);
  }
}

// 应用缓存
const sessionCache = new LRUCache<string, Session>(100);

async function getSession(sessionKey: string): Promise<Session> {
  // 先查缓存
  let session = sessionCache.get(sessionKey);
  
  if (!session) {
    // 从磁盘加载
    session = await loadSessionFromDisk(sessionKey);
    sessionCache.set(sessionKey, session);
  }
  
  return session;
}
```

### 8.4 并发控制

```typescript
// 限流器
class RateLimiter {
  private tokens: number;
  private maxTokens: number;
  private refillRate: number; // tokens/second
  private lastRefill: number;
  
  constructor(maxTokens: number, refillRate: number) {
    this.maxTokens = maxTokens;
    this.tokens = maxTokens;
    this.refillRate = refillRate;
    this.lastRefill = Date.now();
  }
  
  async acquire(cost: number = 1): Promise<void> {
    // 补充tokens
    this.refill();
    
    // 检查是否有足够的tokens
    if (this.tokens >= cost) {
      this.tokens -= cost;
      return;
    }
    
    // 等待补充
    const waitTime = (cost - this.tokens) / this.refillRate * 1000;
    await this.sleep(waitTime);
    
    this.refill();
    this.tokens -= cost;
  }
  
  private refill() {
    const now = Date.now();
    const elapsed = (now - this.lastRefill) / 1000;
    const tokensToAdd = elapsed * this.refillRate;
    
    this.tokens = Math.min(this.maxTokens, this.tokens + tokensToAdd);
    this.lastRefill = now;
  }
  
  private sleep(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## 总结

本文档深入剖析了Moltbot的技术实现细节:

1. **数据流**: 从消息平台到Agent的完整链路
2. **消息处理**: 队列、路由、会话管理
3. **Gateway协议**: WebSocket RPC和事件推送
4. **Agent机制**: Pi集成、系统提示词、工具执行
5. **持久化**: 会话存储和管理策略
6. **工具系统**: 注册、执行、沙箱隔离
7. **安全机制**: 配对、沙箱、权限策略
8. **性能优化**: 连接池、批处理、缓存、限流

这些实现细节对于理解或复现Moltbot至关重要。
