# LiteLLM Provider Transformation 层设计分析

## 1. 整体架构

LiteLLM 的 provider transformation 层采用基于抽象基类的设计模式，实现了统一的 OpenAI 格式到各 provider 私有格式的双向转换。

### 1.1 核心类层次结构

```
BaseConfig (抽象基类)
├── 定义核心转换接口
├── get_supported_openai_params()
├── map_openai_params()
├── transform_request()
├── transform_response()
└── validate_environment()

BaseLLMModelInfo (模型信息抽象类)
├── get_models()
├── get_api_key()
├── get_api_base()
└── get_token_counter()

Provider 具体实现 (多重继承)
├── OpenAIGPTConfig(BaseLLMModelInfo, BaseConfig)
├── AnthropicConfig(AnthropicModelInfo, BaseConfig)
├── CohereChatConfig(BaseConfig)
├── GoogleAIStudioGeminiConfig(VertexGeminiConfig)
└── ... (100+ providers)
```

### 1.2 目录结构

```
litellm/llms/
├── base_llm/
│   └── chat/
│       └── transformation.py    # BaseConfig 抽象基类
├── base.py                       # BaseLLM 基础类
├── openai/
│   └── chat/
│       ├── gpt_transformation.py # OpenAI GPT 实现
│       └── o_series_transformation.py # O1/O3 系列
├── anthropic/
│   └── chat/
│       └── transformation.py     # Anthropic 实现
├── cohere/
│   └── chat/
│       └── transformation.py     # Cohere 实现
├── gemini/
│   └── chat/
│       └── transformation.py     # Gemini 实现
└── ... (其他 100+ providers)

litellm/types/llms/
├── openai.py                     # OpenAI 标准类型定义
├── anthropic.py                  # Anthropic 特定类型
├── cohere.py                     # Cohere 特定类型
└── ...
```

## 2. 核心转换接口

每个 provider 必须实现以下核心方法，定义在 `litellm/llms/base_llm/chat/transformation.py` 中的 `BaseConfig` 抽象类。

### 2.1 get_supported_openai_params(model: str) -> List[str]

**功能**：返回该 provider 支持的 OpenAI 参数列表，用于参数兼容矩阵检查。

**实现示例**：

```python
# OpenAI - 支持所有标准参数
def get_supported_openai_params(self, model: str) -> list:
    base_params = [
        "frequency_penalty", "logit_bias", "logprobs", "top_logprobs",
        "max_tokens", "max_completion_tokens", "modalities", "prediction",
        "n", "presence_penalty", "seed", "stop", "stream", "stream_options",
        "temperature", "top_p", "tools", "tool_choice", "function_call",
        "functions", "max_retries", "extra_headers", "parallel_tool_calls",
        "audio", "web_search_options", "service_tier", "safety_identifier",
        "prompt_cache_key", "prompt_cache_retention", "store",
    ]
    # 模型特定参数
    if model != "gpt-3.5-turbo-16k" and model != "gpt-4":
        model_specific_params.append("response_format")
    return base_params + model_specific_params

# Anthropic - 支持部分参数
def get_supported_openai_params(self, model: str):
    params = [
        "stream", "stop", "temperature", "top_p", "max_tokens",
        "max_completion_tokens", "tools", "tool_choice", "extra_headers",
        "parallel_tool_calls", "response_format", "user", "web_search_options",
        "speed", "context_management", "cache_control",
    ]
    # 推理模型额外支持
    if supports_reasoning(model=model, custom_llm_provider=self.custom_llm_provider):
        params.append("thinking")
        params.append("reasoning_effort")
    return params

# Cohere - 支持的参数
def get_supported_openai_params(self, model: str) -> List[str]:
    return [
        "stream", "temperature", "max_tokens", "max_completion_tokens",
        "top_p", "frequency_penalty", "presence_penalty", "stop", "n",
        "tools", "tool_choice", "seed", "extra_headers",
    ]
```

