# LiteLLM 费用追踪链路深度分析

## 概述

本文档是对 LiteLLM 费用追踪链路的深度分析，聚焦于三个核心问题：

1. **Usage 转换机制**: 不同响应结构间如何统一转换并参与计费
2. **Response Cost 优先级**: 缓存命中、上游已给费用、本地重算之间的优先级与分支逻辑
3. **回调管理器**: 真实接口定义和挂载方式

---

## 一、Usage 对象的统一转换机制

### 1.1 问题背景

LiteLLM 支持 100+ 种 LLM 提供商，不同提供商返回的 `usage` 对象结构各不相同：

| 响应类型 | 字段差异 | 示例 |
|---------|---------|------|
| **标准 OpenAI** | `prompt_tokens`, `completion_tokens`, `total_tokens` | 大多数提供商 |
| **OpenAI Responses API** | `input_tokens`, `output_tokens`, `input_tokens_details` | `/v1/responses` 端点 |
| **Transcription (时长)** | `duration_seconds`, `audio_duration` | 语音转录按时长计费 |
| **Transcription (Token)** | `input_tokens`, `output_tokens`, `input_token_details` | 部分转录按 token 计费 |
| **Anthropic Prompt Caching** | `cache_creation_input_tokens`, `cache_read_input_tokens` | Anthropic 缓存计费 |

LiteLLM 需要将这些异构结构统一转换为标准格式，才能参与费用计算。

### 1.2 核心转换流程

**位置**: `litellm/cost_calculator.py:1140-1235`

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Usage 对象统一转换流程                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 提取原始 usage 对象                                                  │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ 从 completion_response 提取:                                  │    │
│     │ - 如果是 dict: completion_response.get("usage", {})          │    │
│     │ - 如果是 BaseModel: getattr(completion_response, "usage", {})│    │
│     └─────────────────────────────────────────────────────────────┘    │
│                              ↓                                           │
│  2. 标准化为 dict 格式                                                   │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ if isinstance(usage_obj, BaseModel):                         │    │
│     │     # 检查是否是已知类型                                       │    │
│     │     if not _is_known_usage_objects(usage_obj):               │    │
│     │         # 转换为标准 litellm.Usage 对象                       │    │
│     │         litellm.Usage(**_usage_for_dump.model_dump())        │    │
│     │                                                                 │    │
│     │     _usage = cast(BaseModel, usage_obj).model_dump()          │    │
│     │ else:                                                          │    │
│     │     _usage = usage_obj                                         │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                              ↓                                           │
│  3. 类型识别与转换                                                       │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ 分支1: Response API Usage                                     │    │
│     │ ┌─────────────────────────────────────────────────────────┐ │    │
│     │ │ if ResponseAPILoggingUtils._is_response_api_usage(_usage)│ │    │
│     │ │     → _transform_response_api_usage_to_chat_usage()     │ │    │
│     │ └─────────────────────────────────────────────────────────┘ │    │
│     │                              ↓                                │    │
│     │ 分支2: Transcription Usage                                   │    │
│     │ ┌─────────────────────────────────────────────────────────┐ │    │
│     │ │ elif TranscriptionUsageObjectTransformation.            │ │    │
│     │ │      is_transcription_usage_object(_usage)              │ │    │
│     │ │     → transform_transcription_usage_object()             │ │    │
│     │ └─────────────────────────────────────────────────────────┘ │    │
│     │                              ↓                                │    │
│     │ 分支3: 标准 Usage (无需转换)                                  │    │
│     │ ┌─────────────────────────────────────────────────────────┐ │    │
│     │ │ else:                                                     │ │    │
│     │ │     _usage = _usage  # 直接使用                          │ │    │
│     │ └─────────────────────────────────────────────────────────┘ │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                              ↓                                           │
│  4. 提取计费所需字段                                                      │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ prompt_tokens = _usage.get("prompt_tokens", 0)              │    │
│     │ completion_tokens = _usage.get("completion_tokens", 0)      │    │
│     │ cache_creation_input_tokens = _usage.get(..., 0)            │    │
│     │ cache_read_input_tokens = _usage.get(..., 0)                │    │
│     │ # 从 prompt_tokens_details 提取缓存 token                    │    │
│     │ if "prompt_tokens_details" in _usage:                        │    │
│     │     cache_read_input_tokens = prompt_tokens_details.get(     │    │
│     │         "cached_tokens", 0                                    │    │
│     │     )                                                         │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 Response API Usage 转换器

**位置**: `litellm/responses/utils.py:900-975`

OpenAI 的 `/v1/responses` 端点使用不同的字段命名（`input_tokens` 而非 `prompt_tokens`），需要转换。

#### 识别逻辑

```python
@staticmethod
def _is_response_api_usage(usage: Union[dict, ResponseAPIUsage]) -> bool:
    """returns True if usage is from OpenAI Response API"""
    if isinstance(usage, ResponseAPIUsage):
        return True
    if "input_tokens" in usage and "output_tokens" in usage:
        return True
    return False
```

**识别条件**:
1. 是 `ResponseAPIUsage` 类型实例，**或**
2. dict 中同时包含 `input_tokens` 和 `output_tokens` 字段

#### 转换逻辑

