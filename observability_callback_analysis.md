# LiteLLM 可观测性回调与异步日志管线分析

## 目录

1. [概述](#概述)
2. [支持的可观测性平台集成](#支持的可观测性平台集成)
3. [回调注册机制](#回调注册机制)
4. [动态回调的双通道触发机制](#动态回调的双通道触发机制)
5. [异步日志事件处理流程](#异步日志事件处理流程)
6. [架构总结](#架构总结)

---

## 概述

LiteLLM 提供了一套完善的可观测性回调系统，允许用户将 LLM 调用的详细信息发送到各种外部监控和分析平台。该系统采用**异步非阻塞**设计，确保日志操作不会影响主请求的延迟。

核心组件：
- **`LoggingCallbackManager`**: 集中管理回调的注册和路由
- **`LoggingWorker`**: 后台异步任务队列处理器
- **`CustomLogger`**: 自定义回调的基类
- **`Logging`**: 日志事件的核心处理类 (位于 `litellm_logging.py`)

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
_async_success_callback: List[CALLBACK_TYPES] = []  # 调用成功后执行 (异步 - 通过 LoggingWorker)
_async_failure_callback: List[CALLBACK_TYPES] = []  # 调用失败后执行 (异步 - 通过 LoggingWorker)
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
# 常量定义 (constants.py:186)
# Override with LITELLM_MAX_CALLBACKS env var for large deployments (e.g., many teams with guardrails)
MAX_CALLBACKS = get_env_int("LITELLM_MAX_CALLBACKS", 100)

def _check_callback_list_size(self, parent_list) -> bool:
    if len(parent_list) >= MAX_CALLBACKS:
        verbose_logger.warning(
            f"Cannot add callback - would exceed MAX_CALLBACKS limit of {MAX_CALLBACKS}"
        )
        return False
    return True
```

**⚠️ 注意**: 默认回调数量上限是 **100**，可通过环境变量 `LITELLM_MAX_CALLBACKS` 覆盖。

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

### 4. 回调合并与执行顺序

#### ⚠️ 重要修正：回调合并顺序不保证

**之前的错误描述**：回调按动态回调 → 全局异步回调 → 全局同步回调的优先级顺序执行

**实际情况**：回调合并时使用 `set()` 去重，**顺序不被保证**。

```python
# litellm_logging.py:3294-3299
def get_combined_callback_list(
    self, dynamic_success_callbacks: Optional[List], global_callbacks: List
) -> List:
    if dynamic_success_callbacks is None:
        return list(global_callbacks)
    return list(set(dynamic_success_callbacks + global_callbacks))  # ⚠️ 使用 set() 去重！
```

**关键发现**：
1. `set()` 是无序的，因此回调的执行顺序**不固定**
2. `set()` 会自动去重，所以即使同一个回调出现在动态回调和全局回调中，也只会执行一次
3. 如果需要固定执行顺序，不应依赖合并后的顺序

#### 两条独立的执行通道

LiteLLM 有**两条独立的执行通道**，每条通道都有自己的回调合并逻辑：

| 通道 | 触发时机 | 回调源 | 合并方式 |
|------|---------|--------|---------|
| **同步通道** (`success_handler`) | 同步调用或异步调用中同步执行 | `dynamic_success_callbacks` + `litellm.success_callback` | `list(set(...))` 去重 |
| **异步通道** (`async_success_handler`) | 仅异步调用或 Proxy 中通过 LoggingWorker 执行 | `dynamic_async_success_callbacks` + `litellm._async_success_callback` | `list(set(...))` 去重 |

**代码位置**：
- 同步通道：`litellm_logging.py:2073-2076`
- 异步通道：`litellm_logging.py:2640-2643`

---

## 动态回调的双通道触发机制

### 1. 什么是动态回调

动态回调是指**在单次 API 调用时通过参数指定的回调**，只对该次调用生效：

```python
# 动态回调示例
response = litellm.completion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}],
    success_callback=["langfuse"],  # 动态回调 - 仅本次调用
    failure_callback=["posthog"]
)
```

### 2. `_known_custom_logger_compatible_callbacks` 列表

LiteLLM 预定义了一个"已知兼容回调"列表，这些回调字符串会被特殊处理：

```python
# litellm/__init__.py:102-152
_known_custom_logger_compatible_callbacks = [
    "lago", "openmeter", "logfire", "literalai",
    "litellm_agent", "dynamic_rate_limiter", "dynamic_rate_limiter_v3",
    "langsmith", "prometheus", "otel", 
    "datadog", "datadog_metrics", "datadog_llm_observability",
    "galileo", "braintrust", "arize", "arize_phoenix", "langtrace",
    "gcs_bucket", "azure_storage", "opik", "argilla", "mlflow",
    "langfuse", "langfuse_otel", "weave_otel",
    "pagerduty", "humanloop", "azure_sentinel", "gcs_pubsub", "agentops",
    "anthropic_cache_control_hook", "generic_api",
    "resend_email", "sendgrid_email", "smtp_email",
    "deepeval", "s3_v2", "aws_sqs", 
    "vector_store_pre_call_hook", "dotprompt",
    "bitbucket", "gitlab", "cloudzero", "focus", "vantage", "posthog", "levo",
    "compression_interception"
]
```

### 3. 双通道触发机制的核心代码

当动态回调是上述列表中的字符串时，会被**自动添加到同步和异步两条通道**：

```python
# litellm_logging.py:456-497
def _process_dynamic_callback_list(
    self,
    callback_list: Optional[List[Union[str, Callable, CustomLogger]]],
    dynamic_callbacks_type: Literal[
        "input", "success", "failure", "async_success", "async_failure"
    ],
) -> Optional[List[Union[str, Callable, CustomLogger]]]:
    """
    - 如果回调在 litellm._known_custom_logger_compatible_callbacks 中，
      将字符串替换为初始化的回调类实例。
    - 如果动态回调是 "success" 类型且是已知兼容回调，同时添加到 dynamic_async_success_callbacks
    - 如果动态回调是 "failure" 类型且是已知兼容回调，同时添加到 dynamic_failure_callbacks
    """
    if callback_list is None:
        return None

    processed_list: List[Union[str, Callable, CustomLogger]] = []
    for callback in callback_list:
        if (
            isinstance(callback, str)
            and callback in litellm._known_custom_logger_compatible_callbacks
        ):
            callback_class = _init_custom_logger_compatible_class(
                callback, internal_usage_cache=None, llm_router=None
            )
            if callback_class is not None:
                processed_list.append(callback_class)

                # ⚠️ 关键：双通道触发！
                # 如果处理的是 success 类型的动态回调，
                # 同时添加到 dynamic_async_success_callbacks
                if dynamic_callbacks_type == "success":
                    if self.dynamic_async_success_callbacks is None:
                        self.dynamic_async_success_callbacks = []
                    self.dynamic_async_success_callbacks.append(callback_class)
                
                # 同样，failure 类型会添加到 dynamic_async_failure_callbacks
                elif dynamic_callbacks_type == "failure":
                    if self.dynamic_async_failure_callbacks is None:
                        self.dynamic_async_failure_callbacks = []
                    self.dynamic_async_failure_callbacks.append(callback_class)
        else:
            # 非字符串或非已知兼容回调，只添加到当前通道
            processed_list.append(callback)
    return processed_list
```

### 4. 双通道触发流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    用户调用 with 动态回调                                   │
│  litellm.acompletion(model="gpt-3.5", messages=[...],                    │
│                      success_callback=["langfuse"])                        │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Logging 类初始化时处理动态回调                                 │
│  _process_dynamic_callback_list()                                          │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    检查回调类型                                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │  回调是字符串且在 _known_custom_logger_compatible_callbacks 中？       │  │
│  └─────────────────────────────┬───────────────────────────────────────┘  │
│                                │                                           │
│          ┌─────────────────────┴─────────────────────┐                   │
│          │                                             │                   │
│          ▼                                             ▼                   │
│  ┌─────────────────┐                    ┌───────────────────────────────┐ │
│  │ 是              │                    │ 否                            │ │
│  │                 │                    │                               │ │
│  │ 执行：          │                    │ 执行：                         │ │
│  │ 1. 初始化为类实例│                    │ 1. 保持原样（函数/CustomLogger）│ │
│  │ 2. 添加到       │                    │ 2. 只添加到当前通道            │ │
│  │    dynamic_     │                    │                               │ │
│  │    success_     │                    │                               │ │
│  │    callbacks    │                    │                               │ │
│  │ 3. 同时添加到   │                    │                               │ │
│  │    dynamic_     │                    │                               │ │
│  │    async_       │                    │                               │ │
│  │    success_     │                    │                               │ │
│  │    callbacks    │                    │                               │ │
│  └─────────────────┘                    └───────────────────────────────┘ │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         调用完成后的回调执行                                │
│                                                                             │
│  ┌─────────────────────────────────┐    ┌───────────────────────────────┐ │
│  │ 同步通道 (success_handler)      │    │ 异步通道 (async_success_      │ │
│  │                                 │    │ handler)                      │ │
│  │ 回调列表：                      │    │                               │ │
│  │ get_combined_callback_list(    │    │ 回调列表：                    │ │
│  │   dynamic_success_callbacks,   │    │ get_combined_callback_list(  │ │
│  │   litellm.success_callback     │    │   dynamic_async_success_     │ │
│  │ )                               │    │   callbacks,                  │ │
│  │                                 │    │   litellm._async_success_    │ │
│  │ ⚠️ 包含动态注册的 langfuse      │    │   callback                    │ │
│  │                                 │    │ )                             │ │
│  │ 执行方式：                      │    │                               │ │
│  │ 主线程同步执行                  │    │ ⚠️ 也包含动态注册的 langfuse│ │
│  │                                 │    │                               │ │
│  │ 触发时机：                      │    │ 执行方式：                    │ │
│  │ - 同步 completion() 调用       │    │ 通过 LoggingWorker 异步执行  │ │
│  │ - 异步 acompletion() 也会调用  │    │                               │ │
│  │   (通过 handle_sync_success_   │    │ 触发时机：                    │ │
│  │    callbacks_for_async_calls)  │    │ - 仅异步 acompletion() 调用  │ │
│  │                                 │    │ - 或 Proxy 服务器请求        │ │
│  └─────────────────────────────────┘    └───────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5. 实际行为示例

#### 示例1: 已知兼容回调作为动态回调

```python
import litellm

litellm.success_callback = ["prometheus"]  # 全局同步回调
litellm._async_success_callback = ["datadog"]  # 全局异步回调

# 单次调用时指定动态回调
response = await litellm.acompletion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}],
    success_callback=["langfuse"]  # 动态回调，langfuse 是已知兼容回调
)
```

**实际执行的回调**：

| 通道 | 合并后的回调列表 | 执行方式 |
|------|-----------------|---------|
| **同步通道** | `{"langfuse", "prometheus"}` 去重 → 顺序不固定 | 主线程同步执行 |
| **异步通道** | `{"langfuse", "datadog"}` 去重 → 顺序不固定 | LoggingWorker 异步执行 |

**关键发现**：`langfuse` 会在**两条通道都被执行**！

#### 示例2: 自定义回调作为动态回调

```python
from litellm.integrations.custom_logger import CustomLogger

