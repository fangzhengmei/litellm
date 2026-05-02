# LiteLLM 费用追踪与 Token 计算机制分析

## 概述

LiteLLM 提供了一套完整的费用追踪和 token 计算机制，支持 100+ 种 LLM 提供商。本文档详细分析其价格信息来源、费用计算模块、以及日志与回调机制。

---

## 一、价格信息来源

### 1.1 主要来源：`model_prices_and_context_window.json`

这是 LiteLLM 最核心的价格配置文件，位于项目根目录：
- **路径**: `g:\fangzheng\solo-dogfeeding\code\17686-litellm\model_prices_and_context_window.json`
- **作用**: 存储所有支持模型的价格信息、上下文窗口限制、功能支持等

#### 价格字段结构（参考 `sample_spec`）

```json
{
  "input_cost_per_token": 0.0,           // 输入 token 单价
  "output_cost_per_token": 0.0,          // 输出 token 单价
  "output_cost_per_reasoning_token": 0.0, // 推理 token 单价（如 Claude 3.7 Sonnet）
  "cache_read_input_token_cost": 0.0,     // 缓存读取 token 单价（Prompt Caching）
  "cache_creation_input_token_cost": 0.0, // 缓存创建 token 单价
  "input_cost_per_audio_token": 0.0,      // 音频输入 token 单价
  "output_cost_per_image": 0.0,           // 图片生成单价（按张数）
  "input_cost_per_pixel": 0.0,            // 按像素计费（如 DALL-E）
  "litellm_provider": "openai",           // 提供商标识
  "mode": "chat",                          // 模型类型：chat/embedding/image_generation 等
  "max_input_tokens": 8191,               // 最大输入 token 数
  "max_output_tokens": 8191,              // 最大输出 token 数
  "supports_prompt_caching": true,        // 是否支持提示词缓存
  "supports_reasoning": true,              // 是否支持推理模式
  "tiered_pricing": [...]                  // 分层定价（如 Dashscope）
}
```

#### 定价策略多样性

不同提供商有不同的定价策略，体现在配置字段中：

| 定价模式 | 关键字段 | 示例提供商 |
|---------|---------|-----------|
| **标准 token 计费** | `input_cost_per_token`, `output_cost_per_token` | OpenAI, Anthropic |
| **推理 token 计费** | `output_cost_per_reasoning_token` | Anthropic Claude 3.7 |
| **Prompt Caching** | `cache_read_input_token_cost`, `cache_creation_input_token_cost` | Anthropic, Bedrock |
| **分层定价** | `tiered_pricing` | Dashscope（阿里灵积） |
| **按图片数计费** | `output_cost_per_image` | DALL-E, Stability AI |
| **按像素计费** | `input_cost_per_pixel` | DALL-E 2 |
| **按时间计费** | `custom_cost_per_second` | Replicate, 部分云服务 |
| **区域/速度溢价** | `provider_specific_entry` | Anthropic（geo/speed routing） |

### 1.2 自定义定价来源

除了内置的 `model_prices_and_context_window.json`，LiteLLM 还支持多种自定义定价方式：

#### 方式一：程序级别自定义

```python
import litellm
from litellm.utils import CostPerToken

# 方式1：直接传入 cost_per_token
response = litellm.completion(
    model="custom-model",
    messages=[{"role": "user", "content": "Hi"}],
    custom_cost_per_token={
        "input_cost_per_token": 0.00001,
        "output_cost_per_token": 0.00002
    }
)

# 方式2：使用 custom_pricing 配置
# 在 proxy 配置中定义
```

#### 方式二：环境变量配置

部分提供商支持通过环境变量覆盖定价，具体参考各 provider 实现。

#### 方式三：Proxy 配置文件

在 proxy 模式下，可以通过配置文件定义模型的定价策略，包括：
- 折扣配置（discount）
- 利润加成（margin）
- 自定义模型映射

### 1.3 价格信息读取接口

LiteLLM 提供了 `get_model_info` 函数来统一获取模型信息：

**核心函数**: `litellm.utils.get_model_info()`

```python
# 调用示例
model_info = litellm.get_model_info(
    model="gpt-4",
    custom_llm_provider="openai"
)
```

该函数会：
1. 首先检查 `litellm.model_cost` 字典（运行时加载的价格数据）
2. 支持自定义模型覆盖
3. 处理模型名称的别名和映射