```python
@staticmethod
def _transform_response_api_usage_to_chat_usage(
    usage_input: Optional[Union[dict, ResponseAPIUsage]],
) -> Usage:
    """
    Transforms ResponseAPIUsage or ImageUsage to a Usage object.
    
    字段映射:
    - input_tokens → prompt_tokens
    - output_tokens → completion_tokens
    - input_tokens_details → prompt_tokens_details
    - output_tokens_details → completion_tokens_details
    """
    
    # 1. 标准化为 ResponseAPIUsage 对象
    if isinstance(usage_input, dict):
        # 计算 total_tokens（如果缺失）
        if total_tokens is None:
            input_tokens = usage_input.get("input_tokens")
            output_tokens = usage_input.get("output_tokens")
            if input_tokens is not None and output_tokens is not None:
                total_tokens = input_tokens + output_tokens
                usage_input["total_tokens"] = total_tokens
        response_api_usage = ResponseAPIUsage(**usage_input)
    else:
        response_api_usage = usage_input
    
    # 2. 字段映射
    prompt_tokens: int = response_api_usage.input_tokens or 0
    completion_tokens: int = response_api_usage.output_tokens or 0
    
    # 3. 处理 details（包含缓存、音频、图片等细分）
    prompt_tokens_details: Optional[PromptTokensDetailsWrapper] = None
    if response_api_usage.input_tokens_details:
        if isinstance(response_api_usage.input_tokens_details, dict):
            prompt_tokens_details = PromptTokensDetailsWrapper(
                **response_api_usage.input_tokens_details
            )
        else:
            prompt_tokens_details = PromptTokensDetailsWrapper(
                cached_tokens=getattr(...),
                audio_tokens=getattr(...),
                text_tokens=getattr(...),
                image_tokens=getattr(...),
            )
    
    # 4. 处理输出 details（包含推理 token）
    completion_tokens_details: Optional[CompletionTokensDetailsWrapper] = None
    output_tokens_details = getattr(
        response_api_usage, "output_tokens_details", None
    )
    if output_tokens_details:
        completion_tokens_details = CompletionTokensDetailsWrapper(
            reasoning_tokens=getattr(output_tokens_details, "reasoning_tokens", None),
            image_tokens=getattr(output_tokens_details, "image_tokens", None),
            text_tokens=getattr(output_tokens_details, "text_tokens", None),
        )
    
    # 5. 构建标准 Usage 对象
    chat_usage = Usage(
        prompt_tokens=prompt_tokens,
        completion_tokens=completion_tokens,
        total_tokens=prompt_tokens + completion_tokens,
        prompt_tokens_details=prompt_tokens_details,
        completion_tokens_details=completion_tokens_details,
    )
    
    # 6. 保留上游已计算的费用（如果有）
    if hasattr(response_api_usage, "cost") and response_api_usage.cost is not None:
        setattr(chat_usage, "cost", response_api_usage.cost)
    
    return chat_usage
```

**关键点**:
- 字段映射是双向兼容的
- `input_tokens_details` 可能包含 `cached_tokens`（缓存）、`audio_tokens`（音频）、`image_tokens`（图片）
- `output_tokens_details` 可能包含 `reasoning_tokens`（推理 token）
- 如果上游已经计算了 `cost`，会保留这个值

### 1.4 Transcription Usage 转换器

**位置**: `litellm/litellm_core_utils/llm_cost_calc/usage_object_transformation.py`

语音转录有两种计费模式，需要统一处理。

#### 识别逻辑

```python
@staticmethod
def is_transcription_usage_object(
    usage_object: Any,
) -> bool:
    return isinstance(usage_object, TranscriptionUsageDurationObject) or isinstance(
        usage_object, TranscriptionUsageTokensObject
    )
```

#### 转换逻辑

```python
@staticmethod
def transform_transcription_usage_object(
    usage_object: Union[
        TranscriptionUsageDurationObject, TranscriptionUsageTokensObject
    ],
) -> Optional[Usage]:
    # 情况1: 按时长计费 → 返回 None（由其他逻辑处理）
    if isinstance(usage_object, TranscriptionUsageDurationObject):
        return None
    
    # 情况2: 按 token 计费 → 转换为标准 Usage
    elif isinstance(usage_object, TranscriptionUsageTokensObject):
        return Usage(
            prompt_tokens=usage_object.input_tokens,
            completion_tokens=usage_object.output_tokens,
            total_tokens=usage_object.total_tokens,
            prompt_tokens_details=PromptTokensDetailsWrapper(
                text_tokens=usage_object.input_token_details.text_tokens,
                audio_tokens=usage_object.input_token_details.audio_tokens,
            ),
        )
    return None
```

**设计意图**:
- `TranscriptionUsageDurationObject` 按时长计费，不需要转换为 token（费用计算单独处理）
- `TranscriptionUsageTokensObject` 按 token 计费，需要转换为标准格式参与费用计算

### 1.5 未知 Usage 对象的标准化

**位置**: `litellm/cost_calculator.py:1148-1156`

```python
if isinstance(usage_obj, BaseModel) and not _is_known_usage_objects(
    usage_obj=usage_obj
):
    _usage_for_dump = cast(BaseModel, usage_obj)
    setattr(
        completion_response,
        "usage",
        litellm.Usage(**_usage_for_dump.model_dump()),
    )
```

**目的**: 对于未知的 BaseModel 类型 usage 对象，将其转换为标准的 `litellm.Usage` 对象，确保后续处理的一致性。

---

## 二、Response Cost 优先级与分支逻辑

### 2.1 问题背景

LiteLLM 的 `response_cost` 可能来自多个来源，需要明确的优先级规则：

| 来源 | 场景 | 优先级 |
|-----|------|-------|
| **缓存命中** | 请求命中 LiteLLM 缓存，无需调用上游 | 最高（0 费用） |
| **上游已给费用** | 上游响应中已包含 `_hidden_params["response_cost"]` 或 `usage.cost` | 次高（直接使用） |
| **本地重算** | 以上都不满足时，根据 usage 和 model_info 计算 | 最低 |