class MyCustomLogger(CustomLogger):
    def log_success_event(self, kwargs, response_obj, start_time, end_time):
        print(f"[SYNC] Custom logger called")
    
    async def async_log_success_event(self, kwargs, response_obj, start_time, end_time):
        print(f"[ASYNC] Custom logger called")

# 动态回调使用自定义 Logger 实例
my_logger = MyCustomLogger()
response = await litellm.acompletion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}],
    success_callback=[my_logger]  # 动态回调，不是字符串
)
```

**实际执行的回调**：

| 通道 | 合并后的回调列表 | 执行的方法 |
|------|-----------------|-----------|
| **同步通道** | `{my_logger, ...}` | `my_logger.log_success_event()` |
| **异步通道** | 不包含 my_logger | ❌ 不会执行 `async_log_success_event()` |

**关键发现**：自定义 `CustomLogger` 实例**不会自动添加到异步通道**！

### 6. 不同调用方式下的行为差异

| 调用方式 | 同步通道 | 异步通道 |
|---------|---------|---------|
| `completion()` (同步) | ✅ 执行动态 + 全局同步回调 | ❌ 不执行（除特殊处理的 `openmeter`） |
| `acompletion()` (异步) | ✅ 执行（通过 `handle_sync_success_callbacks_for_async_calls`） | ✅ 执行（通过 LoggingWorker） |
| Proxy 服务器请求 | ✅ 执行 | ✅ 执行 |

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
│                     Logging 类 (litellm_logging.py:290)                 │
│  - 构建 StandardLoggingPayload                                             │
│  - 计算 response_cost                                                      │
│  - 处理动态回调的双通道注册 (_process_dynamic_callback_list)              │
│  - 准备回调执行                                                             │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ├──────────────────────────────────────────────┐
                              │                                              │
                              ▼                                              ▼
┌─────────────────────────────────────────────┐  ┌─────────────────────────────────────────────┐
│           同步通道 (Sync Channel)            │  │           异步通道 (Async Channel)          │
│                                              │  │                                              │
│  success_handler()                           │  │  async_success_handler()                     │
│  (litellm_logging.py:2007)                  │  │  (litellm_logging.py:2478)                  │
│                                              │  │                                              │
│  回调列表：                                   │  │  回调列表：                                   │
│  - dynamic_success_callbacks                 │  │  - dynamic_async_success_callbacks           │
│    (包含：动态注册的已知兼容回调)            │  │    (包含：动态注册的已知兼容回调 +          │
│  - litellm.success_callback                  │  │     动态注册的 success 类型兼容回调)        │
│    (全局同步回调)                            │  │  - litellm._async_success_callback          │
│                                              │  │    (全局异步回调)                            │
│  合并方式：list(set(...)) 去重               │  │                                              │
│  ⚠️ 顺序不固定                               │  │  合并方式：list(set(...)) 去重              │
│                                              │  │  ⚠️ 顺序不固定                              │
│  执行方式：主线程同步执行                    │  │                                              │
│                                              │  │  执行方式：通过 LoggingWorker 异步执行       │
│  触发时机：                                  │  │                                              │
│  - 同步 completion() 调用                   │  │  触发时机：                                  │
│  - 异步 acompletion() 也会触发             │  │  - 仅异步 acompletion() 调用                │
│    (handle_sync_success_callbacks_for_      │  │  - Proxy 服务器请求                          │
│     async_calls)                            │  │                                              │
└─────────────────────────────────────────────┘  └─────────────────────────────┬───────────────┘
                                                                               │
                                                                               ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────┐
│                         LoggingWorker (logging_worker.py:32)                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │  asyncio.Queue (LoggingTask)                                                             │  │
│  │  ┌─────────┬─────────┬─────────┬─────────┬─────────┐                                  │  │
│  │  │ Task 1  │ Task 2  │ Task 3  │  ...    │ Task N  │                                  │  │
│  │  └─────────┴─────────┴─────────┴─────────┴─────────┘                                  │  │
│  │                                                                                           │  │
│  │  配置参数 (默认值)：                                                                       │  │
│  │  - 并发数: 100 (LOGGING_WORKER_CONCURRENCY)                                            │  │
│  │  - 队列大小: 50,000 (LOGGING_WORKER_MAX_QUEUE_SIZE)                                   │  │
│  │  - 单任务超时: 20秒 (LOGGING_WORKER_MAX_TIME_PER_COROUTINE)                            │  │
│  │  - 清理百分比: 50% (LOGGING_WORKER_CLEAR_PERCENTAGE)                                   │  │
│  │  - 清理冷却: 0.5秒 (LOGGING_WORKER_AGGRESSIVE_CLEAR_COOLDOWN_SECONDS)                 │  │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2. LoggingWorker 核心实现

#### 2.1 核心配置参数

```python
# 常量定义 (constants.py:476-489)
LOGGING_WORKER_CONCURRENCY = int(
    os.getenv("LOGGING_WORKER_CONCURRENCY", 100)           # 默认并发数: 100
)
LOGGING_WORKER_MAX_QUEUE_SIZE = int(
    os.getenv("LOGGING_WORKER_MAX_QUEUE_SIZE", 50_000)     # 默认队列大小: 50,000
)
LOGGING_WORKER_MAX_TIME_PER_COROUTINE = float(
    os.getenv("LOGGING_WORKER_MAX_TIME_PER_COROUTINE", 20.0)  # 默认超时: 20秒
)
LOGGING_WORKER_CLEAR_PERCENTAGE = int(
    os.getenv("LOGGING_WORKER_CLEAR_PERCENTAGE", 50)        # 默认清理百分比: 50%
)
LOGGING_WORKER_AGGRESSIVE_CLEAR_COOLDOWN_SECONDS = float(
    os.getenv("LOGGING_WORKER_AGGRESSIVE_CLEAR_COOLDOWN_SECONDS", 0.5)  # 默认冷却: 0.5秒
)
MAX_ITERATIONS_TO_CLEAR_QUEUE = int(
    os.getenv("MAX_ITERATIONS_TO_CLEAR_QUEUE", 200)         # 默认最大清理迭代: 200
)
MAX_TIME_TO_CLEAR_QUEUE = float(
    os.getenv("MAX_TIME_TO_CLEAR_QUEUE", 5.0)                # 默认最大清理时间: 5秒
)
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
        # 1. 获取信号量（控制并发，默认 100）
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
                    timeout=self.timeout,  # 默认 20 秒超时
                )
            except Exception as e:
                verbose_logger.exception(f"LoggingWorker error: {e}")
            finally:
                self._queue.task_done()
    finally:
        sem.release()