---

## 二、Token 数量来源

### 2.1 主要来源：API 响应中的 `usage` 对象

绝大多数 LLM 提供商在 API 响应中返回 token 使用情况：

```python
# 示例：OpenAI 响应结构
{
  "usage": {
    "prompt_tokens": 10,           // 输入 token 数
    "completion_tokens": 20,       // 输出 token 数
    "total_tokens": 30,            // 总计
    "prompt_tokens_details": {      // OpenAI 缓存详情
      "cached_tokens": 5
    },
    "cache_creation_input_tokens": 100,  // Anthropic 缓存创建
    "cache_read_input_tokens": 50,        // Anthropic 缓存读取
    "reasoning_tokens": 15,        // 推理 token 数（部分模型）
  }
}
```

### 2.2 备选方案：LiteLLM 内部 Token 计数器

当 API 响应中没有 `usage` 对象时，LiteLLM 会使用内部计数器：

**核心函数**: `litellm.utils.token_counter()`

```python
# 调用路径（来自 cost_calculator.py:1231-1235）
if len(messages) > 0:
    prompt_tokens = token_counter(model=model, messages=messages)
elif len(prompt) > 0:
    prompt_tokens = token_counter(model=model, text=prompt)
completion_tokens = token_counter(model=model, text=completion)
```

**适用场景**:
- 流式响应部分实现（如果 provider 不在每个 chunk 中返回 usage）
- 自定义模型或未标准化的响应
- 预估算 token 数（如 `estimate_cost` 端点）

### 2.3 特殊场景的 Token 提取

#### 场景1：流式响应（Streaming）

对于流式响应，LiteLLM 有两种处理方式：
1. **Provider 流式 usage**: 部分 provider（如 Anthropic）在流结束时返回完整的 usage
2. **累计计数**: 其他情况通过每个 chunk 的 delta 累计计算

#### 场景2：多模态输入（Vision/Audio）

对于包含图片或音频的输入：
- **图片**: 通常按每张图片固定 token 数或按尺寸计算
- **音频**: 部分提供商有专门的 `input_cost_per_audio_token`

---

## 三、费用计算模块与流程

### 3.1 核心计算模块

| 模块 | 路径 | 职责 |
|-----|------|-----|
| **主计算器** | `litellm/cost_calculator.py` | 统一入口、路由分发、折扣/利润计算 |
| **通用计算器** | `litellm/litellm_core_utils/llm_cost_calc/utils.py` | `generic_cost_per_token` 等通用函数 |
| **Provider 特定计算器** | `litellm/llms/{provider}/cost_calculator.py` 或 `cost_calculation.py` | 各提供商特有定价逻辑 |

### 3.2 主入口函数：`completion_cost()`

**位置**: `litellm/cost_calculator.py:1015`

这是费用计算的统一入口，支持各种调用类型：

```python
def completion_cost(
    completion_response=None,           # API 响应对象
    model: Optional[str] = None,        # 模型名
    prompt="",                           # 备用：prompt 文本
    messages: List = [],                 # 备用：消息列表
    completion="",                       # 备用：补全文本
    total_time: Optional[float] = 0.0,  # 按时间计费时使用
    call_type: Optional[CallTypesLiteral] = None,  # 调用类型
    custom_llm_provider=None,            # 提供商标识
    custom_cost_per_token: Optional[CostPerToken] = None,  # 自定义定价
    custom_cost_per_second: Optional[float] = None,         # 自定义按秒定价
    litellm_logging_obj: Optional[LitellmLoggingObject] = None,  # 日志对象
    ...
) -> float:
```

### 3.3 计算流程详解

#### 阶段1：输入数据提取（Line 1083-1235）

```
┌─────────────────────────────────────────────────────────────┐
│                    输入数据提取阶段                            │
├─────────────────────────────────────────────────────────────┤
│  1. 从 completion_response 提取 usage 对象                   │
│     ├── prompt_tokens                                        │
│     ├── completion_tokens                                    │
│     ├── cache_creation_input_tokens (Prompt Caching)        │
│     ├── cache_read_input_tokens (Prompt Caching)            │
│     └── 特殊字段（如 reasoning_tokens）                      │
│                                                              │
│  2. 提取 model 和 custom_llm_provider                       │
│     ├── 从 _hidden_params 获取                               │
│     └── 从响应对象的其他属性获取                              │
│                                                              │
│  3. 备用方案：如果没有 usage，使用 token_counter 估算        │
└─────────────────────────────────────────────────────────────┘
```

