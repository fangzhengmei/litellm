# LiteLLM 可观测性回调与异步日志管线分析

## 目录

1. [概述](#概述)
2. [支持的可观测性平台集成](#支持的可观测性平台集成)
3. [回调注册机制](#回调注册机制)
4. [异步日志事件处理流程](#异步日志事件处理流程)
5. [架构总结](#架构总结)

---

## 概述

LiteLLM 提供了一套完善的可观测性回调系统，允许用户将 LLM 调用的详细信息发送到各种外部监控和分析平台。该系统采用**异步非阻塞**设计，确保日志操作不会影响主请求的延迟。

核心组件：
- **`LoggingCallbackManager`**: 集中管理回调的注册和路由
- **`LoggingWorker`**: 后台异步任务队列处理器
- **`CustomLogger`**: 自定义回调的基类
- **`Logging`**: 日志事件的核心处理类

---

## 支持的可观测性平台集成

LiteLLM 支持 50+ 种可观测性平台和工具的集成。以下是完整列表：

### 1. 可观测性与追踪平台

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **LangFuse** | `langfuse`, `langfuse_otel` | `integrations/langfuse/` | 完整的 LLM 可观测性、成本追踪、提示管理 |
| **LangSmith** | `langsmith` | `integrations/langsmith.py` | LLM 应用调试和评估 |
| **LangTrace** | `langtrace` | `integrations/langtrace.py` | 开源 LLM 可观测性 |
| **Arize** | `arize`, `arize_phoenix` | `integrations/arize/` | ML 可观测性和模型监控 |
| **Traceloop** | `traceloop` | `integrations/traceloop.py` | OpenLLMetry 标准实现 |
| **Weave** | `weave_otel` | `integrations/weave/` | Weights & Biases 可观测性 |
| **Literal AI** | `literalai` | `integrations/literal_ai.py` | LLM 应用可观测性平台 |

### 2. 指标与监控平台

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **Prometheus** | `prometheus` | `integrations/prometheus.py` | 开源指标监控，提供 `/metrics` 端点 |
| **Datadog** | `datadog`, `datadog_metrics`, `datadog_llm_observability` | `integrations/datadog/` | 企业级可观测性，支持日志、指标、追踪 |
| **OpenTelemetry** | `otel` | `integrations/opentelemetry.py` | 开源可观测性框架 |
| **New Relic** | - | 通过 OpenTelemetry | 应用性能监控 |

### 3. 日志与事件平台

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **Logfire** | `logfire` | `integrations/logfire_logger.py` | Pydantic 团队的 LLM 日志平台 |
| **PostHog** | `posthog` | `integrations/posthog.py` | 产品分析和事件追踪 |
| **Helicone** | `helicone` | `integrations/helicone.py` | LLM 可观测性和成本管理 |
| **PromptLayer** | `promptlayer` | `integrations/prompt_layer.py` | LLM 请求日志和调试 |
| **Galileo** | `galileo` | `integrations/galileo.py` | LLM 质量监控 |
| **HumanLoop** | `humanloop` | `integrations/humanloop.py` | LLM 开发平台 |
| **Braintrust** | `braintrust` | `integrations/braintrust_logging.py` | LLM 评估平台 |
| **Athina** | `athina` | `integrations/athina.py` | LLM 可观测性和评估 |
| **Opik** | `opik` | `integrations/opik/opik.py` | Comet 的 LLM 可观测性 |
| **Lunary** | `lunary` | `integrations/lunary.py` | LLM 可观测性平台 |
| **AgentOps** | `agentops` | `integrations/agentops.py` | AI Agent 可观测性 |

### 4. 成本与使用追踪

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **OpenMeter** | `openmeter` | `integrations/openmeter.py` | 开源用量计量和计费 |
| **Lago** | `lago` | `integrations/lago.py` | 开源计费平台 |
| **CloudZero** | `cloudzero` | - | 云成本智能 |
| **Vantage** | `vantage` | `integrations/vantage/` | 云成本管理 |

### 5. 数据存储与冷存储

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **Amazon S3** | `s3`, `s3_v2` | `integrations/s3.py`, `integrations/s3_v2.py` | 对象存储，支持日志归档 |
| **Google Cloud Storage** | `gcs_bucket` | `integrations/gcs_bucket/` | GCS 存储集成 |
| **Azure Blob Storage** | `azure_storage` | `integrations/azure_storage/` | Azure 存储集成 |
| **DynamoDB** | `dynamodb` | `integrations/dynamodb.py` | NoSQL 数据库 |
| **Supabase** | `supabase` | `integrations/supabase.py` | 开源 Firebase 替代 |
| **Google Cloud Pub/Sub** | `gcs_pubsub` | `integrations/gcs_pubsub/` | 消息队列服务 |
| **Amazon SQS** | `aws_sqs` | `integrations/sqs.py` | 简单队列服务 |
| **Focus** | `focus` | `integrations/focus/` | 数据处理框架 |

### 6. 安全与合规

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **Azure Sentinel** | `azure_sentinel` | `integrations/azure_sentinel/` | 安全信息和事件管理 (SIEM) |
| **Levo** | `levo` | `integrations/levo/` | API 安全测试 |

### 7. 提示管理

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **Dotprompt** | `dotprompt` | `integrations/dotprompt.py` | 提示管理框架 |

### 8. 代码仓库与 CI/CD

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **GitLab** | `gitlab` | `integrations/gitlab/` | GitLab CI/CD 集成 |
| **Bitbucket** | `bitbucket` | - | Atlassian 代码仓库 |

### 9. 机器学习平台

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **MLflow** | `mlflow` | `integrations/mlflow.py` | 机器学习生命周期管理 |
| **Weights & Biases** | `wandb` | `integrations/weights_biases.py` | ML 实验追踪 |

### 10. Argilla (数据标注)

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **Argilla** | `argilla` | `integrations/argilla.py` | 开源数据标注平台 |

### 11. 深度评估

| 平台名称 | 回调标识符 | 集成文件 | 主要功能 |
|---------|-----------|---------|---------|
| **DeepEval** | `deepeval` | `integrations/deepeval/` | LLM 评估框架 |

### 12. 企业级功能 (Enterprise)

这些功能需要 LiteLLM Enterprise 许可证：

| 功能名称 | 回调标识符 | 描述 |
|---------|-----------|------|
| **PagerDuty 告警** | `pagerduty` | 事件告警和通知 |
| **邮件通知** | `resend_email`, `sendgrid_email`, `smtp_email` | 通过各种邮件服务发送通知 |
| **通用 API 回调** | `generic_api` | 自定义 Webhook 端点 |

### 13. 速率限制与预算管理 (Proxy/Router)

这些回调主要用于 Proxy 服务器和 Router：

| 功能名称 | 描述 |
|---------|------|
| `dynamic_rate_limiter`, `dynamic_rate_limiter_v3` | 动态速率限制 |
| `parallel_request_limiter`, `parallel_request_limiter_v3` | 并行请求限制 |
| `max_budget_per_session_limiter` | 会话级别预算限制 |
| `model_max_budget_limiter` | 模型级别预算限制 |
| `proxy_track_cost_callback` | Proxy 成本追踪 |
| `anthropic_cache_control_hook` | Anthropic 缓存控制 |
| `vector_store_pre_call_hook` | 向量存储预调用钩子 |

---

## 回调注册机制

### 1. 回调类型分类

LiteLLM 的回调系统按**调用时机**和**执行方式**分为多个类别：

```python
# 在 litellm/__init__.py 中定义的回调列表
input_callback: List[CALLBACK_TYPES] = []           # API 调用前执行
success_callback: List[CALLBACK_TYPES] = []         # 调用成功后执行 (同步)
failure_callback: List[CALLBACK_TYPES] = []         # 调用失败后执行 (同步)
_async_success_callback: List[CALLBACK_TYPES] = []  # 调用成功后执行 (异步)
_async_failure_callback: List[CALLBACK_TYPES] = []  # 调用失败后执行 (异步)
callbacks: List[CALLBACK_TYPES] = []                 # 通用回调 (成功+失败都执行)
service_callback: List[CALLBACK_TYPES] = []          # 服务级别回调
audit_log_callbacks: List[CALLBACK_TYPES] = []       # 审计日志回调
```

### 2. LoggingCallbackManager 核心功能

`LoggingCallbackManager` 是回调管理的核心类，位于 `litellm/litellm_core_utils/logging_callback_manager.py`。

#### 主要功能：

**2.1 自动路由同步/异步回调**

```python
def add_litellm_success_callback(self, callback):
    # 特殊字符串回调（如 dynamodb, openmeter）自动路由到异步
    if isinstance(callback, str) and callback in ("dynamodb", "openmeter"):
        self._safe_add_callback_to_list(
            callback=callback, parent_list=litellm._async_success_callback
        )
    # 检测函数是否为异步
    elif not isinstance(callback, str) and self._is_async_callable(callback):
        self._safe_add_callback_to_list(
            callback=callback, parent_list=litellm._async_success_callback
        )
    else:
        self._safe_add_callback_to_list(
            callback=callback, parent_list=litellm.success_callback
        )
```

**2.2 防重复注册**

```python
def _add_custom_logger_to_list(self, custom_logger, parent_list):
    # 通过类名和基础属性生成唯一键
    custom_logger_key = self._get_custom_logger_key(custom_logger)
    for existing_logger in parent_list:
        if (isinstance(existing_logger, CustomLogger) and 
            self._get_custom_logger_key(existing_logger) == custom_logger_key):
            verbose_logger.debug(f"Logger already exists, skipping")
            return
    parent_list.append(custom_logger)
```

**2.3 最大数量限制**

```python
MAX_CALLBACKS = 30  # 最大回调数量限制

def _check_callback_list_size(self, parent_list) -> bool:
    if len(parent_list) >= MAX_CALLBACKS:
        verbose_logger.warning(
            f"Cannot add callback - would exceed MAX_CALLBACKS limit of {MAX_CALLBACKS}"
        )
        return False
    return True
```

**2.4 Generic API 回调支持**

支持通过配置文件定义自定义 Webhook 回调：

```python
# litellm_settings:
#   success_callback: ["custom_callback_name"]
#
# callback_settings:
#   custom_callback_name:
#     callback_type: generic_api
#     endpoint: https://webhook-test.com/xxx
#     headers:
#       Authorization: Bearer sk-1234
#     max_retries: 3
#     retry_delay: 1.0
```

### 3. 回调注册 API

#### 3.1 全局注册方式

```python
import litellm

# 方式1: 字符串标识符（最简单）
litellm.success_callback = ["langfuse", "prometheus"]

# 方式2: 通过 LoggingCallbackManager
litellm.logging_callback_manager.add_litellm_success_callback("langfuse")
litellm.logging_callback_manager.add_litellm_async_success_callback("datadog")

# 方式3: 注册 CustomLogger 实例
from litellm.integrations.custom_logger import CustomLogger

class MyLogger(CustomLogger):
    async def async_log_success_event(self, kwargs, response_obj, start_time, end_time):
        print(f"Success: {kwargs}")

litellm.callbacks = [MyLogger()]

# 方式4: 注册函数
def my_callback(kwargs, response_obj, start_time, end_time):
    print(f"Callback called")

litellm.success_callback = [my_callback]
```

#### 3.2 单次调用注册 (动态回调)

```python
# 只对特定调用生效的回调
response = await litellm.acompletion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}],
    success_callback=["langfuse"],  # 仅本次调用使用
    failure_callback=["posthog"]
)
```

### 4. 回调执行优先级

回调执行的优先级顺序（从高到低）：

1. **动态回调** (`dynamic_async_success_callbacks` 等) - 单次调用指定的
2. **全局异步回调** (`litellm._async_success_callback`) - 通过 LoggingWorker 异步执行
3. **全局同步回调** (`litellm.success_callback`) - 在主线程同步执行

---

## 异步日志事件处理流程

### 1. 架构概览

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户 API 调用                                    │
│  litellm.acompletion() / litellm.completion()                             │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     Logging 类 (litellm_logging.py)                      │
│  - 构建 StandardLoggingPayload                                             │
│  - 计算 response_cost                                                      │
│  - 准备回调执行                                                             │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   logging_wrapper_async (utils.py:1192)                  │
│  - 检测异步回调是否需要执行                                                 │
│  - 将 async_success_handler 包装为 coroutine                              │
│  - 提交给 GLOBAL_LOGGING_WORKER                                           │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                   LoggingWorker (logging_worker.py)                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  asyncio.Queue (LoggingTask)                                         │  │
│  │  ┌─────────┬─────────┬─────────┬─────────┬─────────┐              │  │
│  │  │ Task 1  │ Task 2  │ Task 3  │  ...    │ Task N  │              │  │
│  │  └─────────┴─────────┴─────────┴─────────┴─────────┘              │  │
│  │                         ▲                                             │  │
│  │                         │ enqueue()                                   │  │
│  └─────────────────────────┼─────────────────────────────────────────────┘  │
│                            │                                                │
│                            ▼                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  _worker_loop() + Semaphore 并发控制                                  │  │
│  │  - 从 Queue 获取任务                                                   │  │
│  │  - 通过 contextvars 恢复原始上下文                                     │  │
│  │  - 执行 coroutine (带超时控制)                                         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      async_success_handler()                              │
│  - 遍历 litellm._async_success_callback + dynamic_async_success_callbacks │
│  - 对每个 CustomLogger 调用 async_log_success_event()                     │
│  - 处理流式响应的特殊逻辑                                                   │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                      外部可观测性平台                                      │
│  LangFuse ──►  API / SDK 调用                                             │
│  Prometheus ──► Counter/Histogram 更新                                    │
│  Datadog ──► 批量提交 + 定时 flush                                         │
│  S3 ──► 对象存储写入                                                        │
│  ... 等 50+ 平台                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2. LoggingWorker 核心实现

#### 2.1 核心配置参数

```python
# 常量定义 (constants.py)
LOGGING_WORKER_CONCURRENCY = 10              # 并发执行的任务数
LOGGING_WORKER_MAX_QUEUE_SIZE = 1000         # 队列最大容量
LOGGING_WORKER_MAX_TIME_PER_COROUTINE = 60.0  # 单个任务超时时间
LOGGING_WORKER_CLEAR_PERCENTAGE = 50          # 队列满时清理百分比
LOGGING_WORKER_AGGRESSIVE_CLEAR_COOLDOWN_SECONDS = 1.0  # 清理冷却时间
```

#### 2.2 任务入队机制

```python
def enqueue(self, coroutine: Coroutine) -> None:
    # 捕获当前上下文（contextvars）
    task = LoggingTask(
        coroutine=coroutine, 
        context=contextvars.copy_context()
    )
    
    try:
        self._queue.put_nowait(task)
    except asyncio.QueueFull:
        # 队列满时的处理策略
        self._handle_queue_full(task)

def _handle_queue_full(self, task: LoggingTask) -> None:
    if self._should_start_aggressive_clear():
        self._mark_aggressive_clear_started()
        # 异步执行队列清理：取出 50% 的任务直接处理
        asyncio.create_task(self._aggressively_clear_queue_async(task))
    else:
        # 冷却期间，延迟重试
        self._schedule_delayed_enqueue_retry(task)
```

#### 2.3 工作循环 (Worker Loop)

```python
async def _worker_loop(self) -> None:
    while True:
        # 1. 获取信号量（控制并发）
        await self._sem.acquire()
        try:
            # 2. 从队列获取任务
            task = await self._queue.get()
            
            # 3. 创建任务并追踪
            processing_task = asyncio.create_task(
                self._process_log_task(task, self._sem)
            )
            self._running_tasks.add(processing_task)
            processing_task.add_done_callback(self._running_tasks.discard)
        except Exception:
            self._sem.release()
            raise
```

#### 2.4 任务执行与上下文恢复

```python
async def _process_log_task(self, task: LoggingTask, sem: asyncio.Semaphore):
    try:
        if self._queue is not None:
            try:
                # 在原始上下文中执行任务
                # 这确保了 tracing 上下文、请求 ID 等被正确保留
                await asyncio.wait_for(
                    task["context"].run(asyncio.create_task, task["coroutine"]),
                    timeout=self.timeout,
                )
            except Exception as e:
                verbose_logger.exception(f"LoggingWorker error: {e}")
            finally:
                self._queue.task_done()
    finally:
        sem.release()
```

### 3. 完整数据流

#### 3.1 异步调用路径 (acompletion)

```
1. 用户调用 litellm.acompletion(...)
   │
   ▼
2. 实际 LLM API 调用完成
   │
   ▼
3. logging_wrapper_async() 被调用 (utils.py:1177)
   │
   ├──► 检查是否是 completion_with_fallbacks（防止重复日志）
   │
   └──► 提交到 LoggingWorker:
        GLOBAL_LOGGING_WORKER.ensure_initialized_and_enqueue(
            async_coroutine=logging_obj.async_success_handler(...)
        )
   │
   ▼
4. LoggingWorker 后台处理
   │
   ├──► enqueue() → 放入 asyncio.Queue
   │
   └──► _worker_loop() 消费队列
        │
        └──► 执行 async_success_handler()
             │
             ├──► 遍历所有 async callbacks
             │
             └──► 调用每个 callback 的 async_log_success_event()
                  │
                  └──► 发送到外部平台（LangFuse, Datadog 等）
```

#### 3.2 同步调用路径 (completion)

同步调用的日志处理有所不同：

```
1. 用户调用 litellm.completion(...)
   │
   ▼
2. API 调用完成后直接调用 success_handler()
   │
   ├──► 同步回调 (litellm.success_callback) 在主线程执行
   │
   └──► 异步回调会怎样？
        │
        └──► 注意：同步调用中，async callbacks 默认不会执行！
             除非显式配置或使用 acompletion。
```

### 4. StandardLoggingPayload 标准化

所有回调都接收一个标准化的日志对象 `StandardLoggingPayload`，位于 `litellm/types/utils.py`。

#### 4.1 核心字段

```python
class StandardLoggingPayload(TypedDict):
    # 基础信息
    model: str                          # 模型名称
    messages: List[Dict]                # 请求消息
    response: Optional[ModelResponse]   # 响应对象
    response_cost: float                # 计算的成本
    
    # 用量统计
    usage: Dict[str, int]               # token 使用量
    prompt_tokens: int                  # 输入 tokens
    completion_tokens: int              # 输出 tokens
    total_tokens: int                   # 总 tokens
    
    # 时间信息
    start_time: datetime                # 开始时间
    end_time: datetime                  # 结束时间
    
    # 标识信息
    litellm_call_id: str                # 调用唯一 ID
    litellm_trace_id: str               # 追踪 ID
    
    # 元数据
    metadata: Dict                      # 用户自定义元数据
    user: Optional[str]                 # 用户 ID
    cache_hit: bool                     # 是否缓存命中
    
    # 错误信息（失败时）
    error_str: Optional[str]            # 错误字符串
    original_exception: Optional[Exception]  # 原始异常
```

#### 4.2 构建流程

```python
# 在 _success_handler_helper_fn 中构建
def _process_hidden_params_and_response_cost(self, logging_result, start_time, end_time):
    # 1. 计算响应成本
    self.model_call_details["response_cost"] = self._response_cost_calculator(
        result=logging_result
    )
    
    # 2. 构建标准日志对象
    self.model_call_details["standard_logging_object"] = (
        self._build_standard_logging_payload(
            logging_result, start_time, end_time
        )
    )
    
    # 3. 触发事件（用于 OTEL 等）
    emit_standard_logging_payload(standard_logging_payload)
```

### 5. 流式响应处理

流式响应 (`stream=True`) 需要特殊处理：

#### 5.1 挑战

- 流式响应逐 chunk 返回
- 需要收集所有 chunk 才能计算完整的 token 使用量
- 用户需要看到实时响应，不能等待日志完成

#### 5.2 解决方案

```python
# streaming_iterator.py 中的处理
class CustomStreamWrapper:
    def __init__(self, ...):
        self.chunks = []  # 收集所有 chunk
    
    def __iter__(self):
        for chunk in self.stream:
            self.chunks.append(chunk)
            yield chunk
        
        # 流结束后，组装完整响应并触发日志
        self._on_stream_end()
    
    def _on_stream_end(self):
        # 组装完整响应
        complete_response = self._assemble_streaming_response()
        
        # 触发异步日志
        asyncio.create_task(
            self.logging_obj.async_success_handler(complete_response)
        )
```

### 6. 优雅关闭与退出处理

#### 6.1 atexit 注册

```python
class LoggingWorker:
    def __init__(self, ...):
        # 注册退出处理
        atexit.register(self._flush_on_exit)
```

#### 6.2 退出时的冲洗

```python
def _flush_on_exit(self):
    if self._queue is None or self._queue.empty():
        return
    
    queue_size = self._queue.qsize()
    self._safe_log("info", f"Flushing {queue_size} remaining events...")
    
    # 创建新的事件循环（原循环已关闭）
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    
    try:
        processed = 0
        while not self._queue.empty() and processed < MAX_ITERATIONS_TO_CLEAR_QUEUE:
            try:
                task = self._queue.get_nowait()
                # 同步执行剩余任务
                loop.run_until_complete(task["coroutine"])
                processed += 1
            except Exception:
                pass
    finally:
        loop.close()
```

---

## 架构总结

### 1. 设计亮点

**1.1 非阻塞设计**
- 异步回调通过 `LoggingWorker` 在后台队列执行
- 主请求路径不会被日志操作阻塞
- 带来 **+200 RPS** 的性能提升（官方数据）

**1.2 上下文保留**
- 使用 `contextvars` 捕获和恢复执行上下文
- 确保 tracing ID、请求元数据等在异步执行中正确传递

**1.3 限流与背压**
- `asyncio.Semaphore` 控制并发数
- 队列有界（默认 1000）防止内存无限增长
- 队列满时有策略：清理 50% 旧任务 + 延迟重试

**1.4 标准化与扩展性**
- `StandardLoggingPayload` 统一所有平台的日志格式
- `CustomLogger` 基类提供丰富的钩子方法
- 支持字符串、函数、类实例三种回调形式

### 2. 关键代码位置参考

| 组件 | 文件路径 | 主要职责 |
|------|---------|---------|
| LoggingCallbackManager | `litellm/litellm_core_utils/logging_callback_manager.py` | 回调注册、路由、去重 |
| LoggingWorker | `litellm/litellm_core_utils/logging_worker.py` | 异步任务队列执行 |
| Logging (核心日志类) | `litellm/litellm_core_utils/logging.py` | 构建日志对象、执行回调 |
| CustomLogger (基类) | `litellm/integrations/custom_logger.py` | 自定义回调基类，定义所有钩子 |
| 回调常量定义 | `litellm/__init__.py` | 回调列表、已知兼容回调 |
| logging_wrapper_async | `litellm/utils.py:1192` | 异步调用的日志包装 |

### 3. 常见使用模式

#### 模式1: 快速集成单个平台

```python
import litellm
import os

os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-xxx"
os.environ["LANGFUSE_SECRET_KEY"] = "sk-xxx"

litellm.success_callback = ["langfuse"]

# 后续所有调用都会自动记录到 LangFuse
response = litellm.completion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}]
)
```

#### 模式2: 多平台组合

```python
import litellm

# 同时使用 Prometheus + LangFuse + S3 冷存储
litellm._async_success_callback = [
    "prometheus",   # 指标监控
    "langfuse",     # 详细追踪
    "s3_v2"         # 日志归档
]

# 配置 S3
os.environ["AWS_ACCESS_KEY_ID"] = "xxx"
os.environ["AWS_SECRET_ACCESS_KEY"] = "xxx"
os.environ["S3_BUCKET_NAME"] = "my-litellm-logs"
```

#### 模式3: 自定义回调

```python
from litellm.integrations.custom_logger import CustomLogger
import litellm

class MyCustomLogger(CustomLogger):
    async def async_log_success_event(self, kwargs, response_obj, start_time, end_time):
        # 访问标准化日志对象
        payload = kwargs.get("standard_logging_object")
        model = payload.get("model")
        cost = payload.get("response_cost")
        tokens = payload.get("total_tokens")
        
        # 发送到内部系统
        await self.send_to_internal_api({
            "model": model,
            "cost": cost,
            "tokens": tokens,
            "timestamp": end_time.isoformat()
        })
    
    async def async_log_failure_event(self, kwargs, response_obj, start_time, end_time):
        # 处理失败情况
        error = kwargs.get("standard_logging_object", {}).get("error_str")
        await self.alert_on_error(error)

# 注册
litellm.callbacks = [MyCustomLogger()]
```

#### 模式4: 仅在 Proxy 中配置

```yaml
# proxy_config.yaml
model_list:
  - model_name: gpt-3.5-turbo
    litellm_params:
      model: openai/gpt-3.5-turbo
      api_key: "os.environ/OPENAI_API_KEY"

general_settings:
  success_callback: ["langfuse", "prometheus", "datadog"]
  callback_settings:
    datadog:
      callback_type: generic_api
      endpoint: "https://http-intake.logs.datadoghq.com/api/v2/logs"
      headers:
        DD-API-KEY: "os.environ/DD_API_KEY"
```

### 4. 性能优化建议

1. **优先使用异步回调**
   - 异步回调通过 `LoggingWorker` 后台执行，不阻塞主路径
   - 同步回调 (`success_callback`) 会阻塞响应返回

2. **控制回调数量**
   - 最大 30 个回调限制是有原因的
   - 太多回调会增加日志处理延迟

3. **流式响应注意**
   - 流式响应会在流结束后才触发完整日志
   - 如果需要实时日志，考虑使用 `log_stream_event` 钩子

4. **监控日志工作器**
   - 关注 `LoggingWorker queue is full` 警告
   - 这表示日志处理速度跟不上生产速度

---

## 附录：回调钩子方法完整列表

所有继承自 `CustomLogger` 的回调可以实现以下钩子方法：

### 1. 生命周期钩子

| 方法名 | 调用时机 | 主要用途 |
|--------|---------|---------|
| `log_pre_api_call` / `async_log_pre_api_call` | API 调用之前 | 记录请求即将发出、准备追踪 |
| `log_success_event` / `async_log_success_event` | 调用成功后 | 记录成功调用、计算成本 |
| `log_failure_event` / `async_log_failure_event` | 调用失败后 | 记录错误、异常追踪 |
| `log_stream_event` / `async_log_stream_event` | 流式每个 chunk | 实时流式日志 |

### 2. 数据修改钩子

| 方法名 | 调用时机 | 主要用途 |
|--------|---------|---------|
| `logging_hook` / `async_logging_hook` | 回调执行前 | 修改请求/响应日志内容 |
| `async_pre_call_deployment_hook` | 部署选择后 | 修改请求参数 |
| `async_post_call_success_hook` | 成功响应后 | 修改响应返回给用户 |

### 3. 代理/路由特有钩子

| 方法名 | 主要用途 |
|--------|---------|
| `async_pre_call_hook` | 代理请求预处理、鉴权 |
| `async_post_call_response_headers_hook` | 注入自定义响应头 |
| `async_post_call_failure_hook` | 转换错误响应 |
| `log_success_fallback_event` | Router 降级成功 |
| `log_failure_fallback_event` | Router 降级失败 |
| `log_model_group_rate_limit_error` | 模型组限流错误 |

### 4. 提示管理钩子

| 方法名 | 主要用途 |
|--------|---------|
| `get_chat_completion_prompt` | 从提示管理系统获取提示 |
| `async_get_chat_completion_prompt` | 异步版本 |

### 5. MCP (Model Context Protocol) 钩子

| 方法名 | 主要用途 |
|--------|---------|
| `async_post_mcp_tool_call_hook` | MCP 工具调用后处理 |

### 6. Agentic Loop 钩子

| 方法名 | 主要用途 |
|--------|---------|
| `async_should_run_agentic_loop` | 判断是否执行 agentic loop |
| `async_run_agentic_loop` | 执行 agentic loop |
| `async_build_agentic_loop_plan` | 构建 agentic loop 计划 |

---

*文档生成时间: 2026-05-02*
*基于 LiteLLM 代码库分析*