```

### 3. 完整数据流

#### 3.1 异步调用路径 (acompletion) - 推荐

```
1. 用户调用 litellm.acompletion(
       model="gpt-3.5-turbo",
       messages=[...],
       success_callback=["langfuse"]  # 动态回调
   )
   │
   ▼
2. Logging 类初始化
   │
   ├──► _process_dynamic_callback_list("langfuse", "success")
   │       │
   │       ├──► "langfuse" 是 known_custom_logger_compatible_callbacks
   │       ├──► 初始化为 LangFuseLogger 实例
   │       ├──► 添加到 dynamic_success_callbacks (同步通道)
   │       └──► 同时添加到 dynamic_async_success_callbacks (异步通道)
   │
   ▼
3. 实际 LLM API 调用完成
   │
   ├──► 同步通道触发 (handle_sync_success_callbacks_for_async_calls)
   │       │
   │       ├──► get_combined_callback_list(
   │       │       dynamic_success_callbacks=["langfuse"],
   │       │       global_callbacks=litellm.success_callback
   │       │   )
   │       │
   │       └──► 主线程同步执行每个 callback 的 log_success_event()
   │
   └──► 异步通道触发 (_client_async_logging_helper)
           │
           ├──► GLOBAL_LOGGING_WORKER.ensure_initialized_and_enqueue(
           │       async_coroutine=logging_obj.async_success_handler(...)
           │   )
           │
           └──► LoggingWorker 后台异步执行
                   │
                   └──► get_combined_callback_list(
                           dynamic_async_success_callbacks=["langfuse"],
                           global_callbacks=litellm._async_success_callback
                       )
                   │
                   └──► 异步执行每个 callback 的 async_log_success_event()