**关键代码位置**: `cost_calculator.py:1083-1235`

#### 阶段2：调用类型路由（Line 1237-1422）

根据 `call_type` 选择不同的计算路径：

```python
# 不同调用类型的路由
if call_type in _A2A_CALL_TYPES:
    # A2A (Agent-to-Agent) 协议
    return A2ACostCalculator.calculate_a2a_cost(...)
    
elif CostCalculatorUtils._call_type_has_image_response(call_type):
    # 图片生成
    return CostCalculatorUtils.route_image_generation_cost_calculator(...)
    
elif call_type in _VIDEO_CALL_TYPES:
    # 视频生成
    return video_generation_cost(...)  # 或 default_video_cost_calculator
    
elif call_type in _SPEECH_CALL_TYPES:
    # 语音合成（TTS）- 按字符数
    prompt_characters = litellm.utils._count_characters(text=prompt)
    
elif call_type in _TRANSCRIPTION_CALL_TYPES:
    # 语音转录 - 按时长
    audio_transcription_file_duration = ...
    
elif call_type in _RERANK_CALL_TYPES:
    # Rerank - 按 search_units 或 total_tokens
    
elif call_type in _SEARCH_CALL_TYPES:
    # 搜索 - 按查询次数
    (prompt_cost, completion_cost_result) = search_provider_cost_per_query(...)
    
elif call_type == _AREALTIME_CALL_TYPE:
    # 实时流
    return handle_realtime_stream_cost_calculation(...)
    
elif call_type == _MCP_CALL_TYPE:
    # MCP 工具调用
    return MCPCostCalculator.calculate_mcp_tool_call_cost(...)
```

**关键代码位置**: `cost_calculator.py:1237-1422`

#### 阶段3：Provider 特定计算（标准文本生成场景）

对于标准的文本补全/聊天调用，流程如下：

```
┌─────────────────────────────────────────────────────────────┐
│              Provider 特定计算流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  1. 检查自定义定价（优先级最高）                               │
│     └── custom_cost_per_token / custom_cost_per_second     │
│                    ↓                                         │
│  2. 模型名称标准化                                            │
│     ├── together_ai: 根据模型大小选择价格档位                │
│     ├── databricks: 模型名映射到基础模型                     │
│     ├── vertex_ai: 处理 128k+ 分层定价                      │
│     └── 其他: 直接使用模型名                                  │
│                    ↓                                         │
│  3. 路由到 provider 特定的 cost_per_token 函数               │
│                    ↓                                         │
│  4. 应用折扣和利润加成                                        │
│                    ↓                                         │
│  5. 存储费用明细到日志对象                                    │
└─────────────────────────────────────────────────────────────┘
```

### 3.4 Provider 特定计算器实现

#### 分类体系

LiteLLM 的 provider 计算器分为两类：

| 类型 | 实现方式 | 适用 Provider |
|-----|---------|--------------|
| **通用实现** | 直接调用 `generic_cost_per_token()` | OpenAI, Bedrock, Amazon Nova, Gemini |
| **自定义实现** | 有独立的计算逻辑 | Anthropic, Dashscope, Databricks, Fireworks AI, Vertex AI |

#### 通用实现示例（OpenAI）

**位置**: `litellm/llms/openai/cost_calculation.py:34`

```python
def cost_per_token(
    model: str, usage: Usage, service_tier: Optional[str] = None
) -> Tuple[float, float]:
    return generic_cost_per_token(
        model=model,
        usage=usage,
        custom_llm_provider="openai",
        service_tier=service_tier,
    )
```

#### 自定义实现示例（Anthropic - 复杂定价）

**位置**: `litellm/llms/anthropic/cost_calculation.py:56`

