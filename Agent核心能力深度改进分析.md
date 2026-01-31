# Agent核心能力深度改进分析

> **目标**: 达到或超越Moltbot的Agent智能水平  
> **聚焦**: Agent推理能力 + 交互体验  
> **忽略**: 多平台Gateway

---

## 执行摘要

### 🎯 核心发现

经过深度分析，您的Agent与Moltbot在**核心智能能力**上有以下关键差距：

| 能力维度 | 您的实现 | Moltbot | 差距严重度 |
|---------|---------|---------|-----------|
| **推理灵活性** | 计划→执行（刚性） | ReAct循环（动态） | 🔴 高 |
| **自主行为** | 完全被动 | Hooks+Heartbeat | 🔴 高 |
| **错误恢复** | 简单重试 | 重新规划 | 🟡 中 |
| **思维透明度** | 内部CoT | 可见thinking | 🟡 中 |
| **工具组合** | 预规划固定 | 动态组合 | 🟡 中 |
| **中断与恢复** | 不支持 | 完整支持 | 🟡 中 |

### 🚨 最关键的3个改进点

1. **从"计划-执行"升级到"ReAct循环"** — 这是智能水平差距的核心
2. **引入自主行为机制** — 让Agent能主动触发而非被动等待
3. **增强错误恢复与重新规划** — 提升鲁棒性和智能感

---

## 一、推理模式的根本差异（最重要）

### 1.1 您的当前模式：计划-执行（Plan-Execute）

```
用户问题
    ↓
understand_node（理解）
    ↓
plan_node（生成完整计划）
    ↓
┌─────────────────────────────────┐
│  ExecutionPlan（固定步骤列表）  │
│  step1 → step2 → step3 → ...   │
└─────────────────────────────────┘
    ↓
step_node（机械执行每一步）
    ↓
synthesize_node（汇总）
    ↓
输出
```

**问题所在**:

1. **计划在执行前就固定了** — 无法根据中间结果调整策略
2. **step_node是"执行者"而非"思考者"** — 只是机械执行预定步骤
3. **缺乏反馈循环** — 执行结果不影响后续决策

**典型失败场景**:

```python
# 用户问题："分析杭州仓今天的瓶颈环节"

# plan_node生成的计划：
steps = [
    {"id": "1", "action": "query", "tool": "get_warehouse_stages"},
    {"id": "2", "action": "query", "tool": "get_stage_current_status", "args": {"stage": "分拣"}},
    {"id": "3", "action": "synthesize"}
]

# 问题：如果step1发现今天根本没有"分拣"环节数据（节假日），
# plan_node无法动态调整去查看其他环节
# step_node会继续执行step2，得到空结果
# synthesize_node只能说"分拣数据不足"
```

### 1.2 Moltbot的模式：ReAct循环（Reasoning-Action）

```
用户问题
    ↓
┌──────────────────────────────────────────┐
│              ReAct 循环                   │
│                                          │
│  Thought: "我需要先查看有哪些环节..."     │
│      ↓                                   │
│  Action: get_warehouse_stages()          │
│      ↓                                   │
│  Observation: [分拣, 上架, 打包...]       │
│      ↓                                   │
│  Thought: "分拣数据为空，我应该看上架..."  │  ← 动态调整！
│      ↓                                   │
│  Action: get_stage_status("上架")        │
│      ↓                                   │
│  Observation: {...}                      │
│      ↓                                   │
│  Thought: "现在可以得出结论了..."         │
│      ↓                                   │
│  Final Answer: "..."                     │
│                                          │
└──────────────────────────────────────────┘
```

**核心优势**:

1. **每一步都重新思考** — 根据上一步结果决定下一步
2. **动态调整策略** — 遇到意外情况可以改变方向
3. **更接近人类推理** — "先看看，再决定"

### 1.3 🔧 改进方案：混合ReAct架构

**目标**: 保留您的双路径优势，同时引入ReAct的动态决策能力

```python
# 新架构：Adaptive ReAct

class AdaptiveReActAgent:
    """
    混合架构：
    - Standard路径：保持现有设计（简单问题）
    - Complex路径：升级为ReAct循环
    """
    
    async def complex_path_react(self, state, runtime):
        """
        ReAct循环实现
        """
        max_iterations = 10  # 安全限制
        iteration = 0
        
        # 初始思考
        context = state.get("enriched_context", {})
        understanding = state.get("understanding", {})
        
        while iteration < max_iterations:
            iteration += 1
            
            # === THOUGHT 阶段 ===
            thought = await self._generate_thought(
                messages=state["messages"],
                context=context,
                previous_observations=state.get("observations", []),
                goal=understanding.get("high_level_goal")
            )
            
            # 流式输出思考过程
            writer = get_stream_writer()
            writer({
                "type": "thinking",
                "content": thought.reasoning,
                "iteration": iteration
            })
            
            # === 决策点：是否需要行动 ===
            if thought.decision == "final_answer":
                # 可以直接回答了
                return await self._generate_final_answer(state, thought)
            
            if thought.decision == "need_action":
                # === ACTION 阶段 ===
                action = thought.next_action
                
                writer({
                    "type": "action",
                    "tool": action.tool_name,
                    "reason": action.reason
                })
                
                # 执行工具
                observation = await self._execute_tool(
                    action.tool_name,
                    action.tool_args,
                    runtime
                )
                
                # === OBSERVATION 阶段 ===
                # 将观察结果加入上下文
                state.setdefault("observations", []).append({
                    "iteration": iteration,
                    "thought": thought.reasoning,
                    "action": action.tool_name,
                    "observation": observation
                })
                
                writer({
                    "type": "observation",
                    "result_summary": self._summarize_observation(observation)
                })
                
                # 继续循环
                continue
            
            if thought.decision == "stuck":
                # 无法继续，需要请求用户澄清
                return await self._request_clarification(state, thought)
        
        # 超过最大迭代
        return await self._force_conclusion(state)
    
    async def _generate_thought(self, **kwargs) -> ThoughtResult:
        """
        生成思考
        
        Prompt结构：
        你是一个专业的仓库分析助手。
        
        当前目标：{goal}
        
        已有观察：
        {formatted_observations}
        
        请思考：
        1. 我现在知道了什么？
        2. 我还需要了解什么？
        3. 下一步应该做什么？
        
        输出格式：
        <reasoning>你的推理过程</reasoning>
        <decision>need_action | final_answer | stuck</decision>
        <next_action>如果need_action，指定工具和参数</next_action>
        """
        
        prompt = self._build_thought_prompt(**kwargs)
        
        response = await self.model.with_structured_output(ThoughtSchema).ainvoke(prompt)
        
        return response
```