```

#### 3.2 同步调用路径 (completion) - 关键行为

**⚠️ 重要发现：同步调用时异步回调的实际触发条件**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        同步调用 (completion)                               │
│                                                                             │
│  调用 success_handler() (litellm_logging.py:2007)                         │
│                                                                             │
└─────────────────────────────┬─────────────────────────────────────────────┘
                              │
                              ├──────────────────────────────────────────────┐
                              │                                              │
                              ▼                                              ▼
┌─────────────────────────────────────────────┐  ┌─────────────────────────────────────────────┐
│           同步通道执行                        │  │           异步通道状态                        │
│                                              │  │                                              │
│  回调列表：                                   │  │  ⚠️ 不会触发！                                │
│  - dynamic_success_callbacks                 │  │                                              │
│  - litellm.success_callback                  │  │  原因：                                       │
│                                              │  │  1. _client_async_logging_helper 是 async def │
│  合并方式：list(set(...)) 去重               │  │  2. 同步 completion() 不会调用它              │
│  ⚠️ 顺序不固定                               │  │  3. async_success_handler 不会被调用          │
│                                              │  │                                              │
│  执行方式：主线程同步执行                    │  │  例外：                                       │
│                                              │  │  - "openmeter" 在 success_handler 中有特殊处理 │
│  触发条件：总是执行                          │  │    (line 2383)                                │
│                                              │  │  - 但它是同步执行，不是通过 LoggingWorker     │
└─────────────────────────────────────────────┘  └─────────────────────────────────────────────┘
```

