# LiteLLM 费用追踪分支深度分析

## 概述

本文档深入分析 LiteLLM 费用追踪链路中的三个关键问题：

1. **上游直接给费用时的信任机制**：哪些 provider 和字段会被优先信任
2. **回退本地重算的场景**：哪些情况下仍会回退到本地重算
3. **本地重算的依赖**：本地重算依赖哪些前置条件和调用参数
4. **日志落点一致性**：这些分支在日志对象里的落点是否一致

---

## 一、上游直接给费用的信任机制

### 1.1 信任字段体系

LiteLLM 有多层次的上游费用信任字段，按优先级从高到低排列：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    上游费用信任字段优先级                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  优先级 1 (最高): model_call_details["cache_hit"] == True              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 场景: 请求命中 LiteLLM 缓存                                      │   │
│  │ 行为: response_cost = 0.0（缓存命中免费）                          │   │
│  │ 设置位置: 多个位置在调用前设置                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  优先级 2: _hidden_params["response_cost"] 存在且非空                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 场景: Passthrough 模式下，handler 直接计算并设置                  │   │
│  │ 行为: 直接使用这个值                                              │   │
│  │ 设置位置: passthrough_logging_handlers                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  优先级 3: model_call_details["response_cost"] 已存在                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 场景: 各种 handler 直接在日志对象中设置                           │   │
│  │ 行为: 保留已有的值（不覆盖）                                     │   │
│  │ 设置位置: passthrough handlers、A2A protocol 等                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  优先级 4: _hidden_params["additional_headers"]                         │
│           ["llm_provider-x-litellm-response-cost"]                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 场景: Provider 从上游响应中提取费用                               │   │
│  │ 行为: 在 _response_cost_calculator 中检查并返回                  │   │
│  │ 设置位置: OpenRouter、Stability AI、Bedrock 等 provider          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  优先级 5 (最低): 本地重算                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 场景: 以上都不满足                                               │   │
│  │ 行为: 调用 completion_cost() 根据 usage 和 model_info 计算       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 优先级 1: 缓存命中

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1758-1759

```python
if self.model_call_details.get("cache_hit") is True:
    self.model_call_details["response_cost"] = 0.0
```

**设置场景**:
- LiteLLM 缓存层在命中缓存时设置
- 任何使用 `litellm.caching` 模块的场景

**特点**:
- 这是最高优先级，无论其他字段是否存在
- 缓存命中直接返回 0 费用

### 1.3 优先级 2: `_hidden_params["response_cost"]`

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1760-1761

```python
elif "response_cost" in hidden_params:
    self.model_call_details["response_cost"] = hidden_params["response_cost"]
```

**设置此字段的 Provider/Handler**:

| Handler | 文件位置 | 场景 |
|---------|-----------|------|
| **Cohere Passthrough** | `cohere_passthrough_logging_handler.py:125` | Cohere 嵌入请求 |
| **Gemini Passthrough** | `gemini_passthrough_logging_handler.py:71` | Gemini 视频请求 |
| **OpenAI Passthrough** | `openai_passthrough_logging_handler.py:312, 337` | OpenAI 各种请求 |
| **Vertex Passthrough** | `vertex_passthrough_logging_handler.py:84` | Vertex 视频请求 |
| **Files Streaming** | `files/streaming.py:147-148` | 文件操作（默认设为 None） |