```python
def cost_per_token(model: str, usage: "Usage") -> Tuple[float, float]:
    # 1. 基础计算：使用 generic_cost_per_token
    prompt_cost, completion_cost = generic_cost_per_token(
        model=model, usage=usage, custom_llm_provider="anthropic"
    )
    
    # 2. 应用区域/速度溢价（provider_specific_entry 中的乘数）
    try:
        model_info = litellm.get_model_info(...)
        provider_specific_entry: dict = model_info.get("provider_specific_entry") or {}
        
        multiplier = 1.0
        # 区域溢价：如 us-east-5 可能有不同定价
        if hasattr(usage, "inference_geo") and ...:
            multiplier *= provider_specific_entry.get(usage.inference_geo.lower(), 1.0)
        # 速度溢价：fast 模式可能更贵
        if hasattr(usage, "speed") and usage.speed == "fast":
            multiplier *= provider_specific_entry.get("fast", 1.0)
        
        if multiplier != 1.0:
            # 缓存费用不参与溢价（需要单独计算）
            cache_cost = _compute_cache_only_cost(...)
            prompt_cost = (prompt_cost - cache_cost) * multiplier + cache_cost
            completion_cost *= multiplier
    except Exception:
        pass
    
    return prompt_cost, completion_cost
```

**Anthropic 特有功能**:
- **Prompt Caching 支持**: 区分 cache_read 和 cache_creation token
- **区域路由定价**: 不同 inference_geo 有不同乘数
- **速度路由定价**: `fast` 模式可能有溢价

#### 自定义实现示例（Dashscope - 分层定价）

**位置**: `litellm/llms/dashscope/cost_calculator.py:187`

```python
def cost_per_token(model: str, usage: Usage) -> Tuple[float, float]:
    model_info = get_model_info(model=model, custom_llm_provider="dashscope")
    breakdown = _extract_token_breakdown(usage)
    
    # 检查是否有分层定价
    tiered_pricing = model_info.get("tiered_pricing")  # list 类型
    
    # 按分层计算输入费用
    prompt_cost = _calculate_prompt_cost(
        breakdown=breakdown, 
        model_info=model_info, 
        tiered_pricing=tiered_pricing
    )
    
    # 按分层计算输出费用
    completion_cost = _calculate_completion_cost(...)
    
    return prompt_cost, completion_cost
```

**Dashscope 特有功能**:
- **分层定价**: 根据 token 数量落在不同区间应用不同单价
- **缓存 token 支持**: 支持 cached_tokens 和 reasoning_tokens

### 3.5 通用计算核心：`generic_cost_per_token()`

**位置**: `litellm/litellm_core_utils/llm_cost_calc/utils.py`

这是最常用的计算函数，处理：

```
计算维度：
├── 标准输入 token: input_cost_per_token × prompt_tokens
├── 标准输出 token: output_cost_per_token × completion_tokens
├── 推理 token: output_cost_per_reasoning_token × reasoning_tokens
├── 缓存读取: cache_read_input_token_cost × cache_read_input_tokens
├── 缓存创建: cache_creation_input_token_cost × cache_creation_input_tokens
├── 服务层级差异: _get_service_tier_cost_key() 处理 priority/flex 等
└── 128k+ 分层定价（部分模型）: 超过 128k 后单价变化
```

### 3.6 折扣与利润计算

#### 折扣应用

**位置**: `litellm/cost_calculator.py:772`

```python
def _apply_cost_discount(
    base_cost: float,
    custom_llm_provider: Optional[str] = None,
) -> Tuple[float, Optional[float], Optional[float]]:
    """
    应用折扣
    
    返回: (final_cost, discount_percent, discount_amount)
    """
    # 从 litellm.discounts 配置中读取
    # 支持按 custom_llm_provider 或全局设置
```

#### 利润加成

**位置**: `litellm/cost_calculator.py:863`

```python
def _apply_cost_margin(
    base_cost: float,
    custom_llm_provider: Optional[str] = None,
) -> Tuple[float, Optional[float], Optional[float], Optional[float]]:
    """
    应用利润加成
    
    支持:
    - margin_percentage: 百分比加成
    - margin_fixed_amount: 固定金额加成
    """
```

---

## 四、费用存储、日志与回调机制

### 4.1 费用明细数据结构

费用计算完成后，明细会存储在 `CostBreakdown` 对象中：

**位置**: 通过 `set_cost_breakdown()` 方法存储