**关键结论：**

| 调用类型 | 同步通道 | 异步通道 |
|---------|---------|---------|
| **acompletion** (异步) | ✅ 执行 (dynamic_success_callbacks + success_callback) | ✅ 执行 (dynamic_async_success_callbacks + _async_success_callback) |
| **completion** (同步) | ✅ 执行 (dynamic_success_callbacks + success_callback) | ❌ 不执行 (除 "openmeter" 特殊处理) |

**特殊回调 "openmeter" 的行为：**
- 在 `LoggingCallbackManager.add_litellm_success_callback()` 中，`openmeter` 和 `dynamodb` 会被自动添加到 `_async_success_callback`
- 但在同步调用的 `success_handler()` (line 2383) 中，**只有 `openmeter`** 被特殊处理，会同步执行
- `dynamodb` 在同步调用中 **不会** 被执行

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
        while not self._queue.empty() and processed < MAX_ITERATIONS_TO_CLEAR_QUEUE:  # 默认 200
            try:
                task = self._queue.get_nowait()
                # 同步执行剩余任务，有 MAX_TIME_TO_CLEAR_QUEUE (默认 5秒) 限制
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
- `asyncio.Semaphore` 控制并发数（默认 100）
- 队列有界（默认 50,000）防止内存无限增长
- 队列满时有策略：清理 50% 旧任务 + 延迟重试（冷却期 0.5 秒）