**设置方式示例** (`cohere_passthrough_logging_handler.py:111-125

```python
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
```

**特点**:
- 主要用于 passthrough 模式下，handler 已经调用了 `completion_cost()` 计算费用
-将结果存储在 `_hidden_params` 避免后续重复计算

### 1.4 优先级 3: `model_call_details["response_cost"]` 已存在

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1762-1765

```python
elif self.model_call_details.get("response_cost") is not None:
    # Preserve response_cost if already calculated (e.g., by pass-through
    # handlers like Gemini/Vertex which call completion_cost directly)
    pass
```

**设置此字段的场景**:

| 场景 | 文件位置 | 说明 |
|-----|---------|------|
| **Anthropic Passthrough** | `anthropic_passthrough_logging_handler.py:450` | Batch 请求 |
| **AssemblyAI Passthrough** | `assembly_passthrough_logging_handler.py:152` | 转录请求 |
| **Cohere Passthrough** | `cohere_passthrough_logging_handler.py:159` | 嵌入请求（同时设置两处 |
| **Cursor Passthrough** | `cursor_passthrough_logging_handler.py:108` | 直接设为 0.0 |
| **Gemini Passthrough** | `gemini_passthrough_logging_handler.py:76, 253` | 视频、生成内容请求 |
| **OpenAI Passthrough** | `openai_passthrough_logging_handler.py:396, 588` | 各种请求 |
| **Vertex Passthrough** | `vertex_passthrough_logging_handler.py:89, 225, 326, 405, 750` | 视频、搜索、生成内容、Batch 请求 |
| **Success Handler** | `pass_through_endpoints/success_handler.py:492` | 从 `cost_per_request` 获取 |
| **A2A Protocol** | `a2a_protocol/streaming_iterator.py:134` | Agent-to-Agent 协议 |
| **MCP Tool Call** | `litellm_logging.py:1374` | 从 `response.hidden_params.response_cost` 获取 |

**设置方式示例** (`anthropic_passthrough_logging_handler.py:448-450

```python
logging_obj.model = model_name
logging_obj.model_call_details["model"] = logging_obj.model
logging_obj.model_call_details["response_cost"] = response_cost
logging_obj.model_call_details["batch_id"] = batch_id
```

**特点**:
- 这是最直接的设置方式，直接操作日志对象
- 注释说明："Preserve response_cost if already calculated"
- 各种 passthrough handler 会同时设置 `_hidden_params["response_cost"]` 和 `model_call_details["response_cost"]` 以确保安全

### 1.5 优先级 4: `llm_provider-x-litellm-response-cost`

**位置**: 
- 检查: `litellm/cost_calculator.py:1668-1685` (`get_response_cost_from_hidden_params`)
- 使用: `litellm/cost_calculator.py:1749-1753` (`response_cost_calculator`)

**检查逻辑**:

```python
def get_response_cost_from_hidden_params(
    hidden_params: Union[dict, BaseModel],
) -> Optional[float]:
    if isinstance(hidden_params, BaseModel):
        _hidden_params_dict = cast(BaseModel, hidden_params).model_dump()
    else:
        _hidden_params_dict = hidden_params

    additional_headers = _hidden_params_dict.get("additional_headers", {})
    if (
        additional_headers
        and "llm_provider-x-litellm-response-cost" in additional_headers
    ):
        response_cost = additional_headers["llm_provider-x-litellm-response-cost"]
        if response_cost is None:
            return None
        return float(additional_headers["llm_provider-x-litellm-response-cost"])
    return None
```

**使用逻辑** (`response_cost_calculator` 中:

```python
if isinstance(response_object, BaseModel):
    if hasattr(response_object, "_hidden_params"):
        response_object._hidden_params["optional_params"] = optional_params
        provider_response_cost = get_response_cost_from_hidden_params(
            response_object._hidden_params
        )
        if provider_response_cost is not None:
            return provider_response_cost  # 直接返回上游费用
```

**设置此字段的 Provider**:

| Provider | 文件位置 | 费用来源 |
|---------|---------|---------|
| **OpenRouter** | `openrouter/chat/transformation.py:227-229` | 从 `response_json["usage"]["cost"]` |
| **OpenRouter Image Edit** | `openrouter/image_edit/transformation.py:347-349` | 从 `usage_data["cost"]` |
| **OpenRouter Image Gen** | `openrouter/image_generation/transformation.py:228-230` | 从 `usage_data["cost"]` |
| **Stability AI** | `stability/image_edit/transformations.py:313-315` | 从 `model_info["output_cost_per_image"]` |
| **Bedrock (Nova Canvas** | `bedrock/image_edit/amazon_nova_canvas_image_edit_transformation.py:458-460` | 从 `model_info["output_cost_per_image"]` × 图片数 |
| **Bedrock (Stability)** | `bedrock/image_edit/stability_transformation.py:339-341` | 从 `model_info["output_cost_per_image"]` |
| **OpenAI Containers** | `openai/containers/transformation.py:140-142` | 计算 `container_cost` |

**OpenRouter 示例** (`openrouter/chat/transformation.py:215-232

```python
# Extract cost from OpenRouter response body
# OpenRouter returns cost information in the usage object when usage.include=true
try:
    response_json = raw_response.json()
    if "usage" in response_json and response_json["usage"]:
        response_cost = response_json["usage"].get("cost")
        if response_cost is not None:
            # Store cost in hidden params for the cost calculator to use
            if not hasattr(model_response, "_hidden_params"):
                model_response._hidden_params = {}
            if "additional_headers" not in model_response._hidden_params:
                model_response._hidden_params["additional_headers"] = {}
            model_response._hidden_params["additional_headers"][
                "llm_provider-x-litellm-response-cost"
            ] = float(response_cost)
except Exception:
    # If we can't extract cost, continue without it - don't fail the response
    pass
```

**关键差异**:
- 与优先级 2/3 不同，这个字段是从**真正的上游费用**，不是 LiteLLM 计算的
- OpenRouter 从其 API 响应中直接提取 `usage.cost` 字段
- 其他 provider（如 Stability AI、Bedrock）实际上是**根据 `model_info` 计算后设置的（类似预计算）

### 1.6 额外费用明细字段

**位置**: 
- `_hidden_params["response_cost_details"]`

**设置场景**:

| Provider | 文件位置 | 说明 |
|---------|---------|------|
| **OpenRouter Image Edit** | `openrouter/image_edit/transformation.py:353-357` | 从 `usage_data["cost_details"]` 获取 |
| **OpenRouter Image Gen** | `openrouter/image_generation/transformation.py:234-238` | 从 `usage_data["cost_details"]` 获取 |

**示例**:

```python
cost_details = usage_data.get("cost_details", {})
if cost_details:
    if "response_cost_details" not in model_response._hidden_params:
        model_response._hidden_params["response_cost_details"] = {}
    model_response._hidden_params["response_cost_details"].update(
        cost_details
    )
```

**用途**: 存储更详细的费用构成（如模型费用、平台费用等）

---

## 二、回退本地重算的场景

### 2.1 回退条件

当以下**所有条件都不满足时，才会回退到本地重算：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    回退本地重算的条件判断                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  在 _process_hidden_params_and_response_cost 中的判断顺序:               │
│                                                                          │
│  IF cache_hit == True?                                                  │
│    ├── Yes → 不回退（费用 = 0.0）                                   │
│    └── No  → 继续判断下一个条件                                         │
│                                                                          │
│  "response_cost" in hidden_params?                                     │
│    ├── Yes → 不回退（使用这个值）                                      │
│    └── No  → 继续判断下一个条件                                         │
│                                                                          │
│  model_call_details["response_cost"] is not None?                       │
│    ├── Yes → 不回退（保留已有值）                                      │
│    └── No  → 继续判断下一个条件                                         │
│                                                                          │
│  以上都不满足 → 回退到本地重算                                        │
│    └── 调用 self._response_cost_calculator(result=logging_result)        │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 `_response_cost_calculator` 内部判断

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1451-1586

在 `_response_cost_calculator` 内部，还有一层判断：

```
┌─────────────────────────────────────────────────────────────────────────┐
│              _response_cost_calculator 内部判断逻辑                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  1. 检查 cache_hit                                                      │
│     if cache_hit is None:                                               │
│         cache_hit = self.model_call_details.get("cache_hit", False)       │
│                                                                          │
│     if cache_hit is True:                                               │
│         return 0.0  # 缓存命中，不重算也返回 0                       │
│                                                                          │
│  2. 检查 _hidden_params["response_cost"]                               │
│     if isinstance(result, BaseModel) and hasattr(result, "_hidden_params")│
│         hidden_params = getattr(result, "_hidden_params", {})              │
│                                                                          │
│         if ("response_cost" in hidden_params                            │
│             and hidden_params["response_cost"] is not None):            │
│             return hidden_params["response_cost"]  # 直接使用            │
│                                                                          │
│  3. 以上都不满足 → 调用 litellm.response_cost_calculator()             │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.3 `response_cost_calculator` 内部判断

**位置**: `litellm/cost_calculator.py:1737-1772

```python
def response_cost_calculator(...):
    try:
        response_cost: float = 0.0
        if cache_hit is not None and cache_hit is True:
            response_cost = 0.0
        else:
            if isinstance(response_object, BaseModel):
                if hasattr(response_object, "_hidden_params"):
                    provider_response_cost = get_response_cost_from_hidden_params(
                        response_object._hidden_params
                    )
                    if provider_response_cost is not None:
                        return provider_response_cost  # 优先使用上游费用

            # 以上都不满足 → 本地重算
            response_cost = completion_cost(...)
        return response_cost
    except Exception as e:
        raise e
```

### 2.4 典型回退场景

| 场景 | 说明 |
|-----|------|
| **标准文本补全** | 大多数非 passthrough 模式下的标准调用 |
| **非 OpenRouter 的 provider** | 大多数 provider 不会从 `model_info` 计算，不是从上游提取 |
| **流式响应** | 如果上游没有在每个 chunk 中返回费用 |
| **没有 usage 对象存在但上游没给费用** | 如标准 OpenAI 调用 |
| **新添加的自定义模型** | 没有预计算费用 |

**具体示例**：标准 OpenAI 调用

```python
# 标准调用流程
response = litellm.completion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hi"}]
)

# 此时：
# - cache_hit = False（默认）
# - _hidden_params["response_cost"] 不存在
# - model_call_details["response_cost"] 不存在
# - additional_headers["llm_provider-x-litellm-response-cost"] 不存在

# 因此会回退到本地重算：
# completion_cost(completion_response=response, model="gpt-3.5-turbo")
```

---

## 三、本地重算的依赖

### 3.1 `response_cost_calculator` 参数

**位置**: `litellm/cost_calculator.py:1688-1736

```python
def response_cost_calculator(
    response_object: Union[
        ModelResponse,
        EmbeddingResponse,
        ImageResponse,
        TranscriptionResponse,
        TextCompletionResponse,
        HttpxBinaryResponseContent,
        RerankResponse,
        ResponsesAPIResponse,
        LiteLLMRealtimeStreamLoggingObject,
        OpenAIModerationResponse,
        Response,
        SearchResponse,
    ],
    model: str,
    custom_llm_provider: Optional[str],
    call_type: Literal[
        "embedding",
        "aembedding",
        "completion",
        "acompletion",
        "atext_completion",
        "text_completion",
        "image_generation",
        "aimage_generation",
        "moderation",
        "amoderation",
        "atranscription",
        "transcription",
        "aspeech",
        "speech",
        "rerank",
        "arerank",
        "search",
        "asearch",
    ],
    optional_params: dict,
    cache_hit: Optional[bool] = None,
    base_model: Optional[str] = None,
    custom_pricing: Optional[bool] = None,
    prompt: str = "",
    standard_built_in_tools_params: Optional[StandardBuiltInToolsParams] = None,
    litellm_model_name: Optional[str] = None,
    router_model_id: Optional[str] = None,
    litellm_logging_obj: Optional[LitellmLoggingObject] = None,
    service_tier: Optional[str] = None,
) -> float:
```

### 3.2 `completion_cost` 参数

**位置**: `litellm/cost_calculator.py:1015-1042

```python
def completion_cost(
    completion_response=None,
    model: Optional[str] = None,
    prompt="",
    messages: List = [],
    completion="",
    total_time: Optional[float] = 0.0,
    call_type: Optional[CallTypesLiteral] = None,
    custom_llm_provider=None,
    region_name=None,
    size: Optional[str] = None,
    quality: Optional[str] = None,
    n: Optional[int] = None,
    custom_cost_per_token: Optional[CostPerToken] = None,
    custom_cost_per_second: Optional[float] = None,
    optional_params: Optional[dict] = None,
    custom_pricing: Optional[bool] = None,
    base_model: Optional[str] = None,
    standard_built_in_tools_params: Optional[StandardBuiltInToolsParams] = None,
    litellm_model_name: Optional[str] = None,
    router_model_id: Optional[str] = None,
    litellm_logging_obj: Optional[LitellmLoggingObject] = None,
    service_tier: Optional[str] = None,
) -> float:
```

### 3.3 本地重算的核心依赖

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    本地重算的核心依赖                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  前置条件:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. usage 对象 (从 response_object 提取)                        │   │
│  │    ├── prompt_tokens                                            │   │
│  │    ├── completion_tokens                                          │   │
│  │    ├── cache_creation_input_tokens (Prompt Caching)               │   │
│  │    ├── cache_read_input_tokens (Prompt Caching)                  │   │
│  │    └── reasoning_tokens (如适用)                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  数据来源:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. model_info (从 model_prices_and_context_window.json 获取)   │   │
│  │    ├── input_cost_per_token                                       │   │
│  │    ├── output_cost_per_token                                      │   │
│  │    ├── cache_read_input_token_cost                                 │   │
│  │    ├── cache_creation_input_token_cost                           │   │
│  │    ├── output_cost_per_reasoning_token                           │   │
│  │    └── 其他定价相关字段                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
│  计算逻辑:                                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. provider 特定的 cost_per_token 函数                            │   │
│  │    ├── 大多数 provider: generic_cost_per_token()                  │   │
│  │    ├── Anthropic: 特殊处理区域溢价、缓存费用                        │   │
│  │    ├── Dashscope: 分层定价处理                                    │   │
│  │    └── Vertex AI: 128k+ 分层定价处理                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4 本地重算失败的处理

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1546-1585

```python
try:
    response_cost = litellm.response_cost_calculator(
        **response_cost_calculator_kwargs
    )
    return response_cost
except Exception as e:
    debug_info = StandardLoggingModelCostFailureDebugInformation(
        error_str=str(e),
        traceback_str=_get_traceback_str_for_error(str(e)),
        model=response_cost_calculator_kwargs["model"],
        cache_hit=response_cost_calculator_kwargs["cache_hit"],
        custom_llm_provider=response_cost_calculator_kwargs["custom_llm_provider"],
        base_model=response_cost_calculator_kwargs["base_model"],
        call_type=response_cost_calculator_kwargs["call_type"],
        custom_pricing=response_cost_calculator_kwargs["custom_pricing"],
    )
    verbose_logger.debug(
        f"response_cost_failure_debug_information: {debug_info}"
    )
    self.model_call_details["response_cost_failure_debug_information"] = debug_info
    return None
```

**失败时**:
- 返回 `None`
- 记录详细的调试信息到 `model_call_details["response_cost_failure_debug_information"]`
- 包含：错误信息、堆栈、模型名、cache_hit、provider、base_model、call_type、custom_pricing

---

## 四、日志对象落点一致性

### 4.1 主要落点

所有分支最终都会将费用写入同一个位置：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    费用在日志对象中的落点                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  主要落点: model_call_details["response_cost"]                          │
│                                                                          │
│  设置位置:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. 缓存命中:                                                 │   │
│  │    if cache_hit == True:                                        │   │
│  │        self.model_call_details["response_cost"] = 0.0           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. _hidden_params["response_cost"] 存在:                       │   │
│  │    elif "response_cost" in hidden_params:                        │   │
│  │        self.model_call_details["response_cost"] =                 │   │
│  │            hidden_params["response_cost"]                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. model_call_details["response_cost"] 已存在:                   │   │
│  │    elif self.model_call_details.get("response_cost") is not None:│   │
│  │        pass  # 不覆盖，直接使用已有值                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 4. 本地重算:                                                    │   │
│  │    else:                                                         │   │
│  │        self.model_call_details["response_cost"] =                   │   │
│  │            self._response_cost_calculator(result=logging_result)  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 辅助落点

除了主要落点，还有一些辅助信息也会被设置：

| 辅助信息 | 设置位置 | 说明 |
|---------|---------|------|
| `cost_breakdown` | `set_cost_breakdown()` | 费用明细（输入、输出、缓存、折扣、利润等） |
| `standard_logging_object` | `_build_standard_logging_payload()` | 标准日志对象，包含完整的调用信息 |
| `response_cost_failure_debug_information` | 重算失败时 | 调试信息，用于排查费用计算失败 |

### 4.3 落点一致性验证

**所有分支的最终落点完全一致**：

```python
# 无论走哪条分支，最终都是设置：
self.model_call_details["response_cost"] = <费用值>

# 唯一的例外是分支 3：
elif self.model_call_details.get("response_cost") is not None:
    pass  # 不做任何操作，保留已有值
```

**一致性保证**:
- 所有成功计算的费用都会写入 `model_call_details["response_cost"]`
- 后续的回调、日志、持久化都会从这个位置读取
- 没有多个不同的"费用"字段可能导致不一致

### 4.4 特殊情况：流式响应

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1893-1894

```python
else:
    self.model_call_details["response_cost"] = None
```

**流式响应的特殊处理**:
- 在 `_success_handler_helper_fn` 中，如果是流式响应（`self.stream is True`），且没有走 `_process_hidden_params_and_response_cost`
- 此时 `response_cost` 会被设为 `None`
- 流式响应的费用计算通常在流结束时的回调中处理

---

## 五、关键代码位置速查

### 5.1 优先级判断相关

| 功能 | 文件位置 | 函数/方法名 |
|-----|---------|------------|
| 主优先级判断 | `litellm/litellm_core_utils/litellm_logging.py:1758-1769` | `_process_hidden_params_and_response_cost()` |
| 次优先级判断 | `litellm/litellm_core_utils/litellm_logging.py:1483-1499` | `_response_cost_calculator()` 内联 |
| 上游费用提取 | `litellm/cost_calculator.py:1668-1685` | `get_response_cost_from_hidden_params()` |
| 重算失败处理 | `litellm/litellm_core_utils/litellm_logging.py:1546-1585` | `_response_cost_calculator()` 异常处理 |

### 5.2 Provider 设置上游费用相关

| Provider | 文件位置 | 设置字段 |
|---------|---------|---------|
| **OpenRouter Chat** | `litellm/llms/openrouter/chat/transformation.py:227-229` | `additional_headers["llm_provider-x-litellm-response-cost"]` |
| **OpenRouter Image** | `litellm/llms/openrouter/image_*/transformation.py` | 同上 + `response_cost_details` |
| **Stability AI** | `litellm/llms/stability/image_edit/transformations.py:313-315` | `additional_headers["llm_provider-x-litellm-response-cost"]` |
| **Bedrock** | `litellm/llms/bedrock/image_edit/*.py` | 同上 |
| **Passthrough Handlers** | `litellm/proxy/pass_through_endpoints/llm_provider_handlers/*.py` | `model_call_details["response_cost"]` + `_hidden_params["response_cost"]` |

### 5.3 日志落点相关

| 功能 | 文件位置 | 说明 |
|-----|---------|------|
| 主要落点设置 | `litellm/litellm_core_utils/litellm_logging.py:1758-1769` | `model_call_details["response_cost"]` |
| 费用明细设置 | `litellm/litellm_core_utils/litellm_logging.py:1383-1449` | `set_cost_breakdown()` |
| 标准日志构建 | `litellm/litellm_core_utils/litellm_logging.py:1782-1797` | `_build_standard_logging_payload()` |

---

## 六、总结

### 6.1 优先级决策树

```
                    ┌─────────────────────┐
                    │   请求完成     │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
    ┌─────────────────┐     │     ┌─────────────────┐
    │ cache_hit=True? │     │     │ cache_hit=None│
    │                 │     │     │    或=False     │
    └────────┬────────┘     │     └────────┬────────┘
             │                │              │
             ▼                │              ▼
    ┌─────────────────┐     │     ┌─────────────────────────────┐
    │ response_cost=0 │     │     │_hidden_params[        │
    │ (最高优先级)   │     │     │ "response_cost"] 存在? │
    └─────────────────┘     │     └──────────┬──────────┘
                             │                │
                             │                ▼
                             │     ┌─────────────────────────────┐
                             │     │ 使用这个值                  │
                             │     │ response_cost = 该值         │
                             │     └─────────────────────────────┘
                             │                │
                             │                ▼
                             │     ┌─────────────────────────────┐
                             │     │ model_call_details[          │
                             │     │ "response_cost"] 已存在?   │
                             │     └──────────┬──────────────────┘
                             │                │
                             │                ▼
                             │     ┌─────────────────────────────┐
                             │     │ 保留已有值 (不覆盖)        │
                             │     └─────────────────────────────┘
                             │                │
                             │                ▼
                             │     ┌─────────────────────────────┐
                             │     │ 检查 _hidden_params[        │
                             │     │ "additional_headers"][       │
                             │     │ "llm_provider-x-litellm-   │
                             │     │ response-cost"] 存在?          │
                             │     └──────────┬──────────────────┘
                             │                │
                             │                ▼
                             │     ┌─────────────────────────────┐
                             │     │ 使用上游费用                    │
                             │     └─────────────────────────────┘
                             │                │
                             │                ▼
                             │     ┌─────────────────────────────┐
                             │     │ 本地重算                      │
                             │     │ completion_cost()           │
                             │     └─────────────────────────────┘
                             │
```

### 6.2 设计要点总结

1. **多级信任机制**:
   - 缓存命中最高优先级（0 费用）
   - 上游直接给费用次之（`_hidden_params["response_cost"]`、`model_call_details["response_cost"]`）
   - Provider 从上游提取的费用再次之（`additional_headers["llm_provider-x-litellm-response-cost"]`）
   - 本地重算为最后兜底

2. **上游费用来源**:
   - **真正的上游费用**: OpenRouter 从其 API 响应的 `usage.cost` 提取
   - **预计算费用**: Stability AI、Bedrock 等从 `model_info` 计算后设置
   - **Passthrough 模式**: Handler 调用 `completion_cost()` 后设置到 `_hidden_params`

3. **本地重算依赖**:
   - `usage` 对象（从响应提取）
   - `model_info`（从配置文件获取）
   - Provider 特定的计算逻辑

4. **日志落点一致性**:
   - 所有分支最终都写入 `model_call_details["response_cost"]`
   - 没有多个可能导致不一致的"费用"字段
   - 流式响应有特殊处理（设为 `None`，流结束时再计算）

5. **失败处理**:
   - 本地重算失败时返回 `None`
   - 详细的调试信息被记录到 `model_call_details["response_cost_failure_debug_information"]`
   - 不影响请求的正常返回