```python
# cost_calculator.py:993 调用
litellm_logging_obj.set_cost_breakdown(
    input_cost=prompt_tokens_cost_usd_dollar,           # 输入费用
    output_cost=completion_tokens_cost_usd_dollar,       # 输出费用
    total_cost=total_cost_usd_dollar,                     # 总费用
    cost_for_built_in_tools_cost_usd_dollar=...,         # 内置工具费用
    additional_costs=additional_costs,                    # 额外费用（如 Azure router flat cost）
    original_cost=original_cost,                          # 折扣前费用
    discount_percent=discount_percent,                    # 折扣百分比
    discount_amount=discount_amount,                      # 折扣金额
    margin_percent=margin_percent,                        # 利润百分比
    margin_fixed_amount=margin_fixed_amount,              # 固定利润
    margin_total_amount=margin_total_amount,              # 总利润
    cache_read_cost=cache_read_cost,                      # 缓存读取费用
    cache_creation_cost=cache_creation_cost,              # 缓存创建费用
)
```

### 4.2 存储到日志对象

#### `set_cost_breakdown()` 方法

**位置**: `litellm/litellm_core_utils/litellm_logging.py:1383`

```python
def set_cost_breakdown(
    self,
    input_cost: float,
    output_cost: float,
    total_cost: float,
    cost_for_built_in_tools_cost_usd_dollar: float,
    additional_costs: Optional[dict] = None,
    original_cost: Optional[float] = None,
    discount_percent: Optional[float] = None,
    discount_amount: Optional[float] = None,
    margin_percent: Optional[float] = None,
    margin_fixed_amount: Optional[float] = None,
    margin_total_amount: Optional[float] = None,
    cache_read_cost: Optional[float] = None,
    cache_creation_cost: Optional[float] = None,
) -> None:
    """
    将费用明细存储到 logging 对象的 cost_breakdown 属性
    """
    self.cost_breakdown = CostBreakdown(
        input_cost=input_cost,
        output_cost=output_cost,
        total_cost=total_cost,
        tool_usage_cost=cost_for_built_in_tools_cost_usd_dollar,
    )
    
    # 可选字段：只有有值时才存储
    if cache_read_cost is not None and cache_read_cost > 0:
        self.cost_breakdown["cache_read_cost"] = cache_read_cost
    if cache_creation_cost is not None and cache_creation_cost > 0:
        self.cost_breakdown["cache_creation_cost"] = cache_creation_cost
    # ... 其他字段类似
```

### 4.3 日志与回调流程

#### 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      LiteLLM 日志与回调架构                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  ┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐   │
│  │ API 请求完成  │────▶│ Logging 对象创建  │────▶│ 费用计算完成      │   │
│  └──────────────┘     └──────────────────┘     └──────────────────┘   │
│                                                      │                    │
│                                                      ▼                    │
│                                            ┌──────────────────┐          │
│                                            │ set_cost_breakdown│          │
│                                            │ (存储费用明细)     │          │
│                                            └──────────────────┘          │
│                                                      │                    │
│                                                      ▼                    │
│                                            ┌──────────────────┐          │
│                                            │ success_handler   │          │
│                                            │ 或 failure_handler│          │
│                                            └──────────────────┘          │
│                                                      │                    │
│                              ┌───────────────────────┼──────────────────┐│
│                              ▼                       ▼                  ▼│
│                     ┌──────────────┐        ┌──────────────┐   ┌──────┐│
│                     │ 成功回调      │        │ 失败回调      │   │ 日志 ││
│                     │ (success)    │        │ (failure)    │   │ 集成 ││
│                     │              │        │              │   │      ││
│                     │ - Langfuse   │        │ - 相同集成    │   │- Stdout││
│                     │ - LangSmith  │        │   但带错误信息│   │- File ││
│                     │ - Helicone   │        │              │   │- OTEL ││
│                     │ - Prometheus │        │              │   │- 等   ││
│                     │ - 自定义回调  │        │              │   │      ││
│                     └──────────────┘        └──────────────┘   └──────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

#### 关键方法：`success_handler()`

**位置**: `litellm/litellm_core_utils/litellm_logging.py:2007`

这是请求成功后的统一处理入口，包含：

```
success_handler 执行流程：
1. 计算响应时间（response_ms）
2. 提取响应中的 usage 和 model 信息
3. 调用 _response_cost_calculator() 计算费用
   └── 内部调用 completion_cost()
4. 准备回调数据（model_call_details）
5. 分发到各个日志集成：
   ├── async_success_handler() 异步回调
   ├── 各 observability 集成（Langfuse, LangSmith, Helicone 等）
   └── 自定义 success_callback
```