### 2.2 核心决策流程

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1451-1586`

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  Response Cost 优先级决策流程                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  入口: _response_cost_calculator() 方法                                  │
│                                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │ 优先级 1: 检查缓存命中                                            │    │
│  │ ┌─────────────────────────────────────────────────────────────┐ │    │
│  │ │                                                                 │ │    │
│  │ │ if cache_hit is None:                                          │ │    │
│  │ │     cache_hit = self.model_call_details.get("cache_hit", False)│ │    │
│  │ │                                                                 │ │    │
│  │ │ if cache_hit is True:                                          │ │    │
│  │ │     return 0.0  # 缓存命中，费用为 0                           │ │    │
│  │ │                                                                 │ │    │
│  │ └─────────────────────────────────────────────────────────────┘ │    │
│  │                              ↓                                    │    │
│  │ 优先级 2: 检查上游已计算的费用                                     │    │
│  │ ┌─────────────────────────────────────────────────────────────┐ │    │
│  │ │                                                                 │ │    │
│  │ │ if isinstance(result, BaseModel) and hasattr(result,        │ │    │
│  │ │    "_hidden_params"):                                          │ │    │
│  │ │     hidden_params = getattr(result, "_hidden_params", {})      │ │    │
│  │ │                                                                 │ │    │
│  │ │     # 检查是否已有预计算的费用                                   │ │    │
│  │ │     if ("response_cost" in hidden_params and                  │ │    │
│  │ │         hidden_params["response_cost"] is not None):           │ │    │
│  │ │         return hidden_params["response_cost"]  # 直接使用      │ │    │
│  │ │                                                                 │ │    │
│  │ │     # 同时提取 model_id 供后续使用                              │ │    │
│  │ │     elif (router_model_id is None and "model_id" in          │ │    │
│  │ │           hidden_params):                                      │ │    │
│  │ │         router_model_id = hidden_params["model_id"]            │ │    │
│  │ │                                                                 │ │    │
│  │ └─────────────────────────────────────────────────────────────┘ │    │
│  │                              ↓                                    │    │
│  │ 优先级 3: 本地重算费用                                            │    │
│  │ ┌─────────────────────────────────────────────────────────────┐ │    │
│  │ │                                                                 │ │    │
│  │ │ # 准备计算参数                                                 │ │    │
│  │ │ custom_pricing = use_custom_pricing_for_model(...)            │ │    │
│  │ │ prompt = ...  # 用于 TTS 按字符计费                           │ │    │
│  │ │                                                                 │ │    │
│  │ │ # 构建 kwargs                                                   │ │    │
│  │ │ response_cost_calculator_kwargs = {                            │ │    │
│  │ │     "response_object": result,                                 │ │    │
│  │ │     "model": litellm_model_name or self.model,                 │ │    │
│  │ │     "cache_hit": cache_hit,                                    │ │    │
│  │ │     "custom_llm_provider": ...,                                │ │    │
│  │ │     "base_model": ...,                                         │ │    │
│  │ │     "call_type": self.call_type,                               │ │    │
│  │ │     "optional_params": self.optional_params,                   │ │    │
│  │ │     "custom_pricing": custom_pricing,                          │ │    │
│  │ │     "prompt": prompt,                                           │ │    │
│  │ │     "standard_built_in_tools_params": ...,                     │ │    │
│  │ │     "router_model_id": router_model_id,                        │ │    │
│  │ │     "litellm_logging_obj": self,                               │ │    │
│  │ │     "service_tier": ...,                                       │ │    │
│  │ │ }                                                               │ │    │
│  │ │                                                                 │ │    │
│  │ │ # 调用主计算器                                                  │ │    │
│  │ │ try:                                                            │ │    │
│  │ │     response_cost = litellm.response_cost_calculator(          │ │    │
│  │ │         **response_cost_calculator_kwargs                      │ │    │
│  │ │     )                                                           │ │    │
│  │ │     return response_cost                                        │ │    │
│  │ │ except Exception as e:                                          │ │    │
│  │ │     # 记录调试信息以便排查                                       │ │    │
│  │ │     debug_info = StandardLoggingModelCostFailureDebugInformation(│ │    │
│  │ │         error_str=str(e),                                       │ │    │
│  │ │         traceback_str=...,                                      │ │    │
│  │ │         model=...,                                              │ │    │
│  │ │         cache_hit=...,                                          │ │    │
│  │ │         custom_llm_provider=...,                                │ │    │
│  │ │         base_model=...,                                         │ │    │
│  │ │         call_type=...,                                          │ │    │
│  │ │         custom_pricing=...,                                     │ │    │
│  │ │     )                                                           │ │    │
│  │ │     self.model_call_details[                                    │ │    │
│  │ │         "response_cost_failure_debug_information"              │ │    │
│  │ │     ] = debug_info                                              │ │    │
│  │ │     return None                                                 │ │    │
│  │ │                                                                 │ │    │
│  │ └─────────────────────────────────────────────────────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 上游费用传递机制

**位置**: 多处使用，典型场景在 `litellm/proxy/pass_through_endpoints/llm_provider_handlers/cohere_passthrough_logging_handler.py:111-118`

```python
# 示例：在 passthrough 处理器中预计算费用并存储

# 1. 计算费用
response_cost = litellm.completion_cost(
    completion_response=litellm_model_response,
    model=model,
    custom_llm_provider="cohere",
    call_type="aembedding",
)

# 2. 存储到 _hidden_params 避免重复计算
if not hasattr(litellm_model_response, "_hidden_params"):
    litellm_model_response._hidden_params = {}
litellm_model_response._hidden_params["response_cost"] = response_cost

# 3. 传递给 kwargs 供后续使用
kwargs["response_cost"] = response_cost
```

**设计目的**:
- 在 passthrough 模式下，上游可能已经计算或返回了费用
- 将费用存储在 `_hidden_params` 中，避免在后续日志流程中重复计算
- 确保费用计算的一致性（使用相同的逻辑）

### 2.4 Response API 中的上游费用

**位置**: `litellm/responses/utils.py:971-973`

```python
# 保留上游已计算的费用
if hasattr(response_api_usage, "cost") and response_api_usage.cost is not None:
    setattr(chat_usage, "cost", response_api_usage.cost)