**1.4 标准化与扩展性**
- `StandardLoggingPayload` 统一所有平台的日志格式
- `CustomLogger` 基类提供丰富的钩子方法
- 支持字符串、函数、类实例三种回调形式

**1.5 动态回调的双通道设计**
- 已知兼容回调作为动态回调时，会自动添加到同步和异步两条通道
- 确保单次调用指定的回调能被完整执行

### 2. 关键代码位置参考

| 组件 | 文件路径 | 主要职责 |
|------|---------|---------|
| Logging (核心日志类) | `litellm/litellm_core_utils/litellm_logging.py:290` | 定义 `class Logging`，包含 `success_handler` (line 2007) 和 `async_success_handler` (line 2478) |
| get_combined_callback_list | `litellm/litellm_core_utils/litellm_logging.py:3294` | 合并动态回调和全局回调，使用 `set()` 去重 |
| _process_dynamic_callback_list | `litellm/litellm_core_utils/litellm_logging.py:456` | 处理动态回调，实现双通道触发机制 |
| LoggingCallbackManager | `litellm/litellm_core_utils/logging_callback_manager.py` | 回调注册、路由、去重 |
| LoggingWorker | `litellm/litellm_core_utils/logging_worker.py:32` | 异步任务队列执行；全局单例 `GLOBAL_LOGGING_WORKER` 定义在 line 533 |
| CustomLogger (基类) | `litellm/integrations/custom_logger.py` | 自定义回调基类，定义所有钩子 |
| _known_custom_logger_compatible_callbacks | `litellm/__init__.py:102-157` | 已知兼容回调列表，这些回调会被双通道触发 |
| _client_async_logging_helper | `litellm/utils.py:1176` | 异步调用的日志包装，将异步回调入队到 LoggingWorker |
| 常量定义 | `litellm/constants.py` | MAX_CALLBACKS, LOGGING_WORKER_* 等常量 |