#### 关键方法：`failure_handler()`

**位置**: `litellm/litellm_core_utils/litellm_logging.py:2928`

失败请求的处理入口：

```
failure_handler 执行流程：
1. 记录错误信息（exception, traceback）
2. 计算响应时间
3. 准备失败回调数据
4. 分发到：
   ├── async_failure_handler() 异步回调
   ├── 各 observability 集成
   └── 自定义 failure_callback
```

### 4.4 回调管理器：`LoggingCallbackManager`

**位置**: `litellm/litellm_core_utils/logging_callback_manager.py`

LiteLLM 使用回调管理器来管理多个日志集成：

```python
class LoggingCallbackManager:
    """
    管理所有已注册的回调处理器
    
    支持的集成类型：
    - langfuse
    - langsmith
    - helicone
    - lunary
    - athina
    - traceloop
    - opentelemetry
    - promptlayer
    - sagemaker
    - 自定义 custom_logger
    """
    
    # 注册回调
    def register_callback(self, callback_type: str, callback_instance: Any)
    
    # 触发成功回调
    def run_success_callbacks(self, kwargs)
    
    # 触发失败回调
    def run_failure_callbacks(self, kwargs)
```

### 4.5 响应中嵌入费用信息

费用计算完成后，除了存储到日志对象，还会嵌入到响应对象的 `_hidden_params` 中：

**位置**: `cost_calculator.py` 多处使用

```python
# 示例：passthrough 处理器中的处理
# litellm/proxy/pass_through_endpoints/llm_provider_handlers/cohere_passthrough_logging_handler.py:111-118

# 计算费用
response_cost = litellm.completion_cost(...)

# 存储到 _hidden_params 避免重复计算
if not hasattr(litellm_model_response, "_hidden_params"):
    litellm_model_response._hidden_params = {}
litellm_model_response._hidden_params["response_cost"] = response_cost
```

这样做的好处：
- 避免重复计算费用
- 让响应对象携带完整的费用信息
- 便于后续的日志和回调使用

---

## 五、关键文件索引

### 5.1 核心计算模块

| 文件路径 | 主要职责 |
|---------|---------|
| `litellm/cost_calculator.py` | 费用计算主入口、路由分发、折扣/利润计算 |
| `litellm/litellm_core_utils/llm_cost_calc/utils.py` | 通用计算函数（`generic_cost_per_token` 等） |
| `litellm/litellm_core_utils/llm_cost_calc/usage_object_transformation.py` | Usage 对象转换 |
| `litellm/litellm_core_utils/llm_cost_calc/tool_call_cost_tracking.py` | 工具调用费用追踪 |

### 5.2 Provider 特定计算器

| Provider | 文件路径 |
|---------|---------|
| Anthropic | `litellm/llms/anthropic/cost_calculation.py` |
| OpenAI | `litellm/llms/openai/cost_calculation.py` |
| Azure | `litellm/llms/azure/cost_calculation.py` |
| Azure AI | `litellm/llms/azure_ai/cost_calculator.py` |
| Bedrock | `litellm/llms/bedrock/cost_calculation.py` |
| Gemini | `litellm/llms/gemini/cost_calculator.py` |
| Vertex AI | `litellm/llms/vertex_ai/cost_calculator.py` |
| Dashscope | `litellm/llms/dashscope/cost_calculator.py` |
| Databricks | `litellm/llms/databricks/cost_calculator.py` |
| Fireworks AI | `litellm/llms/fireworks_ai/cost_calculator.py` |
| Perplexity | `litellm/llms/perplexity/cost_calculator.py` |
| Together AI | `litellm/llms/together_ai/cost_calculator.py` |
| XAI | `litellm/llms/xai/cost_calculator.py` |
| Amazon Nova | `litellm/llms/amazon_nova/cost_calculation.py` |

### 5.3 日志与回调模块

| 文件路径 | 主要职责 |
|---------|---------|
| `litellm/litellm_core_utils/litellm_logging.py` | Logging 类实现、success/failure handler |
| `litellm/litellm_core_utils/logging_callback_manager.py` | 回调管理器 |
| `litellm/litellm_core_utils/logging_worker.py` | 异步日志工作线程 |
| `litellm/types/integrations/` | 各集成类型定义 |

### 5.4 配置与工具