```

**说明**: 某些 API（如 OpenAI Responses API）可能在响应中直接返回 `cost` 字段，LiteLLM 会保留这个值。

### 2.5 决策流程图总结

```
                    ┌─────────────────────────────────────┐
                    │         费用计算请求到达             │
                    └──────────────────┬──────────────────┘
                                       │
                    ┌──────────────────▼──────────────────┐
                    │      优先级 1: 检查 cache_hit       │
                    │    (从 model_call_details 获取)      │
                    └──────────────────┬──────────────────┘
                                       │
              ┌────────────────────────┼────────────────────────┐
              │                        │                        │
    ┌─────────▼─────────┐    ┌─────────▼─────────┐    ┌─────────▼─────────┐
    │   cache_hit=True  │    │  cache_hit=False  │    │   cache_hit=None  │
    │   (缓存命中)       │    │   (未命中缓存)     │    │  (未知，继续检查)  │
    └─────────┬─────────┘    └─────────┬─────────┘    └─────────┬─────────┘
              │                        │                        │
              │                        │                        │
    ┌─────────▼─────────┐              │                        │
    │   return 0.0      │              │                        │
    │   (缓存免费)       │              │                        │
    └───────────────────┘              │                        │
                                       │                        │
                    ┌──────────────────▼────────────────────────┘
                    │
                    ▼
    ┌─────────────────────────────────────────────────────────────┐
    │              优先级 2: 检查 _hidden_params["response_cost"]  │
    │              (上游是否已预计算费用)                           │
    └───────────────────────────────┬─────────────────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
    ┌─────────▼─────────┐  ┌───────▼───────┐  ┌─────────▼─────────┐
    │  response_cost    │  │response_cost  │  │  response_cost    │
    │    存在且非空      │  │   不存在      │  │     为 None       │
    └─────────┬─────────┘  └───────┬───────┘  └─────────┬─────────┘
              │                     │                     │
              │                     │                     │
    ┌─────────▼─────────┐           │                     │
    │  return 上游费用   │           │                     │
    │  (避免重复计算)    │           │                     │
    └───────────────────┘           │                     │
                                    │                     │
                    ┌───────────────▼─────────────────────┘
                    │
                    ▼
    ┌─────────────────────────────────────────────────────────────┐
    │              优先级 3: 本地重算费用                          │
    │                                                               │
    │  1. 检查是否使用自定义定价 (custom_pricing)                   │
    │  2. 构建 response_cost_calculator_kwargs                     │
    │  3. 调用 litellm.response_cost_calculator()                  │
    │  4. 如果计算失败，记录 debug_info 供排查                      │
    └───────────────────────────────┬─────────────────────────────┘
                                    │
                                    ▼
    ┌─────────────────────────────────────────────────────────────┐
    │                    返回计算结果 (或 None)                     │
    └─────────────────────────────────────────────────────────────┘
```

---

## 三、回调管理器的真实接口与挂载方式

### 3.1 整体架构

LiteLLM 的回调系统采用三层架构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LiteLLM 回调系统架构                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  第一层: 全局回调列表 (litellm 模块级变量)                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  litellm.callbacks           # 通用回调（成功+失败都触发）        │   │
│  │  litellm.success_callback    # 成功回调                          │   │
│  │  litellm.failure_callback    # 失败回调                          │   │
│  │  litellm._async_success_callback  # 异步成功回调                 │   │
│  │  litellm._async_failure_callback  # 异步失败回调                 │   │
│  │  litellm.input_callback      # 输入预处理回调                    │   │
│  │  litellm.service_callback    # 服务级回调                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│                                    ▼                                     │
│  第二层: LoggingCallbackManager (统一管理)                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  职责:                                                           │   │
│  │  - 防止重复添加回调                                              │   │
│  │  - 自动路由同步/异步回调                                         │   │
│  │  - 限制最大回调数量 (MAX_CALLBACKS=30)                          │   │
│  │  - 字符串回调 → GenericAPILogger 转换                          │   │
│  │  - 去重逻辑 (按类型、按 key)                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                     │
│                                    ▼                                     │
│  第三层: 回调执行点 (Logging 类中的 handler)                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  success_handler()  /  failure_handler()                         │   │
│  │                                                                   │   │
│  │  执行流程:                                                        │   │
│  │  1. 构建标准日志数据 (model_call_details, standard_logging_object)│   │
│  │  2. 遍历 callbacks 列表                                           │   │
│  │  3. 根据回调类型调用对应方法:                                      │   │
│  │     - CustomLogger 实例 → log_success_event() / async_*        │   │
│  │     - 字符串 → 按名称查找并执行                                   │   │
│  │     - Callable → 直接调用函数                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 LoggingCallbackManager 核心接口

**位置**: `litellm/litellm_core_utils/logging_callback_manager.py`

#### 类定义与核心配置

```python
class LoggingCallbackManager:
    """
    中心化的回调管理类
    
    设计目标:
    1. 防止重复添加回调
    2. 限制最大回调数量 (防止性能问题)
    3. 自动路由同步/异步回调
    """
    
    # 健康的最大回调数量（不太可能需要超过 30 个）
    MAX_CALLBACKS = 30
```

#### 回调添加接口

```python
# ┌─────────────────────────────────────────────────────────────────────┐
# │                      回调添加方法族                                    │
# └─────────────────────────────────────────────────────────────────────┘

# 1. 通用回调（成功和失败都触发）
def add_litellm_callback(self, callback: Union[CustomLogger, str, Callable])

# 2. 输入回调（请求前触发）
def add_litellm_input_callback(self, callback: Union[CustomLogger, str, Callable])

# 3. 服务回调
def add_litellm_service_callback(self, callback: Union[CustomLogger, str, Callable])

# 4. 成功回调（自动路由同步/异步）
def add_litellm_success_callback(
    self, callback: Union[CustomLogger, str, Callable]
)

# 5. 失败回调（自动路由同步/异步）
def add_litellm_failure_callback(
    self, callback: Union[CustomLogger, str, Callable]
)

# 6. 显式异步成功回调
def add_litellm_async_success_callback(
    self, callback: Union[CustomLogger, Callable, str]
)