### 1.4 ReAct vs Plan-Execute 对比

| 场景 | Plan-Execute（当前） | ReAct（改进后） |
|------|---------------------|-----------------|
| 数据缺失 | 继续执行，报告失败 | 调整策略，查其他数据 |
| 发现异常 | 按计划综合，可能遗漏 | 深入追问，分析异常 |
| 用户意图模糊 | 猜测后执行固定计划 | 边执行边澄清 |
| 复杂多步骤 | 一次规划可能不准 | 逐步推进，更准确 |

### 1.5 实施建议

**Phase 1** (1-2周): 在Complex路径引入ReAct

```python
# 修改工作流
workflow.add_node("react_loop", react_loop_node)  # 替代 plan + step + synthesize

# 路由
def route_complexity(state):
    if state["understanding"]["complexity"] == "complex":
        return "react_loop"  # 进入ReAct
    return "reason_node"  # Standard保持不变
```

**Phase 2** (2-4周): 优化思考-行动质量

- 调优Thought生成的Prompt
- 增加observation总结能力
- 优化工具选择准确率

**Phase 3** (1月): 高级特性

- 多轮对话中的策略延续
- 不确定时主动澄清
- 学习用户反馈优化决策

---

## 二、自主行为能力（Moltbot的杀手级特性）

### 2.1 当前状态：完全被动

```
您的Agent工作模式：

用户 → [发送消息] → Agent响应 → 等待...
用户 → [发送消息] → Agent响应 → 等待...
                    ↑
                永远在等待用户
```

### 2.2 Moltbot的自主行为系统

```
Moltbot工作模式：

┌──────────────────────────────────────────────┐
│                 Heartbeat                     │
│        每30分钟自动检查一次                    │
│   - 有未处理的任务吗？                        │
│   - 需要提醒用户什么吗？                      │
│   - 系统状态正常吗？                          │
│        ↓                                      │
│   有重要事项 → 主动发送消息                   │
│   无事项 → HEARTBEAT_OK（静默）               │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│                   Hooks                       │
│        事件驱动的自动触发                      │
│   - 收到新邮件 → 分析并摘要                   │
│   - 定时任务 → 生成日报                       │
│   - 系统启动 → 运行检查清单                   │
└──────────────────────────────────────────────┘

┌──────────────────────────────────────────────┐
│                 Cron Jobs                     │
│        定时任务系统                           │
│   - "每天9点生成昨日报告"                     │
│   - "每周五总结本周数据"                      │
│   - "每小时检查异常指标"                      │
└──────────────────────────────────────────────┘
```

### 2.3 🔧 改进方案：引入自主行为框架