### 2.2 map_openai_params(non_default_params, optional_params, model, drop_params) -> dict

**功能**：将 OpenAI 格式的参数映射到 provider 特定格式。

**核心流程**：
1. 遍历用户传入的 `non_default_params`
2. 对每个参数进行 provider 特定的转换
3. 将转换后的参数添加到 `optional_params`

**实现示例 - Anthropic**：

```python
def map_openai_params(self, non_default_params: dict, optional_params: dict, 
                       model: str, drop_params: bool) -> dict:
    for param, value in non_default_params.items():
        if param == "max_tokens":
            optional_params["max_tokens"] = value
        elif param == "max_completion_tokens":
            optional_params["max_tokens"] = value  # 同名映射
        elif param == "stop":
            # OpenAI stop (str/list) → Anthropic stop_sequences
            _value = self._map_stop_sequences(value)
            if _value is not None:
                optional_params["stop_sequences"] = _value
        elif param == "tools":
            # OpenAI tools → Anthropic tools 格式转换
            anthropic_tools, mcp_servers = self._map_tools(value)
            optional_params = self._add_tools_to_optional_params(
                optional_params=optional_params, tools=anthropic_tools
            )
        elif param == "tool_choice" or param == "parallel_tool_calls":
            # OpenAI tool_choice + parallel_tool_calls → Anthropic tool_choice
            _tool_choice = self._map_tool_choice(
                tool_choice=non_default_params.get("tool_choice"),
                parallel_tool_use=non_default_params.get("parallel_tool_calls"),
            )
            if _tool_choice is not None:
                optional_params["tool_choice"] = _tool_choice
        elif param == "response_format":
            # OpenAI response_format → Anthropic output_format 或 tool
            if model_supports_output_format(model):
                _output_format = self.map_response_format_to_anthropic_output_format(value)
                optional_params["output_format"] = _output_format
            else:
                # 旧模型：转换为 tool 调用
                _tool = self.map_response_format_to_anthropic_tool(value, ...)
                optional_params = self._add_tools_to_optional_params(
                    optional_params=optional_params, tools=[_tool]
                )
        elif param == "reasoning_effort":
            # OpenAI reasoning_effort → Anthropic thinking
            optional_params["thinking"] = AnthropicConfig._map_reasoning_effort(
                reasoning_effort=value, model=model
            )
        elif param == "web_search_options":
            # OpenAI web_search_options → Anthropic web_search hosted tool
            hosted_web_search_tool = self.map_web_search_tool(cast(OpenAIWebSearchOptions, value))
            self._add_tools_to_optional_params(
                optional_params=optional_params, tools=[hosted_web_search_tool]
            )
    return optional_params
```

### 2.3 transform_request(model, messages, optional_params, litellm_params, headers) -> dict

**功能**：转换整个请求体（包括 messages 格式转换）。

### 2.4 transform_response(...) -> ModelResponse

**功能**：将 provider 的响应转换为统一的 OpenAI 格式。

## 3. 参数兼容矩阵的维护

LiteLLM 使用**双层机制**维护参数兼容矩阵：静态声明 + 动态检查。

### 3.1 静态参数支持声明

每个 provider 通过 `get_supported_openai_params()` 方法静态声明支持的参数。

### 3.2 动态模型能力检查

对于模型特定的能力（如推理、视觉、函数调用等），LiteLLM 使用动态检查机制。

#### 3.2.1 配置文件：model_prices_and_context_window.json

每个模型在 `model_prices_and_context_window.json` 中有能力标志：

```json
{
  "claude-3-7-sonnet-20250219": {
    "litellm_provider": "anthropic",
    "max_input_tokens": 200000,
    "max_output_tokens": 8192,
    "mode": "chat",
    "supports_function_calling": true,
    "supports_parallel_function_calling": true,
    "supports_prompt_caching": true,
    "supports_reasoning": true,
    "supports_response_schema": true,
    "supports_system_messages": true,
    "supports_vision": true,
    "supports_web_search": true
  }
}
```