# 7. 显式异步失败回调
def add_litellm_async_failure_callback(
    self, callback: Union[CustomLogger, Callable, str]
)
```

#### 自动路由逻辑

**位置**: `logging_callback_manager.py:74-109`

```python
def add_litellm_success_callback(
    self, callback: Union[CustomLogger, str, Callable]
):
    """
    Add a success callback to `litellm.success_callback`.
    Auto-routes async callbacks to litellm._async_success_callback.
    Special-cases 'dynamodb' and 'openmeter' as async callbacks.
    """
    
    # 特殊处理：某些字符串回调默认是异步的
    if isinstance(callback, str) and callback in ("dynamodb", "openmeter"):
        self._safe_add_callback_to_list(
            callback=callback, parent_list=litellm._async_success_callback
        )
    # 检查是否是异步 Callable
    elif not isinstance(callback, str) and self._is_async_callable(callback):
        self._safe_add_callback_to_list(
            callback=callback, parent_list=litellm._async_success_callback
        )
    # 默认：同步回调
    else:
        self._safe_add_callback_to_list(
            callback=callback, parent_list=litellm.success_callback
        )
```

**自动路由规则**:
| 回调类型 | 路由目标 | 判断条件 |
|---------|---------|---------|
| 字符串 `"dynamodb"` | `_async_success_callback` | 硬编码特例 |
| 字符串 `"openmeter"` | `_async_success_callback` | 硬编码特例 |
| 异步 Callable | `_async_*_callback` | `coroutine_checker.is_async_callable()` |
| 同步 Callable | `success_callback/failure_callback` | 其他情况 |
| CustomLogger 实例 | 根据方法是否异步 | 运行时判断 |

#### 安全添加逻辑

**位置**: `logging_callback_manager.py:283-318`

```python
def _safe_add_callback_to_list(
    self,
    callback: Union[CustomLogger, Callable, str],
    parent_list: List[Union[CustomLogger, Callable, str]],
):
    """
    安全添加回调到列表，防止重复
    
    处理三种回调类型:
    1. str → 可能是 GenericAPILogger 或简单字符串
    2. CustomLogger → 按类型去重
    3. Callable → 按函数对象去重
    """
    
    # 1. 先检查是否超过最大数量限制
    if not self._check_callback_list_size(parent_list):
        return  # 超过限制，静默失败（记录 warning log）
    
    # 2. 字符串回调：可能需要转换为 GenericAPILogger
    if isinstance(callback, str):
        callback = LoggingCallbackManager._add_custom_callback_generic_api_str(
            callback
        )
    
    # 3. 根据类型分发
    if isinstance(callback, str):
        self._add_string_callback_to_list(
            callback=callback, parent_list=parent_list
        )
    elif isinstance(callback, CustomLogger):
        self._add_custom_logger_to_list(
            custom_logger=callback,
            parent_list=parent_list,
        )
    elif callable(callback):
        self._add_callback_function_to_list(
            callback=callback, parent_list=parent_list
        )
```

#### 去重策略

不同类型的回调有不同的去重规则：

```python
# ┌─────────────────────────────────────────────────────────────────────┐
# │                      字符串回调去重                                    │
# └─────────────────────────────────────────────────────────────────────┘
def _add_string_callback_to_list(
    self, callback: str, parent_list: List[Union[CustomLogger, Callable, str]]
):
    """字符串直接比较值是否已存在"""
    if callback not in parent_list:
        parent_list.append(callback)


# ┌─────────────────────────────────────────────────────────────────────┐
# │                   CustomLogger 回调去重                               │
# └─────────────────────────────────────────────────────────────────────┘
def _add_custom_logger_to_list(
    self,
    custom_logger: CustomLogger,
    parent_list: List[Union[CustomLogger, Callable, str]],
):
    """
    按类型和关键实例变量去重
    
    两个 CustomLogger 被认为重复，如果:
    1. 是同一个类的实例
    2. 关键实例变量（str, bool, int 类型的非私有属性）相同
    """
    custom_logger_key = self._get_custom_logger_key(custom_logger)
    custom_logger_type_name = type(custom_logger).__name__
    
    for existing_logger in parent_list:
        if (
            isinstance(existing_logger, CustomLogger)
            and self._get_custom_logger_key(existing_logger) == custom_logger_key
        ):
            # 已存在相同的 logger，不添加
            return
    
    parent_list.append(custom_logger)


# ┌─────────────────────────────────────────────────────────────────────┐
# │                   CustomLogger Key 生成规则                           │
# └─────────────────────────────────────────────────────────────────────┘
def _get_custom_logger_key(self, custom_logger: CustomLogger):
    """
    生成唯一 key，只考虑基本类型的实例变量
    
    规则:
    - 类名作为基础
    - 加上所有非私有、非下划线开头的 str/bool/int 类型属性
    - 格式: "ClassName-attr1=value1-attr2=value2"
    """
    key_parts = [type(custom_logger).__name__]
    
    for attr_name, attr_value in vars(custom_logger).items():
        if not attr_name.startswith("_"):  # 跳过私有属性
            if isinstance(attr_value, (str, bool, int)):
                key_parts.append(f"{attr_name}={attr_value}")
    
    return "-".join(key_parts)


# ┌─────────────────────────────────────────────────────────────────────┐
# │                      Callable 回调去重                                │
# └─────────────────────────────────────────────────────────────────────┘
def _add_callback_function_to_list(
    self, callback: Callable, parent_list: List[Union[CustomLogger, Callable, str]]
):
    """函数对象直接比较引用"""
    if callback not in parent_list:
        parent_list.append(callback)