```python
# app/agent/autonomous/heartbeat.py

import asyncio
from datetime import datetime, timedelta
from typing import Optional, List

class HeartbeatRunner:
    """
    心跳运行器 - 让Agent具备自主检查能力
    """
    
    def __init__(
        self,
        agent,
        check_interval_minutes: int = 30,
        quiet_hours: tuple = (22, 8)  # 22:00-08:00 静默
    ):
        self.agent = agent
        self.interval = check_interval_minutes * 60
        self.quiet_hours = quiet_hours
        self.running = False
        self.last_check = None
        
    async def start(self):
        """启动心跳"""
        self.running = True
        
        while self.running:
            # 检查是否在静默时段
            if self._is_quiet_hours():
                await asyncio.sleep(self.interval)
                continue
            
            # 执行心跳检查
            result = await self._run_heartbeat_check()
            
            if result.has_important_items:
                # 有重要事项，发送通知
                await self._send_notification(result)
            
            self.last_check = datetime.now()
            await asyncio.sleep(self.interval)
    
    async def _run_heartbeat_check(self) -> HeartbeatResult:
        """
        执行心跳检查
        
        检查清单（可配置）：
        1. 是否有异常指标需要关注
        2. 是否有待处理的任务
        3. 是否需要生成定期报告
        4. 用户是否有未回复的问题
        """
        
        checks = []
        
        # 检查1: 异常指标
        anomaly_check = await self._check_anomalies()
        checks.append(anomaly_check)
        
        # 检查2: 待处理任务
        pending_check = await self._check_pending_tasks()
        checks.append(pending_check)
        
        # 检查3: 定期报告
        report_check = await self._check_scheduled_reports()
        checks.append(report_check)
        
        # 汇总
        important_items = [c for c in checks if c.is_important]
        
        return HeartbeatResult(
            has_important_items=len(important_items) > 0,
            items=important_items,
            checked_at=datetime.now()
        )
    
    async def _check_anomalies(self) -> CheckResult:
        """检查是否有需要关注的异常"""
        
        # 使用Agent的工具检查
        result = await self.agent.execute_tool(
            "detect_stage_anomalies",
            {"warehouse_code": self._get_user_warehouses()}
        )
        
        if result.get("anomalies"):
            return CheckResult(
                is_important=True,
                type="anomaly",
                message=f"检测到{len(result['anomalies'])}个异常指标",
                details=result
            )
        
        return CheckResult(is_important=False, type="anomaly")


# app/agent/autonomous/hooks.py

class HookRegistry:
    """
    Hook注册表 - 事件驱动的自动触发
    """
    
    def __init__(self):
        self.hooks = defaultdict(list)
    
    def register(self, event_type: str, handler):
        """注册Hook"""
        self.hooks[event_type].append(handler)
    
    async def trigger(self, event_type: str, event_data: dict):
        """触发Hook"""
        handlers = self.hooks.get(event_type, [])
        
        for handler in handlers:
            try:
                await handler(event_data)
            except Exception as e:
                logger.error(f"Hook error [{event_type}]: {e}")

# 内置Hook示例
@hook_registry.register("session:start")
async def on_session_start(event):
    """会话开始时的Hook"""
    # 加载用户上下文
    # 检查是否有未完成的任务
    pass

@hook_registry.register("data:anomaly_detected")
async def on_anomaly_detected(event):
    """检测到异常时的Hook"""
    # 自动分析异常
    # 生成预警通知
    pass

@hook_registry.register("schedule:daily_report")
async def on_daily_report_due(event):
    """日报时间到达的Hook"""
    # 自动生成日报
    # 发送给用户
    pass


# app/agent/autonomous/cron.py

class CronScheduler:
    """
    定时任务调度器
    """
    
    def __init__(self, agent):
        self.agent = agent
        self.jobs = {}
        self.running = False
    
    def add_job(
        self,
        job_id: str,
        schedule: str,  # cron表达式: "0 9 * * *" = 每天9点
        handler,
        params: dict = None
    ):
        """添加定时任务"""
        self.jobs[job_id] = CronJob(
            id=job_id,
            schedule=schedule,
            handler=handler,
            params=params or {}
        )
    
    async def start(self):
        """启动调度器"""
        self.running = True
        
        while self.running:
            now = datetime.now()
            
            for job in self.jobs.values():
                if job.should_run(now):
                    asyncio.create_task(self._run_job(job))
                    job.last_run = now
            
            await asyncio.sleep(60)  # 每分钟检查一次
    
    async def _run_job(self, job: CronJob):
        """执行任务"""
        try:
            await job.handler(self.agent, job.params)
        except Exception as e:
            logger.error(f"Cron job error [{job.id}]: {e}")

# 使用示例
scheduler = CronScheduler(agent)

# 每天9点生成日报
scheduler.add_job(
    "daily_report",
    "0 9 * * *",
    generate_daily_report,
    {"warehouses": ["HZ001", "SH001"]}
)

# 每小时检查异常
scheduler.add_job(
    "anomaly_check",
    "0 * * * *",
    check_anomalies
)

# 每周五17点生成周报
scheduler.add_job(
    "weekly_report",
    "0 17 * * 5",
    generate_weekly_report
)
```

### 2.4 自主行为的价值

| 能力 | 用户体验提升 |
|------|-------------|
| Heartbeat | "Agent主动告诉我有问题，不用我去问" |
| 定时报告 | "每天早上自动收到昨日总结" |
| 异常预警 | "还没等我发现问题，Agent就提醒了" |
| 任务跟进 | "Agent记得我昨天说的事情，主动跟进" |

### 2.5 实施建议

**Phase 1** (1周): 基础Heartbeat
- 实现HeartbeatRunner
- 配置基础检查项
- 静默时段处理

**Phase 2** (1-2周): Cron定时任务
- 日报自动生成
- 异常定期检查
- 周报/月报

**Phase 3** (2-4周): Hook事件系统
- 会话生命周期Hook
- 数据变化Hook
- 用户自定义Hook

---

## 三、错误恢复与重新规划

### 3.1 当前状态：简单重试

```python
# 您的当前实现
STEP_NODE_RETRY_POLICY = RetryPolicy(
    max_attempts=2,
    initial_interval=1.0,
    backoff_factor=2.0,
    jitter=True
)

# 问题：只是重试相同的操作，不会调整策略
```

### 3.2 Moltbot的错误恢复

```
Moltbot的错误处理：

Tool Error
    ↓
检测错误类型
    ↓
┌───────────────────────────────────────┐
│  可重试错误（网络、超时）              │
│      ↓                                │
│  自动重试（带退避）                    │
└───────────────────────────────────────┘
┌───────────────────────────────────────┐
│  逻辑错误（参数错误、资源不存在）       │
│      ↓                                │
│  重新思考：                           │
│  "这个工具不行，换个方法..."           │
│      ↓                                │
│  生成新的Action                       │
└───────────────────────────────────────┘
┌───────────────────────────────────────┐
│  致命错误（权限、配置）                │
│      ↓                                │
│  向用户说明问题                        │
│  请求帮助或跳过                        │
└───────────────────────────────────────┘
```