### 3. 常见使用模式

#### 模式1: 快速集成单个平台

```python
import litellm
import os

os.environ["LANGFUSE_PUBLIC_KEY"] = "pk-xxx"
os.environ["LANGFUSE_SECRET_KEY"] = "sk-xxx"

# 推荐：使用 acompletion 以确保所有回调正确执行
litellm.success_callback = ["langfuse"]

# 后续所有调用都会自动记录到 LangFuse
response = litellm.completion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}]
)
```

#### 模式2: 动态回调（单次调用）

```python
import litellm

# 全局配置
litellm.success_callback = ["prometheus"]  # 所有调用都记录到 Prometheus

# 特定调用额外记录到 LangFuse
# ⚠️ langfuse 是 known_custom_logger_compatible_callbacks
# 如果使用 acompletion，会在同步和异步两条通道都执行
response = await litellm.acompletion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "重要请求"}],
    success_callback=["langfuse"]  # 仅本次调用
)
```

#### 模式3: 自定义回调

```python
from litellm.integrations.custom_logger import CustomLogger
import litellm

class MyCustomLogger(CustomLogger):
    def log_success_event(self, kwargs, response_obj, start_time, end_time):
        # 同步通道会调用这个方法
        payload = kwargs.get("standard_logging_object")
        print(f"[SYNC] Model: {payload.get('model')}")
    
    async def async_log_success_event(self, kwargs, response_obj, start_time, end_time):
        # 异步通道会调用这个方法
        # ⚠️ 注意：如果作为动态回调，这个方法不会自动被调用！
        # 因为自定义实例不是 known_custom_logger_compatible_callbacks
        payload = kwargs.get("standard_logging_object")
        print(f"[ASYNC] Cost: {payload.get('response_cost')}")

# 注册为全局回调
my_logger = MyCustomLogger()
litellm.success_callback = [my_logger]  # 同步通道
litellm._async_success_callback = [my_logger]  # 异步通道

# 或者作为动态回调（只触发同步通道的 log_success_event）
response = litellm.completion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hello"}],
    success_callback=[my_logger]  # 只触发 log_success_event()
)
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

### 4. 性能优化与最佳实践

#### 4.1 回调配置建议

| 场景 | 推荐配置 | 原因 |
|------|---------|------|
| 使用 `completion()` 同步调用 | 注册到 `success_callback` | 异步通道不会触发 |
| 使用 `acompletion()` 异步调用 | 可以注册到任一通道 | 两条通道都会触发 |
| 动态回调（单次调用） | 使用已知兼容回调字符串 | 会自动双通道触发 |
| 自定义 CustomLogger | 显式注册到两条通道 | 不会自动双通道触发 |

#### 4.2 注意事项

1. **回调执行顺序不固定**
   - `get_combined_callback_list` 使用 `set()` 去重
   - 不要依赖回调的执行顺序

2. **动态回调的双通道触发**
   - 只有 `known_custom_logger_compatible_callbacks` 中的字符串回调会自动双通道触发
   - 自定义 `CustomLogger` 实例或普通函数不会自动添加到异步通道

3. **同步 vs 异步调用**
   - `completion()` 同步调用不会触发异步通道（除 `openmeter` 特殊处理）
   - `acompletion()` 异步调用会触发两条通道

4. **去重机制**
   - 如果同一个回调同时出现在动态回调和全局回调中，`set()` 会自动去重
   - 只会执行一次

#### 4.3 常见陷阱

**陷阱1: 自定义回调作为动态回调时异步方法不执行**

```python
# ❌ 问题：自定义回调的 async_log_success_event 不会被调用
class MyLogger(CustomLogger):
    async def async_log_success_event(self, ...):
        # 这个方法不会被调用！
        pass