#### 3.2.2 辅助函数：_supports_factory

通用能力检查工厂函数，定义在 `litellm/utils.py`：

```python
def _supports_factory(model: str, custom_llm_provider: Optional[str], key: str) -> bool:
    """检查模型是否支持指定能力"""
    try:
        # 1. 解析模型和 provider
        model, custom_llm_provider, _, _ = litellm.get_llm_provider(
            model=model, custom_llm_provider=custom_llm_provider
        )
        
        # 2. 从 model_prices_and_context_window.json 获取模型信息
        model_info = _get_model_info_helper(
            model=model, custom_llm_provider=custom_llm_provider
        )
        
        # 3. 检查能力标志
        if model_info.get(key, False) is True:
            return True
    except Exception:
        return False
    return False
```

## 4. 不同 Provider 的字段映射和差异处理

### 4.1 OpenAI (基准格式)

**特点**：OpenAI 格式是 LiteLLM 的统一标准格式，几乎不需要转换。

### 4.2 Anthropic (复杂转换)

**主要差异**：

| OpenAI 参数 | Anthropic 参数 | 转换逻辑 |
|-------------|----------------|---------|
| `max_tokens` | `max_tokens` | 直接映射 |
| `stop` (str/list) | `stop_sequences` | 列表包装 |
| `user` | `metadata.user_id` | 包装为 `{"user_id": value}` |
| `tools` | `tools` | 格式转换：`function.parameters` → `input_schema` |
| `parallel_tool_calls` | `tool_choice.disable_parallel_tool_use` | **反向映射** |
| `response_format` | `output_format` 或 `tools` | 模型版本依赖 |
| `reasoning_effort` | `thinking` | 转换为 `{"type": "enabled", "budget_tokens": N}` |

**关键转换代码**：

```python
# Tool Choice 转换
def _map_tool_choice(self, tool_choice: Optional[str], 
                     parallel_tool_use: Optional[bool]) -> Optional[AnthropicMessagesToolChoice]:
    _tool_choice: Optional[AnthropicMessagesToolChoice] = None
    
    if tool_choice == "auto":
        _tool_choice = {"type": "auto"}
    elif tool_choice == "required":
        _tool_choice = {"type": "any"}  # OpenAI "required" → Anthropic "any"
    elif tool_choice == "none":
        _tool_choice = {"type": "none"}
    
    # 处理 parallel_tool_calls - 反向映射
    # OpenAI: parallel_tool_calls=True 表示启用并行
    # Anthropic: disable_parallel_tool_use=False 表示启用并行
    if parallel_tool_use is not None:
        if _tool_choice is not None:
            _tool_choice["disable_parallel_tool_use"] = not parallel_tool_use
    return _tool_choice
```

### 4.3 Cohere

**主要差异**：

| OpenAI 参数 | Cohere 参数 | 转换逻辑 |
|-------------|-------------|---------|
| `top_p` | `p` | **重命名** |
| `n` | `num_generations` | **重命名** |
| `stop` | `stop_sequences` | 直接映射 |
| `tools` | `tools` | 格式转换：`function.parameters` → `parameter_definitions` |

**消息格式差异**：
- OpenAI: 统一的 `messages` 列表
- Cohere: `chat_history` + `message` 分离格式

### 4.4 Gemini

**主要特点**：
- 复用 Vertex AI 的转换逻辑
- 需要将 HTTP/HTTPS 图片 URL 转换为 base64（Gemini 不支持直接的 URL 输入）
- 支持动态能力检查

## 5. 类型定义系统

LiteLLM 使用 Python 的 `TypedDict` 和 `Union` 类型定义了完整的类型系统。

### 5.1 OpenAI 标准类型

**文件**：`litellm/types/llms/openai.py`

### 5.2 Anthropic 特定类型

**文件**：`litellm/types/llms/anthropic.py`

## 6. 核心转换流程