### 3.3 🔧 改进方案：智能错误恢复

```python
# app/agent/recovery/error_handler.py

from enum import Enum
from typing import Optional, Callable

class ErrorCategory(Enum):
    TRANSIENT = "transient"      # 暂时性错误，可重试
    LOGICAL = "logical"          # 逻辑错误，需调整策略
    FATAL = "fatal"              # 致命错误，需人工介入
    UNKNOWN = "unknown"

class ErrorClassifier:
    """错误分类器"""
    
    TRANSIENT_PATTERNS = [
        "timeout", "connection", "rate limit",
        "temporarily unavailable", "retry"
    ]
    
    LOGICAL_PATTERNS = [
        "not found", "invalid", "does not exist",
        "no data", "empty result", "permission denied"
    ]
    
    FATAL_PATTERNS = [
        "authentication failed", "invalid api key",
        "quota exceeded", "account suspended"
    ]
    
    @classmethod
    def classify(cls, error: Exception) -> ErrorCategory:
        """分类错误"""
        error_str = str(error).lower()
        
        for pattern in cls.TRANSIENT_PATTERNS:
            if pattern in error_str:
                return ErrorCategory.TRANSIENT
        
        for pattern in cls.LOGICAL_PATTERNS:
            if pattern in error_str:
                return ErrorCategory.LOGICAL
        
        for pattern in cls.FATAL_PATTERNS:
            if pattern in error_str:
                return ErrorCategory.FATAL
        
        return ErrorCategory.UNKNOWN


class RecoveryStrategy:
    """恢复策略"""
    
    def __init__(self, agent, model):
        self.agent = agent
        self.model = model
    
    async def handle_error(
        self,
        error: Exception,
        context: dict,
        retry_count: int = 0
    ) -> RecoveryResult:
        """处理错误"""
        
        category = ErrorClassifier.classify(error)
        
        if category == ErrorCategory.TRANSIENT:
            return await self._handle_transient(error, context, retry_count)
        
        elif category == ErrorCategory.LOGICAL:
            return await self._handle_logical(error, context)
        
        elif category == ErrorCategory.FATAL:
            return await self._handle_fatal(error, context)
        
        else:
            return await self._handle_unknown(error, context)
    
    async def _handle_transient(self, error, context, retry_count) -> RecoveryResult:
        """处理暂时性错误 - 重试"""
        
        if retry_count >= 3:
            return RecoveryResult(
                action="escalate",
                message=f"重试{retry_count}次后仍然失败: {error}"
            )
        
        # 计算退避时间
        delay = (2 ** retry_count) + random.uniform(0, 1)
        
        return RecoveryResult(
            action="retry",
            delay=delay,
            message=f"网络问题，{delay:.1f}秒后重试..."
        )
    
    async def _handle_logical(self, error, context) -> RecoveryResult:
        """处理逻辑错误 - 重新规划"""
        
        # 让LLM分析错误并生成新策略
        analysis_prompt = f"""
        工具调用失败，需要调整策略。
        
        ## 原始目标
        {context.get('goal')}
        
        ## 失败的操作
        工具: {context.get('tool_name')}
        参数: {context.get('tool_args')}
        
        ## 错误信息
        {str(error)}
        
        ## 已有的观察结果
        {context.get('observations', [])}
        
        请分析：
        1. 为什么这个操作失败了？
        2. 有没有替代方案？
        3. 下一步应该怎么做？
        
        输出你的分析和建议的下一步行动。
        """
        
        analysis = await self.model.ainvoke(analysis_prompt)
        
        # 提取新的行动计划
        new_action = self._extract_new_action(analysis)
        
        if new_action:
            return RecoveryResult(
                action="replanning",
                new_action=new_action,
                reasoning=analysis.content,
                message="调整策略中..."
            )
        else:
            return RecoveryResult(
                action="skip",
                message=f"无法完成此操作: {error}，跳过继续"
            )
    
    async def _handle_fatal(self, error, context) -> RecoveryResult:
        """处理致命错误 - 请求帮助"""
        
        return RecoveryResult(
            action="human_help",
            message=f"遇到无法自动解决的问题: {error}",
            suggestion="请检查配置或联系管理员"
        )


# 在 ReAct 循环中使用
async def react_loop_with_recovery(state, runtime):
    recovery = RecoveryStrategy(agent, model)
    
    while iteration < max_iterations:
        # ... thought阶段 ...
        
        if thought.decision == "need_action":
            try:
                observation = await execute_tool(action)
            except Exception as e:
                # 错误恢复
                result = await recovery.handle_error(
                    error=e,
                    context={
                        "goal": state["understanding"]["high_level_goal"],
                        "tool_name": action.tool_name,
                        "tool_args": action.tool_args,
                        "observations": state.get("observations", [])
                    },
                    retry_count=retry_count
                )
                
                if result.action == "retry":
                    await asyncio.sleep(result.delay)
                    continue  # 重试
                
                elif result.action == "replanning":
                    # 使用新的行动
                    action = result.new_action
                    writer({
                        "type": "recovery",
                        "message": result.reasoning
                    })
                    continue
                
                elif result.action == "skip":
                    # 跳过此步骤
                    observations.append({
                        "action": action.tool_name,
                        "result": f"跳过: {result.message}"
                    })
                    continue
                
                elif result.action == "human_help":
                    # 需要人工介入
                    return await request_human_help(state, result)
```