| 文件路径 | 主要职责 |
|---------|---------|
| `model_prices_and_context_window.json` | 所有模型的价格和配置信息 |
| `litellm/utils.py` | `get_model_info`, `token_counter` 等工具函数 |
| `litellm/proxy/management_endpoints/cost_tracking_settings.py` | Proxy 的费用追踪端点（/cost/estimate 等） |

---

## 六、数据流总结

### 6.1 完整费用追踪数据流

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    LiteLLM 费用追踪完整数据流                                   │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                │
│  ┌──────────────────┐                                                          │
│  │ 1. 价格信息加载   │  启动时加载                                              │
│  │                  │                                                          │
│  │ model_prices_    │ ──────▶ litellm.model_cost (内存字典)                  │
│  │ and_context_     │          └── 支持运行时覆盖                              │
│  │ window.json      │                                                          │
│  └──────────────────┘                                                          │
│                                                                                │
│                         ┌──────────────────────────────────────────────┐      │
│                         │              2. 单次请求处理                   │      │
│                         ├──────────────────────────────────────────────┤      │
│                                                                                │
│  ┌──────────────────┐                                                          │
│  │ API 请求发起     │                                                          │
│  │ (completion/    │                                                          │
│  │  embedding 等)  │                                                          │
│  └────────┬─────────┘                                                          │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────┐                                                          │
│  │ 3. Logging 对象  │ 创建                                                      │
│  │    初始化        │                                                          │
│  └────────┬─────────┘                                                          │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────┐                                                          │
│  │ 4. 发送到 Provider│ ──────▶ 外部 LLM API                                    │
│  └────────┬─────────┘                                                          │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────┐                                                          │
│  │ 5. 接收 API 响应  │                                                          │
│  │    (含 usage)    │                                                          │
│  └────────┬─────────┘                                                          │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ 6. 费用计算 (completion_cost)                                          │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  6.1 提取 usage 对象                                                   │    │
│  │      ├── prompt_tokens                                                 │    │
│  │      ├── completion_tokens                                             │    │
│  │      ├── cache_read_input_tokens                                       │    │
│  │      ├── cache_creation_input_tokens                                   │    │
│  │      └── reasoning_tokens (如适用)                                     │    │
│  │                                                                         │    │
│  │  6.2 路由到对应计算器                                                   │    │
│  │      ├── 文本生成 → provider cost_per_token                           │    │
│  │      ├── 图片生成 → image_generation_cost                             │    │
│  │      ├── 视频生成 → video_generation_cost                             │    │
│  │      ├── 语音 → 按字符/时长                                             │    │
│  │      └── 其他 → 特定类型计算器                                          │    │
│  │                                                                         │    │
│  │  6.3 应用折扣和利润                                                     │    │
│  │      ├── _apply_cost_discount()                                        │    │
│  │      └── _apply_cost_margin()                                          │    │
│  │                                                                         │    │
│  │  6.4 存储明细                                                           │    │
│  │      └── _store_cost_breakdown_in_logging_obj()                       │    │
│  │                                                                         │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│           │                                                                    │
│           ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐    │
│  │ 7. 日志与回调 (success_handler / failure_handler)                     │    │
│  ├──────────────────────────────────────────────────────────────────────┤    │
│  │                                                                         │    │
│  │  7.1 填充 model_call_details                                           │    │
│  │      ├── response_cost                                                  │    │
│  │      ├── cost_breakdown (明细)                                         │    │
│  │      ├── usage                                                          │    │
│  │      ├── model                                                          │    │
│  │      └── 其他请求/响应元数据                                             │    │
│  │                                                                         │    │
│  │  7.2 分发到各集成                                                       │    │
│  │      ├── Langfuse                                                       │    │
│  │      ├── LangSmith                                                      │    │
│  │      ├── Helicone                                                       │    │
│  │      ├── OpenTelemetry                                                  │    │
│  │      ├── Prometheus                                                     │    │
│  │      └── 自定义 success_callback / failure_callback                    │    │
│  │                                                                         │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                                │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 6.2 数据对账机制

LiteLLM 采用"多来源验证"的设计，但主要信任：

| 数据项 | 优先来源 | 备选来源 |
|-------|---------|---------|
| **Token 数量** | Provider API 返回的 `usage` 对象 | `token_counter()` 函数估算 |
| **单价** | `model_prices_and_context_window.json` | 自定义 `custom_cost_per_token` |
| **总费用** | 计算结果 `response_cost` | 无（计算是单一可信源） |