```
用户调用 litellm.completion(model, messages, **kwargs)
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 1. 参数提取与分类                                             │
│    - 提取 standard_params (model, messages, stream, etc.)    │
│    - 提取 non_default_params (用户传入的额外参数)             │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 2. Provider 路由                                              │
│    - 根据 model 名称确定 provider                             │
│    - 获取对应的 Config 类实例                                  │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 3. 参数兼容检查                                               │
│    - 调用 config.get_supported_openai_params(model)          │
│    - 检查 non_default_params 中的参数是否都在支持列表中       │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 4. 参数映射                                                   │
│    - 调用 config.map_openai_params(...)                      │
│    - 将 OpenAI 参数转换为 provider 特定格式                   │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 5. 请求转换                                                   │
│    - 调用 config.transform_request(...)                      │
│    - 转换 messages 格式（OpenAI → provider 特定格式）         │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 6. API 调用                                                   │
│    - 使用 httpx 发送同步或异步请求                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│ 7. 响应转换                                                   │
│    - 调用 config.transform_response(...)                     │
│    - 将 provider 响应转换为统一的 OpenAI 格式                │
└─────────────────────────────────────────────────────────────┘
                              ↓
                    返回 ModelResponse 对象
```

## 7. 设计模式与架构原则

### 7.1 设计模式

| 设计模式 | 应用场景 | 示例 |
|---------|---------|------|
| **模板方法模式** | 定义转换流程骨架 | `BaseConfig` 抽象类 |
| **工厂模式** | 动态创建对象或检查能力 | `_supports_factory`, `get_token_counter` |
| **策略模式** | 不同 provider 不同转换策略 | `map_openai_params` 多态实现 |
| **适配器模式** | 适配各 provider 接口 | `transform_request`/`transform_response` |

### 7.2 架构原则

| 原则 | 实现方式 |
|------|---------|
| **开闭原则** | 新增 provider 只需创建新的 Config 类，无需修改核心流程 |
| **依赖倒置原则** | 高层模块依赖抽象（`BaseConfig`），不依赖具体实现 |
| **单一职责原则** | 每个 Config 类只负责一个 provider 的转换逻辑 |

## 8. 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `litellm/llms/base_llm/chat/transformation.py` | `BaseConfig` 抽象基类，定义核心转换接口 |
| `litellm/llms/base_llm/base_utils.py` | `BaseLLMModelInfo` 抽象类，`BaseTokenCounter` 接口 |
| `litellm/llms/openai/chat/gpt_transformation.py` | OpenAI GPT 系列转换实现 |
| `litellm/llms/anthropic/chat/transformation.py` | Anthropic 转换实现（复杂转换示例） |
| `litellm/llms/cohere/chat/transformation.py` | Cohere 转换实现 |
| `litellm/llms/gemini/chat/transformation.py` | Gemini 转换实现 |
| `litellm/types/llms/openai.py` | OpenAI 标准类型定义 |
| `litellm/types/llms/anthropic.py` | Anthropic 特定类型定义 |
| `model_prices_and_context_window.json` | 模型价格、上下文窗口、能力标志配置 |
| `litellm/utils.py` | `_supports_factory`, `supports_reasoning` 等辅助函数 |

---

## 附录：Provider 实现对照表

| Provider | 转换复杂度 | 主要差异点 |
|---------|-----------|-----------|
| **OpenAI** | 低 | 基准格式，几乎无需转换 |
| **Anthropic** | 高 | System message 分离、Tool 格式、Response format、Reasoning |
| **Cohere** | 中 | Chat history 分离、Tool 格式、参数重命名 |
| **Gemini** | 中 | 图片 URL 转 base64、复用 Vertex AI 逻辑 |
| **Azure OpenAI** | 低 | 主要是 URL 和 headers 处理 |
| **Bedrock** | 中 | AWS 签名、Converse API 封装 |
| **Ollama** | 低 | 消息格式标准化 |
| **Mistral** | 低 | 类 OpenAI，微调差异 |