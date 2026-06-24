# 第五部分：Agent 运行机制分析

## 5.1 Agent 启动流程

```mermaid
flowchart TD
    A[hermes CLI 启动] --> B[load_cli_config 加载配置]
    B --> C[创建 AIAgent 实例]
    C --> D[初始化 MemoryManager]
    D --> E[连接 SessionDB]
    E --> F[加载工具定义]
    F --> G[构建系统提示]
    G --> H[就绪，等待输入]
    
    H --> I{收到用户消息}
    I --> J[run_conversation]
```

## 5.2 主循环状态机

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> Running : run_conversation()
    Running --> WaitingForModel : LLM API 调用
    WaitingForModel --> Running : 收到响应
    Running --> ToolExecution : 检测到工具调用
    ToolExecution --> Running : 工具执行完成
    ToolExecution --> WaitingForModel : 继续 LLM 调用
    Running --> Compressing : 上下文超限
    Compressing --> Running : 压缩完成
    Running --> Interrupted : Ctrl+C / /stop
    Running --> Completed : 无更多输出
    Completed --> Idle : 返回结果
    Interrupted --> Idle : 清理
    Error --> Idle : 记录错误
```

## 5.3 核心运行机制

### 5.3.1 ReAct 模式

Hermes Agent 采用 **ReAct (Reasoning + Acting)** 模式：

```python
# 伪代码展示 ReAct 循环
while iterations < max_iterations:
    # 1. Reason - 生成推理
    response = llm.chat(messages)
    
    # 2. Act - 如果有工具调用则执行
    if response.tool_calls:
        for call in response.tool_calls:
            result = execute_tool(call.name, call.args)
            messages.append(tool_result(result))
    else:
        # 没有工具调用，直接返回
        return response.content
```

### 5.3.2 Plan-Execute 模式

对于复杂任务，支持 **Plan-Execute** 模式：

```python
# 子 Agent 委托实现计划执行
result = delegate_task(
    goal="分析代码库并生成报告",
    context="需要分析的文件列表...",
    role="orchestrator"  # 可再分派
)
```

### 5.3.3 Reflection 机制

```mermaid
flowchart LR
    A[执行工具] --> B[收集结果]
    B --> C{成功?}
    C -->|是| D[继续]
    C -->|否| E{重试次数 < 3?}
    E -->|是| F[重试]
    F --> A
    E -->|否| G[使用备用方案]
    G --> H[记录错误]
```

### 5.3.4 迭代控制

```python
class IterationBudget:
    def __init__(self, max_iterations: int = 90):
        self.max_iterations = max_iterations
        self.remaining = max_iterations
    
    def consume(self, n=1) -> bool:
        """消耗迭代次数，返回是否还有剩余"""
        if self.remaining >= n:
            self.remaining -= n
            return True
        return False
    
    def grace_call(self) -> bool:
        """最后宽限一次调用"""
        if self.remaining <= 0:
            self.remaining = -1  # 允许一次
            return True
        return False
```

## 5.4 Sequence Diagram - 完整对话流程

```mermaid
sequenceDiagram
    participant U as User
    participant CLI as CLI/TUI
    participant AG as AIAgent
    participant LOOP as ConversationLoop
    participant MEM as MemoryManager
    participant TOOL as ToolRegistry
    participant LLM as LLM Provider
    participant DB as SessionDB

    U->>CLI: 用户消息
    CLI->>AG: run_conversation(message)
    
    Note over AG: 1. 前置准备
    
    AG->>MEM: prefetch_all(query)
    MEM-->>AG: memory_context
    AG->>LOOP: build_turn_context()
    
    Note over LOOP: 2. 构建消息列表
    
    LOOP->>DB: get_session_messages()
    DB-->>LOOP: messages[]
    LOOP->>LOOP: build_system_prompt()
    LOOP->>LOOP: build_context_files()
    
    Note over LOOP: 3. LLM 调用循环
    
    LOOP->>LLM: chat.completions.create()
    LLM-->>LOOP: response
    
    alt 有工具调用
        LOOP->>LOOP: handle_tool_calls()
        LOOP->>TOOL: handle_function_call()
        TOOL->>TOOL: execute_tool()
        TOOL-->>LOOP: result
        LOOP->>LOOP: append result to messages
        LOOP->>LOOP: 继续 LLM 调用
    else 无工具调用
        LOOP-->>AG: final_response
    end
    
    Note over AG: 4. 后置处理
    
    AG->>MEM: sync_all(user, assistant)
    AG->>MEM: queue_prefetch()
    AG->>DB: save_messages()
    
    AG-->>CLI: result
    CLI-->>U: 显示响应
