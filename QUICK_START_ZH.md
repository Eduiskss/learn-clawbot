# Moltbot 快速开始指南

> 30分钟快速上手Moltbot - 从零到运行

## 🎯 学习目标

完成本指南后,你将:
- ✅ 成功运行Moltbot Gateway
- ✅ 连接一个消息平台(Telegram)
- ✅ 发送第一条AI消息
- ✅ 理解基本架构和工作流程

---

## 📋 前置要求

```bash
# 必需
✓ Node.js >= 22.12.0
✓ pnpm >= 10.23.0 (或 npm)
✓ Git

# 可选
○ Docker (用于沙箱功能)
○ Telegram账号 (用于测试)
```

### 检查环境

```bash
# 检查Node版本
node --version  # 应该 >= v22.12.0

# 检查pnpm
pnpm --version  # 应该 >= 10.23.0

# 如果没有pnpm,安装它
npm install -g pnpm
```

---

## 🚀 快速安装 (5分钟)

### 方式1: NPM全局安装 (推荐)

```bash
# 安装最新版本
npm install -g moltbot@latest

# 验证安装
moltbot --version

# 运行向导
moltbot onboard
```

### 方式2: 从源码安装 (开发用)

```bash
# 1. 克隆代码
git clone https://github.com/moltbot/moltbot.git
cd moltbot

# 2. 安装依赖
pnpm install

# 3. 构建UI
pnpm ui:build

# 4. 构建项目
pnpm build

# 5. 运行向导
pnpm moltbot onboard
```

---

## ⚙️ 初始化配置 (10分钟)

### 步骤1: 运行Onboarding向导

```bash
moltbot onboard --install-daemon
```

向导会引导你完成:
1. ✓ 选择AI模型提供商
2. ✓ 配置工作空间路径
3. ✓ 选择消息平台
4. ✓ 安装推荐的技能和钩子
5. ✓ 安装系统服务(可选)

### 步骤2: 配置AI模型

向导会提示你选择模型,推荐使用:

```bash
# Anthropic Claude (推荐)
# 需要Claude Pro/Max订阅
# 向导会引导OAuth认证流程
```

或者手动配置:

```json
// ~/.clawdbot/moltbot.json
{
  "agent": {
    "model": "anthropic/claude-opus-4-5"
  }
}
```

### 步骤3: 配置消息平台

**选项A: Telegram (最简单)**