response = await litellm.acompletion(
    model="gpt-3.5",
    messages=[...],
    success_callback=[MyLogger()]  # 动态回调
)

# ✅ 解决方案：显式注册到异步通道，或使用同步方法
class MyLogger(CustomLogger):
    def log_success_event(self, ...):
        # 这个方法会被调用
        pass
```

**陷阱2: 依赖回调执行顺序**

```python
# ❌ 问题：假设回调按注册顺序执行
litellm.success_callback = ["callback_a", "callback_b"]

# ✅ 实际：顺序不固定，使用 set() 去重
# 如果需要固定顺序，不要依赖合并机制
```

**陷阱3: 同步调用时代码期望异步回调执行**

```python
# ❌ 问题：同步调用时异步通道不会触发
litellm._async_success_callback = ["langfuse"]

# 同步调用 - langfuse 不会被执行！
response = litellm.completion(model="gpt-3.5", messages=[...])

# ✅ 解决方案：
# 方案1: 使用 acompletion
# 方案2: 注册到 success_callback
litellm.success_callback = ["langfuse"]
```

### 5. 错误修正汇总

| 修正项 | 之前错误描述 | 正确值 |
|--------|-------------|--------|
| MAX_CALLBACKS | 30 | **100** (可通过 `LITELLM_MAX_CALLBACKS` 覆盖) |
| LOGGING_WORKER_CONCURRENCY | 10 | **100** |
| LOGGING_WORKER_MAX_QUEUE_SIZE | 1000 | **50,000** |
| LOGGING_WORKER_MAX_TIME_PER_COROUTINE | 60 秒 | **20 秒** |
| LOGGING_WORKER_AGGRESSIVE_CLEAR_COOLDOWN_SECONDS | 1 秒 | **0.5 秒** |
| 核心日志文件路径 | `logging.py` | **`litellm_logging.py`** |
| 同步调用时异步回调 | 暗示会触发 | **不会触发** (除 `openmeter` 特殊处理) |
| 回调执行顺序 | 固定优先级 | **顺序不保证** (使用 `set()` 去重) |
| 动态回调 | 只在单通道触发 | **已知兼容回调会双通道触发** |

---

## 附录：回调钩子方法完整列表

所有继承自 `CustomLogger` 的回调可以实现以下钩子方法：

### 1. 生命周期钩子

| 方法名 | 调用时机 | 主要用途 | 执行通道 |
|--------|---------|---------|---------|
| `log_pre_api_call` / `async_log_pre_api_call` | API 调用之前 | 记录请求即将发出、准备追踪 | 同步 / 异步 |
| `log_success_event` / `async_log_success_event` | 调用成功后 | 记录成功调用、计算成本 | 同步 / 异步 |
| `log_failure_event` / `async_log_failure_event` | 调用失败后 | 记录错误、异常追踪 | 同步 / 异步 |
| `log_stream_event` / `async_log_stream_event` | 流式每个 chunk | 实时流式日志 | 同步 / 异步 |

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
*基于 LiteLLM 代码库分析，修正版本 v2*