```

## 5.5 Loop 流程图

```mermaid
flowchart TD
    START[开始回合] --> LOAD[加载上下文]
    LOAD --> BUILD[构建消息列表]
    
    BUILD --> ADD_SYS[添加系统提示]
    ADD_SYS --> ADD_MEM[添加记忆上下文]
    ADD_MEM --> ADD_MSG[添加历史消息]
    ADD_MSG --> ADD_TOOLS[添加工具定义]
    ADD_TOOLS --> CALL_LLM[调用 LLM]
    
    CALL_LLM --> RESP{收到响应}
    
    RESP -->|有工具调用| EXEC[执行工具]
    EXEC --> BUILD_RES[构建结果消息]
    BUILD_RES --> CHECK_BUDGET{预算检查}
    CHECK_BUDGET -->|还有预算| ADD_MSG
    CHECK_BUDGET -->|预算耗尽| RETRY{重试?}
    RETRY -->|是| RETRY_LLM[使用更短重试]
    RETRY_LLM --> CALL_LLM
    RETRY -->|否| ERR[返回错误]
    
    RESP -->|无工具调用| FINAL[返回最终结果]
    
    FINAL --> POST[后置处理]
    POST --> SYNC[同步记忆]
    POST --> SAVE[保存会话]
    POST --> END[回合结束]
    
    ERR --> FINAL
```

## 5.6 失败恢复机制

```python
class TurnRetryState:
    """单轮重试状态"""
    
    def __init__(self):
        self.attempt = 0
        self.last_error: Optional[str] = None
        self.fallback_reason: Optional[FailoverReason] = None
    
    def should_retry(self, max_retries=3) -> bool:
        return self.attempt < max_retries
    
    def record_failure(self, error: str, reason: FailoverReason):
        self.attempt += 1
        self.last_error = error
        self.fallback_reason = reason

# 错误分类
class FailoverReason(Enum):
    RATE_LIMIT = "rate_limit"
    CONTEXT_OVERFLOW = "context_overflow"
    MODEL_ERROR = "model_error"
    TIMEOUT = "timeout"
    AUTH_ERROR = "auth_error"
```

## 5.7 上下文压缩触发条件

```python
def should_compress(agent) -> bool:
    """判断是否需要压缩上下文"""
    
    # 1. Token 数量检查
    total_tokens = estimate_messages_tokens(messages)
    if total_tokens > agent.context_length * 0.85:
        return True
    
    # 2. 消息数量检查
    if len(messages) > 50:
        return True
    
    # 3. 工具调用次数检查
    tool_calls = count_tool_calls(messages)
    if tool_calls > 30:
        return True
    
    return False
```

## 5.8 工具执行流程

```mermaid
flowchart TD
    A[LLM 请求工具调用] --> B[解析工具名和参数]
    B --> C{工具存在?}
    C -->|否| D[返回错误]
    C -->|是| E{参数有效?}
    E -->|否| F[返回参数错误]
    E -->|是| G{环境满足?}
    G -->|否| H[返回环境错误]
    G -->|是| I[执行工具]
    
    I --> J{执行成功?}
    J -->|是| K[返回结果]
    J -->|否| L[记录错误]
    L --> M{可重试?}
    M -->|是| N[重试]
    N --> I
    M -->|否| O[返回错误]
```