1. 创建Bot: 与 [@BotFather](https://t.me/BotFather) 对话
2. 获取Token: `/newbot` 命令后获得
3. 配置:

```json
// ~/.clawdbot/moltbot.json
{
  "channels": {
    "telegram": {
      "botToken": "你的BOT_TOKEN",
      "allowFrom": ["你的Telegram用户ID"]
    }
  }
}
```

或使用环境变量:

```bash
export TELEGRAM_BOT_TOKEN="你的BOT_TOKEN"
```

**选项B: WhatsApp**

```bash
# 启动登录流程(会显示二维码)
moltbot channels login

# 配置白名单
moltbot config set channels.whatsapp.allowFrom '["你的手机号"]'
```

---

## 🎮 启动Gateway (5分钟)

### 启动服务

```bash
# 方式1: 前台运行(开发/调试)
moltbot gateway --port 18789 --verbose

# 方式2: 后台服务(生产)
moltbot daemon start

# 方式3: 开发模式(自动重载)
pnpm gateway:watch  # 仅限源码安装
```

### 验证运行状态

```bash
# 检查Gateway状态
moltbot gateway status

# 输出示例:
# ✓ Gateway is running
# ✓ WebSocket: ws://127.0.0.1:18789
# ✓ Dashboard: http://127.0.0.1:18789
# ✓ Channels: telegram (connected)
```

### 打开Dashboard

浏览器访问: http://127.0.0.1:18789

你会看到:
- 📊 系统状态
- 💬 聊天界面
- ⚙️ 配置管理
- 📝 会话列表

---

## 💬 发送第一条消息 (5分钟)

### 方式1: 通过Telegram

1. 打开Telegram
2. 找到你创建的Bot
3. 发送消息: "Hello!"
4. 等待AI回复

### 方式2: 通过CLI

```bash
# 发送消息到Telegram
moltbot agent --message "写一个Python冒泡排序" --thinking high

# 输出示例:
# [Agent] Thinking...
# [Agent] 好的,我来帮你写一个Python冒泡排序...
```

### 方式3: 通过Dashboard

1. 打开 http://127.0.0.1:18789
2. 点击"Chat"标签
3. 输入消息并发送

---

## 🔍 常见问题排查

### 问题1: Gateway启动失败

```bash
# 检查端口占用
netstat -ano | findstr 18789  # Windows
lsof -i :18789                # Linux/macOS

# 使用其他端口
moltbot gateway --port 19000
```

### 问题2: Telegram Bot无响应

```bash
# 检查Bot Token
moltbot config get channels.telegram.botToken

# 检查白名单
moltbot config get channels.telegram.allowFrom

# 查看日志
moltbot daemon logs
```

### 问题3: 模型认证失败

```bash
# 重新认证
moltbot models auth setup

# 检查认证状态
moltbot models status
```

### 问题4: 找不到命令

```bash
# 检查PATH
echo $PATH  # Linux/macOS
echo %PATH% # Windows

# 重新安装
npm install -g moltbot@latest --force
```

---

## 📚 下一步学习

### 基础使用

```bash
# 查看所有命令
moltbot --help

# 会话管理
moltbot sessions list
moltbot sessions reset main

# 配置管理
moltbot config get
moltbot config set key value

# 技能管理
moltbot skills list
moltbot skills install skill-name
```

### 进阶配置

1. **配置多个消息平台**
   - 阅读: `docs/channels/`
   - 支持: WhatsApp, Telegram, Discord, Slack, iMessage...

2. **自定义Agent行为**
   - 编辑: `~/clawd/AGENTS.md`
   - 定义个性: `~/clawd/SOUL.md`

3. **添加工具和技能**
   - 浏览: https://docs.molt.bot/tools/skills
   - 创建自定义Skill

4. **配置沙箱环境**
   - 安装Docker
   - 配置: `agent.sandbox.mode`

5. **远程访问**
   - Tailscale集成
   - SSH隧道

---

## 🎓 学习路径建议

### 第1天: 基础使用
- ✓ 完成本快速开始指南
- ✓ 发送10条不同类型的消息
- ✓ 探索Dashboard各个功能
- ✓ 阅读 `PROJECT_INIT_ZH.md` 的前3章

### 第2-3天: 深入理解
- ✓ 阅读完整的 `PROJECT_INIT_ZH.md`
- ✓ 理解Gateway、Agent、Channel架构
- ✓ 查看 `src/index.ts` 和 `src/gateway/server.impl.ts`
- ✓ 尝试修改 `AGENTS.md` 自定义行为

### 第4-7天: 代码探索
- ✓ 阅读 `ARCHITECTURE_DEEP_DIVE_ZH.md`
- ✓ 浏览 `src/agents/` 目录
- ✓ 理解工具系统 (`src/agents/pi-tools.ts`)
- ✓ 研究一个Channel实现 (`src/channels/telegram/`)

### 第2周: 扩展开发
- ✓ 创建自定义Tool
- ✓ 开发简单的Skill
- ✓ 编写Hook函数
- ✓ 尝试添加新的Channel适配器

### 第3-4周: 高级应用
- ✓ 配置沙箱环境
- ✓ 实现多Agent路由
- ✓ 优化性能和安全性
- ✓ 开始Python复现规划

---

## 📖 推荐阅读顺序

1. **本文档** (QUICK_START_ZH.md) ← 你在这里
2. **项目初始化文档** (PROJECT_INIT_ZH.md)
   - 第1-3章: 概览和架构
   - 第4-5章: 核心模块和Agent系统
3. **架构深度剖析** (ARCHITECTURE_DEEP_DIVE_ZH.md)
4. **官方文档**: https://docs.molt.bot

---

## 🆘 获取帮助

### 社区资源

- 📖 官方文档: https://docs.molt.bot
- 💬 Discord: https://discord.gg/clawd
- 🐛 GitHub Issues: https://github.com/moltbot/moltbot/issues
- 📧 项目主页: https://molt.bot

### 常用文档链接

- [完整配置参考](https://docs.molt.bot/gateway/configuration)
- [Channel配置](https://docs.molt.bot/channels)
- [Agent配置](https://docs.molt.bot/concepts/agent)
- [工具系统](https://docs.molt.bot/tools)
- [故障排查](https://docs.molt.bot/gateway/troubleshooting)

---

## ✅ 检查清单

完成下列任务以确保环境正常:

```bash
# 环境检查
□ Node.js版本 >= 22.12.0
□ pnpm已安装
□ Moltbot已全局安装

# 配置检查
□ 完成onboarding向导
□ 配置至少一个AI模型
□ 配置至少一个消息平台
□ 配置文件 ~/.clawdbot/moltbot.json 存在

# 运行检查
□ Gateway成功启动
□ Dashboard可以访问
□ 消息平台已连接
□ 成功收到AI回复

# 理解检查
□ 理解Gateway的作用
□ 知道Session的概念
□ 了解Channel适配器
□ 明白Agent和Tool的关系
```

---

## 🎉 恭喜!

你已经成功:
- ✅ 安装并运行了Moltbot
- ✅ 配置了消息平台
- ✅ 发送了第一条AI消息
- ✅ 理解了基本架构

**接下来可以**:
1. 深入学习项目架构 → 阅读 `PROJECT_INIT_ZH.md`
2. 探索高级功能 → 阅读 `ARCHITECTURE_DEEP_DIVE_ZH.md`
3. 开始开发扩展 → 查看 `docs/tools/creating-skills.md`
4. 规划Python复现 → 参考 `PROJECT_INIT_ZH.md` 第10章

祝学习愉快! 🚀