```

### 3.3 字符串回调 → GenericAPILogger 转换

**位置**: `logging_callback_manager.py:198-281`

LiteLLM 支持通过配置文件定义回调，字符串会被转换为 `GenericAPILogger` 实例。

#### 转换流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│              字符串回调 → GenericAPILogger 转换流程                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  输入: callback: str (如 "my_webhook" 或 "langfuse")                   │
│                                                                          │
│  步骤 1: 检查 callback_settings 中的显式配置                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  callback_config = litellm.callback_settings.get(callback)       │   │
│  │                                                                   │   │
│  │  如果配置存在且 callback_type == "generic_api":                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐ │   │
│  │  │ 示例配置:                                                     │ │   │
│  │  │ litellm_settings:                                            │ │   │
│  │  │   success_callback: ["my_webhook"]                           │ │   │
│  │  │                                                               │ │   │
│  │  │ callback_settings:                                            │ │   │
│  │  │   my_webhook:                                                 │ │   │
│  │  │     callback_type: generic_api                                │ │   │
│  │  │     endpoint: https://webhook-test.com/xxx                   │ │   │
│  │  │     headers:                                                  │ │   │
│  │  │       Authorization: Bearer sk-1234                           │ │   │
│  │  │     event_types: ["success", "failure"]  # 可选              │ │   │
│  │  │     log_format: "openai"  # 可选                              │ │   │
│  │  │     max_retries: 3  # 可选                                    │ │   │
│  │  │     retry_delay: 1.0  # 可选                                  │ │   │
│  │  │     timeout: 30  # 可选                                       │ │   │
│  │  └─────────────────────────────────────────────────────────────┘ │   │
│  │                                                                   │   │
│  │  构建 GenericAPILogger:                                           │   │
│  │  new_logger = GenericAPILogger(                                  │   │
│  │      endpoint=endpoint,                                           │   │
│  │      headers=headers,                                             │   │
│  │      event_types=event_types,                                     │   │
│  │      log_format=log_format,                                       │   │
│  │      max_retries=max_retries,                                     │   │
│  │      retry_delay=retry_delay,                                     │   │
│  │      timeout=timeout,                                             │   │
│  │  )                                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  步骤 2: 检查预定义的兼容回调                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  from litellm.integrations.generic_api.generic_api_callback import │   │
│  │       is_callback_compatible                                      │   │
│  │                                                                   │   │
│  │  if is_callback_compatible(callback):                             │   │
│  │      # 如 "langfuse", "langsmith" 等预定义回调                   │   │
│  │      # 从 generic_api_compatible_callbacks.json 加载配置         │   │
│  │                                                                   │   │
│  │      new_logger = GenericAPILogger(callback_name=callback)      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  步骤 3: 都不满足，保持为字符串                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  return callback  # 按原始字符串处理                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

#### 缓存机制

```python
# 全局缓存，避免重复创建相同配置的 GenericAPILogger
_generic_api_logger_cache: Dict[str, GenericAPILogger] = {}

# 使用缓存
cached_logger = _generic_api_logger_cache.get(callback)
if (
    isinstance(cached_logger, GenericAPILogger)
    and cached_logger.endpoint == endpoint
    and cached_logger.headers == headers
    # ... 其他配置比较
):
    return cached_logger  # 复用缓存

# 否则创建新实例并缓存
new_logger = GenericAPILogger(...)
_generic_api_logger_cache[callback] = new_logger
return new_logger
```

### 3.4 CustomLogger 接口定义

**位置**: `litellm/integrations/custom_logger.py`

`CustomLogger` 是所有自定义回调的基类，定义了完整的接口规范。

#### 核心日志接口

```python
class CustomLogger:
    """
    自定义回调基类
    
    文档: https://docs.litellm.ai/docs/observability/custom_callback#callback-class
    """
    
    def __init__(
        self,
        turn_off_message_logging: bool = False,
        message_logging: bool = True,  # 已废弃，使用 turn_off_message_logging
        **kwargs,
    ) -> None:
        """
        Args:
            turn_off_message_logging: 如果为 True，消息内容会在日志中被脱敏
        """
        self.message_logging = message_logging
        self.turn_off_message_logging = turn_off_message_logging


    # ─────────────────────────────────────────────────────────────────
    # 同步日志接口
    # ─────────────────────────────────────────────────────────────────
    
    def log_pre_api_call(self, model, messages, kwargs):
        """API 调用前触发"""
        pass
    
    def log_post_api_call(self, kwargs, response_obj, start_time, end_time):
        """API 调用后触发（成功或失败）"""
        pass
    
    def log_stream_event(self, kwargs, response_obj, start_time, end_time):
        """流式响应的每个 chunk 触发"""
        pass
    
    def log_success_event(self, kwargs, response_obj, start_time, end_time):
        """请求成功时触发"""
        pass
    
    def log_failure_event(self, kwargs, response_obj, start_time, end_time):
        """请求失败时触发"""
        pass


    # ─────────────────────────────────────────────────────────────────
    # 异步日志接口
    # ─────────────────────────────────────────────────────────────────
    
    async def async_log_stream_event(self, kwargs, response_obj, start_time, end_time):
        pass
    
    async def async_log_pre_api_call(self, model, messages, kwargs):
        pass
    
    async def async_log_success_event(self, kwargs, response_obj, start_time, end_time):
        pass
    
    async def async_log_failure_event(self, kwargs, response_obj, start_time, end_time):
        pass
    
    async def async_log_audit_log_event(self, audit_log: "StandardAuditLogPayload"):
        """审计日志创建时触发"""
        pass