### 3.4 实施建议

**Phase 1** (1周): 错误分类
- 实现ErrorClassifier
- 区分暂时性/逻辑/致命错误

**Phase 2** (1-2周): 重新规划能力
- 实现_handle_logical
- LLM分析错误生成新策略

**Phase 3** (2周): 完整恢复流程
- 集成到ReAct循环
- 用户通知机制

---

## 四、思维透明度与交互体验

### 4.1 当前状态：黑盒处理

```
用户问题
    ↓
[内部处理，用户看不到]
    ↓
最终答案
```

**用户体验问题**:
- 不知道Agent在做什么
- 等待时间焦虑
- 结果不符预期时难以理解原因

### 4.2 Moltbot的透明度

```
用户问题
    ↓
<thinking>
我需要先理解用户想要什么...
他问的是关于仓库瓶颈的问题...
我应该先查看各环节状态...
</thinking>
    ↓
调用工具: get_warehouse_stages
    ↓
<thinking>
得到了5个环节，分拣环节看起来有问题...
让我深入看看分拣的具体数据...
</thinking>
    ↓
调用工具: get_stage_status("分拣")
    ↓
<thinking>
分拣效率只有60%，远低于正常水平...
原因可能是...
</thinking>
    ↓
最终答案
```

### 4.3 🔧 改进方案：可见的思维过程

```python
# app/agent/transparency/thinking.py

class ThinkingStream:
    """
    思维流输出器
    
    支持三种模式：
    1. full: 完整输出所有思考
    2. summary: 输出简化的思考摘要
    3. off: 不输出思考（仅进度）
    """
    
    def __init__(self, mode: str = "summary"):
        self.mode = mode
        self.writer = get_stream_writer()
    
    def emit_thought(self, thought: str, stage: str):
        """输出思考"""
        
        if self.mode == "off":
            return
        
        if self.mode == "full":
            # 完整输出
            self.writer({
                "type": "thinking",
                "stage": stage,
                "content": thought,
                "timestamp": datetime.now().isoformat()
            })
        
        elif self.mode == "summary":
            # 简化输出
            summary = self._summarize_thought(thought)
            self.writer({
                "type": "thinking_summary",
                "stage": stage,
                "content": summary
            })
    
    def emit_action(self, tool_name: str, reason: str):
        """输出行动"""
        self.writer({
            "type": "action",
            "tool": tool_name,
            "reason": reason
        })
    
    def emit_observation(self, result_summary: str):
        """输出观察结果"""
        self.writer({
            "type": "observation",
            "summary": result_summary
        })
    
    def emit_progress(self, message: str, percentage: Optional[int] = None):
        """输出进度"""
        self.writer({
            "type": "progress",
            "message": message,
            "percentage": percentage
        })
    
    def _summarize_thought(self, thought: str) -> str:
        """简化思考内容"""
        # 提取关键句子
        lines = thought.strip().split('\n')
        key_lines = [l for l in lines if any(
            kw in l for kw in ['因此', '所以', '需要', '发现', '接下来']
        )]
        
        if key_lines:
            return key_lines[0][:100]
        
        return thought[:100] + "..."


# 在前端展示
"""
┌─────────────────────────────────────────────────┐
│  🤔 思考中...                                    │
│                                                 │
│  💭 "我需要先了解各环节的当前状态..."            │
│                                                 │
│  🔧 调用工具: get_warehouse_stages               │
│     原因: 获取仓库的所有环节列表                 │
│                                                 │
│  📊 结果: 获取到5个环节（分拣、上架、打包...）   │
│                                                 │
│  💭 "分拣环节的数据看起来异常，需要深入..."      │
│                                                 │
│  🔧 调用工具: get_stage_status                   │
│     参数: stage="分拣"                          │
│                                                 │
│  ━━━━━━━━━━━━━━━━━━ 75% ━━━━━━━━━━              │
└─────────────────────────────────────────────────┘
"""
```

### 4.4 中断与恢复能力

```python
# app/agent/control/interruption.py

class InterruptibleAgent:
    """
    可中断的Agent
    
    支持：
    1. 用户主动中断
    2. 超时自动中断
    3. 中断后恢复
    """
    
    def __init__(self, agent):
        self.agent = agent
        self.abort_controller = None
        self.checkpoint = None
    
    async def run_interruptible(self, state, runtime):
        """可中断的运行"""
        
        self.abort_controller = AbortController()
        
        try:
            async for chunk in self.agent.astream(state, runtime):
                # 检查是否被中断
                if self.abort_controller.is_aborted:
                    # 保存检查点
                    await self._save_checkpoint(state, chunk)
                    
                    return InterruptedResult(
                        message="操作已中断",
                        can_resume=True,
                        checkpoint_id=self.checkpoint.id
                    )
                
                yield chunk
        
        except TimeoutError:
            await self._save_checkpoint(state, None)
            return InterruptedResult(
                message="操作超时",
                can_resume=True
            )
    
    async def resume_from_checkpoint(self, checkpoint_id: str):
        """从检查点恢复"""
        
        checkpoint = await self._load_checkpoint(checkpoint_id)
        
        if not checkpoint:
            raise ValueError("检查点不存在或已过期")
        
        # 恢复状态
        state = checkpoint.state
        
        # 从中断点继续
        return await self.run_interruptible(state, checkpoint.runtime)
    
    def abort(self, reason: str = "user_request"):
        """中断执行"""
        if self.abort_controller:
            self.abort_controller.abort(reason)
    
    async def _save_checkpoint(self, state, current_chunk):
        """保存检查点"""
        self.checkpoint = Checkpoint(
            id=str(uuid.uuid4()),
            state=state,
            current_chunk=current_chunk,
            saved_at=datetime.now(),
            expires_at=datetime.now() + timedelta(hours=1)
        )
        
        await self.storage.save_checkpoint(self.checkpoint)


# API接口
@app.post("/chat/interrupt")
async def interrupt_chat(session_id: str):
    """中断当前对话"""
    agent = get_agent_for_session(session_id)
    agent.abort(reason="user_request")
    
    return {"status": "interrupted", "can_resume": True}

@app.post("/chat/resume")
async def resume_chat(checkpoint_id: str):
    """恢复对话"""
    agent = get_agent_for_session(session_id)
    
    return await agent.resume_from_checkpoint(checkpoint_id)
```