**对账点**：
1. **响应头 vs usage 对象**: 部分 provider 在响应头中也返回 token 信息，LiteLLM 会优先信任响应体中的 `usage`
2. **流式累计 vs 最终 usage**: 对于流式响应，最终的 `usage` 对象是可信源，中间累计仅供参考
3. **日志对象 vs 响应 _hidden_params**: 费用信息会同时存储在两处，确保一致性

---

## 七、扩展与定制点

### 7.1 自定义定价

```python
# 方式1：每次调用时传入
from litellm.utils import CostPerToken

response = litellm.completion(
    model="gpt-3.5-turbo",
    messages=[{"role": "user", "content": "Hi"}],
    custom_cost_per_token=CostPerToken(
        input_cost_per_token=0.000001,
        output_cost_per_token=0.000002
    )
)

# 方式2：通过环境变量或配置（Proxy 模式）
# 参考 proxy 配置文档
```

### 7.2 自定义回调

```python
# 定义自定义回调
def my_success_callback(
    kwargs,                 # 完整的调用参数
    completion_response,    # API 响应
    start_time,             # 开始时间
    end_time                # 结束时间
):
    # kwargs 中包含:
    # - model_call_details: 完整的请求/响应详情
    # - response_cost: 计算好的费用
    # - cost_breakdown: 费用明细
    print(f"请求费用: ${kwargs.get('response_cost', 0):.6f}")
    print(f"费用明细: {kwargs.get('model_call_details', {}).get('cost_breakdown')}")

# 注册回调
litellm.success_callback = [my_success_callback]
litellm.failure_callback = [my_failure_callback]  # 失败回调
```

### 7.3 添加新 Provider 的费用计算

如需添加新 provider 的支持，需：

1. **在 `model_prices_and_context_window.json` 中添加模型配置**
2. **（可选）创建 provider 特定的 cost_calculator.py**
   - 如果定价逻辑简单，直接继承通用实现
   - 如果有特殊定价（如分层、缓存、区域溢价），实现自定义 `cost_per_token`

---

## 八、关键代码位置速查

| 功能 | 文件位置 | 函数/方法名 |
|-----|---------|------------|
| 费用计算主入口 | `litellm/cost_calculator.py:1015` | `completion_cost()` |
| 存储费用明细 | `litellm/cost_calculator.py:955` | `_store_cost_breakdown_in_logging_obj()` |
| 应用折扣 | `litellm/cost_calculator.py:772` | `_apply_cost_discount()` |
| 应用利润 | `litellm/cost_calculator.py:863` | `_apply_cost_margin()` |
| 通用 token 计算 | `litellm/litellm_core_utils/llm_cost_calc/utils.py` | `generic_cost_per_token()` |
| 设置费用明细 | `litellm/litellm_core_utils/litellm_logging.py:1383` | `set_cost_breakdown()` |
| 成功处理入口 | `litellm/litellm_core_utils/litellm_logging.py:2007` | `success_handler()` |
| 失败处理入口 | `litellm/litellm_core_utils/litellm_logging.py:2928` | `failure_handler()` |
| 费用估算端点 | `litellm/proxy/management_endpoints/cost_tracking_settings.py:434` | `estimate_cost()` |

---

## 附录：术语表

| 术语 | 说明 |
|-----|------|
| **Prompt Caching** | 提示词缓存，Anthropic/Bedrock 等提供商支持的功能，缓存的 prompt token 费用更低 |
| **Reasoning Tokens** | 推理 token，部分模型（如 Claude 3.7 Sonnet）用于内部推理过程的 token，可能单独定价 |
| **Cost Breakdown** | 费用明细，包含输入、输出、缓存、折扣、利润等详细费用构成 |
| **Custom LLM Provider** | 自定义提供商标识，用于区分同一模型通过不同提供商访问的情况 |
| **Service Tier** | 服务层级，如 OpenAI 的 `priority`/`flex`，不同层级可能有不同定价 |
| **Tiered Pricing** | 分层定价，根据 token 使用量落在不同区间应用不同单价 |
| **Margin** | 利润加成，Proxy 模式下在基础费用上增加的利润 |
| **Discount** | 折扣，在基础费用上应用的减免 |