```

#### Hook 接口（Router/Proxy 专用）

```python
    # ─────────────────────────────────────────────────────────────────
    # 路由前 Hook
    # ─────────────────────────────────────────────────────────────────
    
    async def async_pre_routing_hook(
        self,
        model: str,
        request_kwargs: Dict,
        messages: Optional[List[Dict[str, Any]]] = None,
        input: Optional[Union[str, List]] = None,
        specific_deployment: Optional[bool] = False,
    ) -> Optional[PreRoutingHookResponse]:
        """
        路由决策前调用
        
        用途: 用于 auto-router 在路由决策前修改请求
        """
        return None
    
    async def async_filter_deployments(
        self,
        model: str,
        healthy_deployments: List,
        messages: Optional[List[AllMessageValues]],
        request_kwargs: Optional[dict] = None,
        parent_otel_span: Optional[Span] = None,
    ) -> List[dict]:
        """过滤可用的部署列表"""
        return healthy_deployments


    # ─────────────────────────────────────────────────────────────────
    # 调用前后 Hook
    # ─────────────────────────────────────────────────────────────────
    
    async def async_pre_call_hook(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        cache: "DualCache",
        data: dict,
        call_type: CallTypesLiteral,
    ) -> Optional[Union[Exception, str, dict]]:
        """
        调用模型前的 Hook（Proxy 专用）
        
        返回:
        - Exception: 抛出异常，拒绝请求
        - str: 返回给用户的拒绝消息
        - dict: 修改后的请求参数
        - None: 不做修改
        """
        pass
    
    async def async_post_call_success_hook(
        self,
        data: dict,
        user_api_key_dict: UserAPIKeyAuth,
        response: LLMResponseTypes,
    ) -> Any:
        """成功响应后的 Hook"""
        pass
    
    async def async_post_call_failure_hook(
        self,
        request_data: dict,
        original_exception: Exception,
        user_api_key_dict: UserAPIKeyAuth,
        traceback_str: Optional[str] = None,
    ) -> Optional["HTTPException"]:
        """
        失败响应后的 Hook
        
        可以返回/抛出 HTTPException 来转换错误响应
        """
        pass
    
    async def async_post_call_response_headers_hook(
        self,
        data: dict,
        user_api_key_dict: UserAPIKeyAuth,
        response: Any,
        request_headers: Optional[Dict[str, str]] = None,
        litellm_call_info: Optional[Dict[str, Any]] = None,
    ) -> Optional[Dict[str, str]]:
        """
        注入自定义 HTTP 响应头
        
        返回: 要注入的响应头字典，或 None
        """
        return None


    # ─────────────────────────────────────────────────────────────────
    # 流式 Hook
    # ─────────────────────────────────────────────────────────────────
    
    async def async_post_call_streaming_hook(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        response: str,
    ) -> Any:
        pass
    
    async def async_post_call_streaming_iterator_hook(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        response: Any,
        request_data: dict,
    ) -> AsyncGenerator[ModelResponseStream, None]:
        """
        流式响应迭代器 Hook
        
        可以用来修改或拦截流式 chunk
        """
        async for item in response:
            yield item


    # ─────────────────────────────────────────────────────────────────
    # 日志脱敏 Hook
    # ─────────────────────────────────────────────────────────────────
    
    async def async_logging_hook(
        self, kwargs: dict, result: Any, call_type: str
    ) -> Tuple[dict, Any]:
        """
        日志内容脱敏 Hook
        
        返回修改后的 (kwargs, result)
        """
        return kwargs, result
    
    def logging_hook(
        self, kwargs: dict, result: Any, call_type: str
    ) -> Tuple[dict, Any]:
        """同步版本"""
        return kwargs, result
```

#### Agentic Loop Hook（高级）

```python
    # ─────────────────────────────────────────────────────────────────
    # Agentic Loop Hook（用于服务器端工具执行）
    # ─────────────────────────────────────────────────────────────────
    
    async def async_should_run_agentic_loop(
        self,
        response: Any,
        model: str,
        messages: List[Dict],
        tools: Optional[List[Dict]],
        stream: bool,
        custom_llm_provider: str,
        kwargs: Dict,
    ) -> Tuple[bool, Dict]:
        """
        判断是否应该执行 Agentic Loop
        
        用途: 透明的服务器端工具执行
        - 用户发送一个 API 调用
        - 模型返回 tool_use
        - 这个 Hook 判断是否应该服务器端执行工具
        - 如果是，执行工具并返回最终答案
        
        返回: (should_run: bool, tools: Dict)
        """
        return False, {}
    
    async def async_run_agentic_loop(
        self,
        tools: Dict,
        model: str,
        messages: List[Dict],
        response: Any,
        anthropic_messages_provider_config: Any,
        anthropic_messages_optional_request_params: Dict,
        logging_obj: "LiteLLMLoggingObj",
        stream: bool,
        kwargs: Dict,
    ) -> Any:
        """
        执行 Agentic Loop
        
        只有当 async_should_run_agentic_loop 返回 True 时才调用
        """
        pass
```

#### 工具方法

```python
    # ─────────────────────────────────────────────────────────────────
    # 实用工具方法
    # ─────────────────────────────────────────────────────────────────
    
    def truncate_standard_logging_payload_content(
        self,
        standard_logging_object: StandardLoggingPayload,
    ):
        """
        截断日志 payload 中的长字符串
        
        某些日志服务（如 DataDog、GCS）对 payload 大小有限制（1MB）
        这个方法截断 error_str、messages、response 等字段
        """
        MAX_STR_LENGTH = 10_000
        # ...
    
    def redact_standard_logging_payload_from_model_call_details(
        self, model_call_details: Dict
    ) -> Dict:
        """
        脱敏或排除日志 payload 中的字段
        
        处理两种情况:
        1. turn_off_message_logging: 脱敏 messages 和 responses
        2. standard_logging_payload_excluded_fields: 完全排除指定字段
        """
        # ...
    
    def handle_callback_failure(self, callback_name: str):
        """
        处理回调执行失败
        
        增加 Prometheus 指标（如果配置了）
        """
        # ...