### 4.5 实施建议

**Phase 1** (1周): 思维输出
- ThinkingStream实现
- 前端展示适配

**Phase 2** (1周): 进度反馈
- 详细进度信息
- 预估完成时间

**Phase 3** (1-2周): 中断恢复
- AbortController
- 检查点保存/恢复

---

## 五、工具组合与动态发现

### 5.1 当前问题

```python
# 您的当前实现
# 工具在Agent初始化时硬编码
tools = [
    tool1, tool2, tool3, ...  # 固定列表
]

# 问题1: 无法根据任务动态选择工具子集
# 问题2: 无法组合工具形成新能力
# 问题3: 新工具需要修改代码
```

### 5.2 🔧 改进方案：动态工具系统

```python
# app/agent/tools/dynamic_tools.py

class DynamicToolRegistry:
    """
    动态工具注册表
    
    特性：
    1. 运行时注册新工具
    2. 根据任务过滤工具
    3. 工具组合成复合能力
    """
    
    def __init__(self):
        self.tools = {}
        self.tool_categories = defaultdict(list)
        self.tool_capabilities = {}
    
    def register(
        self,
        tool: BaseTool,
        category: str,
        capabilities: List[str],
        requires: List[str] = None
    ):
        """注册工具"""
        self.tools[tool.name] = tool
        self.tool_categories[category].append(tool.name)
        self.tool_capabilities[tool.name] = {
            "capabilities": capabilities,
            "requires": requires or [],
            "usage_count": 0,
            "success_rate": 1.0
        }
    
    def get_tools_for_task(self, task_description: str) -> List[BaseTool]:
        """根据任务获取相关工具"""
        
        # 1. 分析任务需要的能力
        required_capabilities = self._analyze_required_capabilities(task_description)
        
        # 2. 匹配工具
        matched_tools = []
        for tool_name, meta in self.tool_capabilities.items():
            if any(cap in meta["capabilities"] for cap in required_capabilities):
                matched_tools.append(self.tools[tool_name])
        
        # 3. 添加常用工具
        frequently_used = self._get_frequently_used_tools(top_n=3)
        for tool in frequently_used:
            if tool not in matched_tools:
                matched_tools.append(tool)
        
        return matched_tools
    
    def _analyze_required_capabilities(self, task: str) -> List[str]:
        """分析任务需要的能力"""
        capabilities = []
        
        capability_keywords = {
            "data_query": ["查询", "数据", "多少", "统计"],
            "trend_analysis": ["趋势", "变化", "对比", "分析"],
            "realtime_status": ["实时", "当前", "状态", "现在"],
            "anomaly_detection": ["异常", "问题", "瓶颈", "警告"],
            "optimization": ["优化", "建议", "改进", "提升"]
        }
        
        for cap, keywords in capability_keywords.items():
            if any(kw in task for kw in keywords):
                capabilities.append(cap)
        
        return capabilities or ["general"]


class ToolComposer:
    """
    工具组合器
    
    将多个工具组合成复合能力
    """
    
    @staticmethod
    def compose_data_comparison_tool(
        query_tool: BaseTool,
        analyzer: Any
    ) -> BaseTool:
        """
        组合：数据查询 + 对比分析
        
        单独使用query_tool需要用户多次调用
        组合后一次调用完成查询+对比
        """
        
        @tool("compare_data")
        async def compare_data(
            warehouse: str,
            metric: str,
            period1: str,
            period2: str
        ) -> dict:
            """对比两个时段的数据"""
            
            # 查询两个时段
            data1 = await query_tool.ainvoke({
                "warehouse": warehouse,
                "date_range": period1
            })
            
            data2 = await query_tool.ainvoke({
                "warehouse": warehouse,
                "date_range": period2
            })
            
            # 对比分析
            comparison = analyzer.compare(data1, data2, metric)
            
            return {
                "period1": data1,
                "period2": data2,
                "comparison": comparison,
                "insights": comparison.get_insights()
            }
        
        return compare_data
    
    @staticmethod
    def compose_anomaly_investigation_tool(
        status_tool: BaseTool,
        history_tool: BaseTool,
        detector: Any
    ) -> BaseTool:
        """
        组合：状态查询 + 历史查询 + 异常检测
        
        一次调用完成：查状态 → 查历史 → 检测异常 → 分析原因
        """
        
        @tool("investigate_anomaly")
        async def investigate_anomaly(
            warehouse: str,
            stage: str
        ) -> dict:
            """深度调查异常"""
            
            # 并行查询当前状态和历史
            current, history = await asyncio.gather(
                status_tool.ainvoke({"warehouse": warehouse, "stage": stage}),
                history_tool.ainvoke({"warehouse": warehouse, "stage": stage, "days": 7})
            )
            
            # 异常检测
            anomalies = detector.detect(current, history)
            
            # 根因分析
            if anomalies:
                root_causes = detector.analyze_root_causes(anomalies, history)
            else:
                root_causes = []
            
            return {
                "current_status": current,
                "historical_baseline": history.get_baseline(),
                "anomalies": anomalies,
                "root_causes": root_causes,
                "recommendations": detector.get_recommendations(anomalies)
            }
        
        return investigate_anomaly
```

### 5.3 实施建议

**Phase 1** (1周): 动态注册
- DynamicToolRegistry实现
- 工具元数据标注

**Phase 2** (1-2周): 任务匹配
- 能力分析
- 智能工具选择

**Phase 3** (2周): 工具组合
- 常用组合预定义
- 动态组合能力

---

## 六、总结：优先级排序

### 🔴 P0 - 必须改进（决定智能水平）

| 改进项 | 预计工期 | 预期效果 |
|--------|---------|----------|
| **ReAct推理循环** | 2-4周 | 从"计划执行"升级到"动态思考"，智能感显著提升 |
| **错误恢复与重规划** | 2周 | 遇到意外能自动调整，更像专家 |
| **思维透明度** | 1周 | 用户能看到推理过程，信任感大增 |

### 🟡 P1 - 强烈推荐（提升体验）

| 改进项 | 预计工期 | 预期效果 |
|--------|---------|----------|
| **Heartbeat自主检查** | 1-2周 | Agent主动发现问题，不再被动 |
| **Cron定时任务** | 1周 | 自动日报/周报，解放用户 |
| **中断与恢复** | 1-2周 | 用户掌控感，不再只能干等 |

### 🟢 P2 - 推荐（完善系统）

| 改进项 | 预计工期 | 预期效果 |
|--------|---------|----------|
| **动态工具系统** | 2周 | 更灵活的工具使用 |
| **Hook事件系统** | 2周 | 可扩展的自动化 |
| **工具组合能力** | 2周 | 复合能力，减少调用次数 |

---

## 七、实施路线图

```
┌─────────────────────────────────────────────────────────────────┐
│                        实施路线图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Month 1: 核心智能升级                                          │
│  ├─ Week 1-2: ReAct循环基础实现                                 │
│  ├─ Week 3: 思维透明度                                          │
│  └─ Week 4: 错误恢复机制                                        │
│                                                                 │
│  Month 2: 自主行为能力                                          │
│  ├─ Week 1: Heartbeat                                          │
│  ├─ Week 2: Cron定时任务                                       │
│  ├─ Week 3: 中断与恢复                                         │
│  └─ Week 4: 测试与优化                                         │
│                                                                 │
│  Month 3: 高级特性                                             │
│  ├─ Week 1-2: 动态工具系统                                     │
│  ├─ Week 3: Hook事件系统                                       │
│  └─ Week 4: 工具组合能力                                       │
│                                                                 │
│  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│  预期成果：                                                     │
│  ✅ 智能水平达到/超越Moltbot                                    │
│  ✅ 交互体验显著提升                                            │
│  ✅ 用户满意度大幅提高                                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、核心代码模板

以下是最关键的ReAct循环核心代码，可直接用于改造：

```python
# app/agent/react/core.py

"""
ReAct循环核心实现
替代原有的 plan_node + step_node + synthesize_node
"""

from typing import TypedDict, List, Literal, Optional
from langchain_core.messages import AIMessage, HumanMessage
from langgraph.config import get_stream_writer

class ThoughtResult(TypedDict):
    reasoning: str
    decision: Literal["need_action", "final_answer", "stuck", "clarify"]
    next_action: Optional[dict]  # {"tool_name": "...", "tool_args": {...}}
    final_answer: Optional[str]
    clarification_question: Optional[str]

class Observation(TypedDict):
    iteration: int
    thought: str
    action: str
    action_args: dict
    result: any
    error: Optional[str]