```

### 3.5 回调执行时机

**位置**: `litellm/litellm_core_utils/litellm_logging.py` 中的 `success_handler()` 和 `failure_handler()`

#### 执行流程（简化版）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      回调执行流程（以 success_handler 为例）              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 准备阶段                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ - 计算 response_ms (响应时间)                                     │   │
│  │ - 计算 response_cost (费用)                                       │   │
│  │ - 构建 model_call_details (完整的调用详情)                        │   │
│  │ - 构建 standard_logging_object (标准日志格式)                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  2. 执行同步回调                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 遍历 litellm.success_callback + litellm.callbacks                │   │
│  │                                                                   │   │
│  │ 对于每个 callback:                                                │   │
│  │ ┌─────────────────────────────────────────────────────────────┐ │   │
│  │ │ 类型判断:                                                      │ │   │
│  │ │                                                                 │ │   │
│  │ │ if isinstance(callback, CustomLogger):                        │ │   │
│  │ │     callback.log_success_event(...)                            │ │   │
│  │ │                                                                 │ │   │
│  │ │ elif isinstance(callback, str):                                │ │   │
│  │ │     # 按名称查找预定义回调（如 "langfuse"）                    │ │   │
│  │ │     # 通过 CustomLoggerRegistry 查找                           │ │   │
│  │ │                                                                 │ │   │
│  │ │ elif callable(callback):                                       │ │   │
│  │ │     callback(kwargs, response_obj, start_time, end_time)      │ │   │
│  │ └─────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  3. 执行异步回调（在单独的 task 中）                                      │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 遍历 litellm._async_success_callback + litellm.callbacks 中的   │   │
│  │ 异步部分                                                          │   │
│  │                                                                   │   │
│  │ 使用 asyncio.create_task() 或 LoggingWorker 执行                │   │
│  │ 不阻塞主线程                                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.6 回调数据结构

回调函数接收到的 `kwargs` 包含丰富的上下文信息：

```python
# kwargs 中的关键字段（来自 model_call_details）
{
    # 基本信息
    "model": "gpt-4",
    "custom_llm_provider": "openai",
    "call_type": "completion",
    
    # 费用信息
    "response_cost": 0.0025,
    "cost_breakdown": {
        "input_cost": 0.001,
        "output_cost": 0.0015,
        "total_cost": 0.0025,
        "cache_read_cost": 0.0,
        "cache_creation_cost": 0.0,
        "original_cost": 0.0025,
        "discount_percent": 0.0,
        "margin_percent": 0.0,
    },
    
    # Token 使用
    "prompt_tokens": 100,
    "completion_tokens": 50,
    "total_tokens": 150,
    
    # 时间信息
    "start_time": datetime.datetime(...),
    "end_time": datetime.datetime(...),
    "response_ms": 1234.5,
    
    # 请求/响应内容
    "messages": [...],  # 用户消息
    "response": ModelResponse(...),  # 模型响应
    
    # 标准日志对象
    "standard_logging_object": StandardLoggingPayload(...),
    
    # 其他元数据
    "metadata": {...},  # 用户传入的 metadata
    "cache_hit": False,
    "stream": False,
}
```

---

## 四、关键代码位置速查

### 4.1 Usage 转换相关

| 功能 | 文件位置 | 函数/方法名 |
|-----|---------|------------|
| 主转换流程 | `litellm/cost_calculator.py:1140-1235` | 内联在 `completion_cost()` 中 |
| Response API 转换 | `litellm/responses/utils.py:900-975` | `_transform_response_api_usage_to_chat_usage()` |
| Response API 识别 | `litellm/responses/utils.py:890-897` | `_is_response_api_usage()` |
| Transcription 转换 | `litellm/litellm_core_utils/llm_cost_calc/usage_object_transformation.py` | `TranscriptionUsageObjectTransformation` 类 |

### 4.2 Response Cost 优先级相关

| 功能 | 文件位置 | 函数/方法名 |
|-----|---------|------------|
| 主决策流程 | `litellm/litellm_core_utils/litellm_logging.py:1451-1586` | `_response_cost_calculator()` |
| 缓存命中检查 | `litellm/litellm_core_utils/litellm_logging.py:1483-1487` | 内联 |
| 上游费用检查 | `litellm/litellm_core_utils/litellm_logging.py:1489-1499` | 内联 |
| 上游费用传递示例 | `litellm/proxy/pass_through_endpoints/llm_provider_handlers/cohere_passthrough_logging_handler.py:111-118` | 内联 |

### 4.3 回调管理器相关

| 功能 | 文件位置 | 函数/方法名 |
|-----|---------|------------|
| 回调管理器类 | `litellm/litellm_core_utils/logging_callback_manager.py` | `LoggingCallbackManager` 类 |
| CustomLogger 基类 | `litellm/integrations/custom_logger.py` | `CustomLogger` 类 |
| 成功回调添加 | `litellm/litellm_core_utils/logging_callback_manager.py:74-93` | `add_litellm_success_callback()` |
| 安全添加逻辑 | `litellm/litellm_core_utils/logging_callback_manager.py:283-318` | `_safe_add_callback_to_list()` |
| 字符串回调转换 | `litellm/litellm_core_utils/logging_callback_manager.py:198-281` | `_add_custom_callback_generic_api_str()` |
| 回调执行点 | `litellm/litellm_core_utils/litellm_logging.py:2007+` | `success_handler()`, `failure_handler()` |

---

## 五、设计要点总结

### 5.1 Usage 转换设计要点

1. **多层识别**: 先判断类型（Response API / Transcription / 标准），再应用对应转换
2. **字段保留**: 转换时保留所有可能有用的字段（如 `cached_tokens`, `reasoning_tokens`）
3. **费用透传**: 如果上游已经计算了 `cost`，会保留这个值（避免重复计算或不一致）
4. **未知类型处理**: 对于未知的 BaseModel usage 对象，转换为标准 `litellm.Usage`

### 5.2 Response Cost 优先级设计要点

1. **缓存优先**: 缓存命中直接返回 0，这是最高优先级（业务逻辑决定）
2. **上游信任**: 如果 `_hidden_params["response_cost"]` 存在，直接使用（避免重复计算）
3. **本地兜底**: 以上都不满足时，才调用本地计算器
4. **可观测性**: 计算失败时记录详细的 `debug_info`，便于排查

### 5.3 回调系统设计要点

1. **三层架构**: 全局列表 → 管理器 → 执行点，职责分离
2. **自动路由**: 同步/异步回调自动识别和路由
3. **防止滥用**: `MAX_CALLBACKS=30` 限制防止性能问题
4. **灵活去重**:
   - 字符串: 按值去重
   - CustomLogger: 按类型+关键属性去重
   - Callable: 按引用去重
5. **配置驱动**: 支持通过配置文件定义 `GenericAPILogger`
6. **完整接口**: `CustomLogger` 定义了从日志到 Hook 的完整接口规范