async def react_loop_node(state: ConversationState, *, runtime) -> dict:
    """
    ReAct循环节点
    
    替代Complex路径的 plan_node + step_node + synthesize_node
    """
    
    MAX_ITERATIONS = 10
    
    writer = get_stream_writer()
    understanding = state.get("understanding", {})
    enriched_context = state.get("enriched_context", {})
    user_memory = state.get("user_memory_context", {})
    observations: List[Observation] = []
    
    goal = understanding.get("high_level_goal", "回答用户问题")
    
    # 获取可用工具
    available_tools = runtime.tools or []
    tool_map = {t.name: t for t in available_tools}
    
    for iteration in range(1, MAX_ITERATIONS + 1):
        
        # === THOUGHT 阶段 ===
        thought_result = await generate_thought(
            goal=goal,
            user_message=state["messages"][-1].content,
            observations=observations,
            enriched_context=enriched_context,
            user_memory=user_memory,
            available_tools=[t.name for t in available_tools],
            iteration=iteration,
            max_iterations=MAX_ITERATIONS
        )
        
        # 输出思考过程
        writer({
            "type": "thinking",
            "iteration": iteration,
            "content": thought_result["reasoning"]
        })
        
        # === 决策分支 ===
        
        if thought_result["decision"] == "final_answer":
            # 可以给出最终答案
            return {
                "messages": [
                    AIMessage(content=thought_result["final_answer"])
                ]
            }
        
        if thought_result["decision"] == "clarify":
            # 需要用户澄清
            return {
                "messages": [
                    AIMessage(content=thought_result["clarification_question"])
                ],
                "awaiting_clarification": True
            }
        
        if thought_result["decision"] == "stuck":
            # 无法继续
            return {
                "messages": [
                    AIMessage(content="抱歉，我遇到了困难，无法完成这个任务。")
                ]
            }
        
        if thought_result["decision"] == "need_action":
            # === ACTION 阶段 ===
            action = thought_result["next_action"]
            tool_name = action["tool_name"]
            tool_args = action.get("tool_args", {})
            
            writer({
                "type": "action",
                "tool": tool_name,
                "args": tool_args,
                "reason": action.get("reason", "")
            })
            
            # 执行工具
            try:
                tool = tool_map.get(tool_name)
                if not tool:
                    raise ValueError(f"工具 {tool_name} 不存在")
                
                result = await tool.ainvoke(tool_args)
                
                # === OBSERVATION 阶段 ===
                observations.append({
                    "iteration": iteration,
                    "thought": thought_result["reasoning"],
                    "action": tool_name,
                    "action_args": tool_args,
                    "result": result,
                    "error": None
                })
                
                writer({
                    "type": "observation",
                    "success": True,
                    "summary": summarize_result(result)
                })
                
            except Exception as e:
                # 错误处理
                error_str = str(e)
                
                observations.append({
                    "iteration": iteration,
                    "thought": thought_result["reasoning"],
                    "action": tool_name,
                    "action_args": tool_args,
                    "result": None,
                    "error": error_str
                })
                
                writer({
                    "type": "observation",
                    "success": False,
                    "error": error_str
                })
                
                # 继续循环，让下一次thought处理错误
    
    # 超过最大迭代，强制总结
    final_answer = await force_summarize(observations, goal)
    
    return {
        "messages": [AIMessage(content=final_answer)]
    }


async def generate_thought(
    goal: str,
    user_message: str,
    observations: List[Observation],
    enriched_context: dict,
    user_memory: dict,
    available_tools: List[str],
    iteration: int,
    max_iterations: int
) -> ThoughtResult:
    """
    生成思考
    """
    
    # 格式化已有观察
    obs_text = format_observations(observations)
    
    # 格式化可用工具
    tools_text = "\n".join([f"- {t}" for t in available_tools])
    
    prompt = f"""
你是一个专业的仓库分析助手，正在帮助用户完成以下任务：

## 用户原始问题
{user_message}

## 任务目标
{goal}

## 当前迭代
第 {iteration} 轮，最多 {max_iterations} 轮

## 已有的观察结果
{obs_text if obs_text else "暂无"}

## 可用工具
{tools_text}

## 背景上下文
{enriched_context.get("system_prompt", "")[:500]}

## 用户偏好
{format_user_memory(user_memory)}

---

请进行下一步思考：

1. 分析：我现在知道了什么？我还需要了解什么？
2. 决策：下一步应该怎么做？
   - 如果已经有足够信息回答，输出 final_answer
   - 如果需要更多数据，选择一个工具调用
   - 如果问题不清楚，输出澄清问题
   - 如果完全无法继续，输出 stuck

请严格按照以下JSON格式输出：

```json
{{
    "reasoning": "你的思考过程（2-3句话）",
    "decision": "need_action | final_answer | clarify | stuck",
    "next_action": {{
        "tool_name": "工具名称",
        "tool_args": {{}},
        "reason": "为什么选择这个工具"
    }},
    "final_answer": "如果decision是final_answer，这里是最终答案",
    "clarification_question": "如果decision是clarify，这里是澄清问题"
}}
```
"""
    
    # 使用结构化输出
    response = await model.with_structured_output(ThoughtResult).ainvoke(prompt)
    
    return response


def format_observations(observations: List[Observation]) -> str:
    """格式化观察结果"""
    if not observations:
        return ""
    
    lines = []
    for obs in observations:
        lines.append(f"### 第{obs['iteration']}轮")
        lines.append(f"思考: {obs['thought']}")
        lines.append(f"行动: {obs['action']}({obs['action_args']})")
        
        if obs['error']:
            lines.append(f"结果: ❌ 错误 - {obs['error']}")
        else:
            lines.append(f"结果: ✅ {summarize_result(obs['result'])}")
        
        lines.append("")
    
    return "\n".join(lines)


def summarize_result(result: any) -> str:
    """总结工具结果"""
    if result is None:
        return "无结果"
    
    if isinstance(result, dict):
        # 提取关键字段
        keys = list(result.keys())[:5]
        return f"返回字典，包含: {', '.join(keys)}"
    
    if isinstance(result, list):
        return f"返回列表，{len(result)}项"
    
    result_str = str(result)
    if len(result_str) > 100:
        return result_str[:100] + "..."
    
    return result_str
```

---

**文档结束**

这份文档聚焦于核心智能能力和交互体验的提升。最关键的改进是将Complex路径从"计划-执行"模式升级为"ReAct循环"模式，这将从根本上提升Agent的智能水平。

如需进一步讨论任何技术细节，请随时联系。
