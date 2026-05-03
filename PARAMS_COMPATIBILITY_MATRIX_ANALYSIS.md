# LiteLLM 参数兼容矩阵维护链路分析

## 1. 整体流程概览

LiteLLM 的参数兼容矩阵维护链路是一个多层次、多阶段的处理流程，从用户调用 `litellm.completion()` 开始，到最终发送请求到具体 provider API 结束，包含以下核心阶段：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      用户调用 litellm.completion()                             │
│    (model, messages, temperature, tools, stream, response_format, etc.)     │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                     阶段 1: 参数预处理 (Parameter Preprocessing)              │
│   - 从 kwargs 中提取 standard params 与 non-default params                    │
│   - 调用 pre_process_non_default_params() 标准化参数格式                       │
│   - 处理 Pydantic response_format → JSON Schema 转换                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                     阶段 2: Provider 路由 (Provider Routing)                  │
│   - 调用 get_llm_provider() 解析 model → provider 映射                        │
│   - 支持多种路由策略：前缀匹配、模型列表、api_base 匹配、JSON 配置              │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                阶段 3: 支持参数判定 (Supported Params Detection)              │
│   - 调用 get_supported_openai_params() 获取支持列表                           │
│   - 静态声明：每个 provider 的 Config 类定义                                   │
│   - 动态检查：_supports_factory + model_prices_and_context_window.json       │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                阶段 4: 不支持参数分流 (Unsupported Params Handling)           │
│   - _check_valid_arg() 函数执行分流决策                                        │
│   - 分流规则：                                                                  │
│     • 如果 litellm.drop_params=True OR drop_params=True → 静默丢弃            │
│     • 否则 → 抛出 UnsupportedParamsError 异常                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                阶段 5: Provider 映射入口衔接 (Provider Mapping Entry)         │
│   - 旧方式：get_optional_params() 中的 elif 分支                              │
│   - 新方式：ProviderConfigManager + 多态调用                                   │
│   - 调用 config.map_openai_params() → config.transform_request()             │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                          阶段 6: API 调用与响应转换                            │
│   - Handler.completion() 发送请求                                              │
│   - config.transform_response() 转换为统一 OpenAI 格式                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 2. 阶段 1：参数预处理

### 2.1 核心函数：get_optional_params

**文件**：`litellm/utils.py:3938`

这是参数处理的核心入口函数，承担了参数分类、预处理、兼容检查和映射的全部职责。

```python
def get_optional_params(
    model: str,
    functions=None,
    function_call=None,
    temperature=None,
    top_p=None,
    n=None,
    stream=False,
    ...
    drop_params=None,           # 关键参数：不支持参数的处理策略
    allowed_openai_params=None,  # 关键参数：用户动态指定的支持参数
    additional_drop_params=None, # 关键参数：额外需要丢弃的参数
    ...
    **kwargs,
):
    passed_params = locals().copy()
    special_params = passed_params.pop("kwargs")
    
    # 获取 Provider Config
    provider_config = ProviderConfigManager.get_provider_chat_config(
        model=model, provider=LlmProviders(custom_llm_provider)
    )
    
    # 预处理非默认参数
    non_default_params = pre_process_non_default_params(
        passed_params=passed_params,
        special_params=special_params,
        custom_llm_provider=custom_llm_provider,
        additional_drop_params=additional_drop_params,
        model=model,
        provider_config=provider_config,
    )
    
    optional_params = pre_process_optional_params(...)
```

### 2.2 pre_process_non_default_params

**文件**：`litellm/utils.py:3755`

这个函数负责从用户传入的所有参数中提取出"非默认参数"，即那些用户明确传入且值不同于默认值的参数。

```python
class PreProcessNonDefaultParams:
    @staticmethod
    def base_pre_process_non_default_params(
        passed_params: dict,
        special_params: dict,
        custom_llm_provider: str,
        additional_drop_params: Optional[List[str]],
        default_param_values: dict,      # DEFAULT_CHAT_COMPLETION_PARAM_VALUES
        additional_endpoint_specific_params: List[str],
    ) -> dict:
        # 1. 合并特殊参数（**kwargs）
        for k, v in special_params.items():
            if k.startswith("aws_") and custom_llm_provider != "bedrock":
                continue  # 非 bedrock 时忽略 aws_ 前缀参数
            elif k == "hf_model_name" and custom_llm_provider != "sagemaker":
                continue
            elif k.startswith("vertex_") and custom_llm_provider not in ("vertex_ai", "vertex_ai_beta"):
                continue
            passed_params[k] = v
        
        # 2. 过滤出非默认参数
        non_default_params = {
            k: v
            for k, v in passed_params.items()
            if (
                k != "model"
                and k != "custom_llm_provider"
                and k != "api_version"
                and k != "drop_params"
                and k != "allowed_openai_params"
                and k != "additional_drop_params"
                and k not in additional_endpoint_specific_params
                and k in default_param_values           # 必须是已知的 OpenAI 参数
                and v != default_param_values[k]         # 值不同于默认值
                and _should_drop_param(
                    k=k, additional_drop_params=additional_drop_params
                ) is False                                # 不在额外丢弃列表中
            )
        }
        return non_default_params
```

### 2.3 特殊参数处理

#### 2.3.1 Pydantic response_format 转换

```python
if "response_format" in non_default_params:
    if provider_config is not None:
        non_default_params["response_format"] = (
            provider_config.get_json_schema_from_pydantic_object(
                response_format=non_default_params["response_format"]
            )
        )
    else:
        non_default_params["response_format"] = type_to_response_format_param(
            response_format=non_default_params["response_format"]
        )
```

#### 2.3.2 Tools 参数清理

```python
if "tools" in non_default_params and isinstance(non_default_params, list):
    tools = non_default_params["tools"]
    for tool in tools:
        tool_function = tool.get("function", {})
        parameters = tool_function.get("parameters", None)
        if parameters is not None:
            # 移除 'additionalProperties = False'，避免 Vertex AI/Gemini API Schema 错误
            if "additionalProperties" in new_parameters:
                del new_parameters["additionalProperties"]
```

## 3. 阶段 2：Provider 路由

### 3.1 核心函数：get_llm_provider

**文件**：`litellm/litellm_core_utils/get_llm_provider_logic.py:137`

这个函数负责将模型名称解析为具体的 provider，支持多种路由策略。

```python
def get_llm_provider(
    model: str,
    custom_llm_provider: Optional[str] = None,
    api_base: Optional[str] = None,
    api_key: Optional[str] = None,
    litellm_params: Optional[LiteLLM_Params] = None,
) -> Tuple[str, str, Optional[str], Optional[str]]:
    """
    Returns:
        Tuple[str, str, Optional[str], Optional[str]]:
        - model: 解析后的模型名称
        - custom_llm_provider: 解析后的 provider
        - dynamic_api_key: 动态获取的 API key
        - api_base: 解析后的 API base
    """
```

### 3.2 路由策略优先级

Provider 路由采用多级策略，按以下优先级依次尝试：

#### 策略 1：显式指定的 custom_llm_provider

如果用户明确传入 `custom_llm_provider` 参数，优先使用。

#### 策略 2：模型前缀匹配

如果模型名称格式为 `provider/model-name`，提取前缀作为 provider：

```python
# 检查 model 前缀是否在已知 provider 列表中
if model.split("/", 1)[0] in litellm.provider_list:
    custom_llm_provider = model.split("/", 1)[0]
    model = model.split("/", 1)[1]
```

**示例**：
- `"anthropic/claude-3-opus"` → provider = `"anthropic"`
- `"azure/gpt-4"` → provider = `"azure"`

#### 策略 3：JSON 配置 Provider

检查是否是通过 JSON 配置的自定义 provider：

```python
provider_prefix = model.split("/", 1)[0]
if len(model.split("/")) > 1 and JSONProviderRegistry.exists(provider_prefix):
    return _get_openai_compatible_provider_info(...)
```

#### 策略 4：API Base 匹配

根据 `api_base` 匹配已知的 OpenAI-compatible 端点：

```python
if api_base:
    for endpoint in litellm.openai_compatible_endpoints:
        if _endpoint_matches_api_base(endpoint, api_base):
            if endpoint == "api.perplexity.ai":
                custom_llm_provider = "perplexity"
            elif endpoint == "api.mistral.ai/v1":
                custom_llm_provider = "mistral"
            # ... 更多匹配
```

**支持的端点**（部分）：

| 端点 | Provider |
|------|----------|
| `api.perplexity.ai` | perplexity |
| `api.mistral.ai/v1` | mistral |
| `api.groq.com/openai/v1` | groq |
| `api.deepseek.com/v1` | deepseek |
| `ollama.com` | ollama |

#### 策略 5：已知模型列表匹配

遍历内置的模型列表进行匹配：

```python
if model in litellm.open_ai_chat_completion_models:
    custom_llm_provider = "openai"
elif model in litellm.anthropic_models:
    if litellm.AnthropicTextConfig._is_anthropic_text_model(model):
        custom_llm_provider = "anthropic_text"
    else:
        custom_llm_provider = "anthropic"
elif model in litellm.cohere_chat_models:
    custom_llm_provider = "cohere_chat"
# ... 更多模型列表
```

#### 策略 6：模型名称前缀匹配

部分 provider 通过模型名称前缀识别：

```python
elif model.startswith("bytez/"):
    custom_llm_provider = "bytez"
elif model.startswith("oci/"):
    custom_llm_provider = "oci"
elif model.startswith("sap/"):
    custom_llm_provider = "sap"
```

### 3.3 特殊路由逻辑

#### 3.3.1 OpenRouter 路由

```python
if custom_llm_provider == "openrouter" and model.startswith("openrouter/"):
    remainder = model[len("openrouter/") :]
    if "/" in remainder:
        # "openrouter/anthropic/claude-3.5-sonnet" → "anthropic/claude-3.5-sonnet"
        return remainder, custom_llm_provider, dynamic_api_key, api_base
    # "openrouter/auto" → 保持原样
    return model, custom_llm_provider, dynamic_api_key, api_base
```

#### 3.3.2 Azure 特殊路由

```python
if model.split("/", 1)[0] == "azure":
    if _is_non_openai_azure_model(model):
        # Azure AI Studio 中的非 OpenAI 模型（如 Cohere, Mistral）
        custom_llm_provider = "openai"
        return model, custom_llm_provider, dynamic_api_key, api_base
```

#### 3.3.3 Cohere 路由修正

```python
model, custom_llm_provider = handle_cohere_chat_model_custom_llm_provider(
    model, custom_llm_provider
)
# "cohere/command-r" → provider = "cohere_chat" (不是 "cohere")
```

#### 3.3.4 Anthropic Text 路由修正

```python
model, custom_llm_provider = handle_anthropic_text_model_custom_llm_provider(
    model, custom_llm_provider
)
# 旧版本 Claude 模型 → provider = "anthropic_text"
```

## 4. 阶段 3：支持参数判定

### 4.1 核心函数：get_supported_openai_params

**文件**：`litellm/utils.py`（通过 getattr 动态获取）

获取支持参数列表采用双层机制：**静态声明** + **动态检查**。

### 4.2 静态声明：Config 类方法

每个 provider 的 Config 类都实现了 `get_supported_openai_params()` 方法：

```python
# AnthropicConfig.get_supported_openai_params()
def get_supported_openai_params(self, model: str):
    params = [
        "stream", "stop", "temperature", "top_p", "max_tokens",
        "max_completion_tokens", "tools", "tool_choice", "extra_headers",
        "parallel_tool_calls", "response_format", "user", "web_search_options",
        "speed", "context_management", "cache_control",
    ]
    
    # 动态检查：推理模型额外支持
    if (
        "claude-3-7-sonnet" in model
        or AnthropicConfig._is_claude_4_6_model(model)
        or supports_reasoning(
            model=model,
            custom_llm_provider=self.custom_llm_provider,
        )
    ):
        params.append("thinking")
        params.append("reasoning_effort")
    
    return params
```

### 4.3 动态检查：_supports_factory

**文件**：`litellm/utils.py`

对于模型特定的能力（如推理、视觉、函数调用），通过读取 `model_prices_and_context_window.json` 配置文件进行动态检查：

```python
def _supports_factory(model: str, custom_llm_provider: Optional[str], key: str) -> bool:
    """
    通用能力检查工厂函数
    
    Args:
        key: 能力键名，如 "supports_reasoning", "supports_vision"
    """
    try:
        # 1. 解析模型和 provider
        model, custom_llm_provider, _, _ = litellm.get_llm_provider(
            model=model, custom_llm_provider=custom_llm_provider
        )
        
        # 2. 从配置文件获取模型信息
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

### 4.4 能力辅助函数

基于 `_supports_factory` 的封装：

```python
def supports_reasoning(model: str, custom_llm_provider: Optional[str] = None) -> bool:
    return _supports_factory(model, custom_llm_provider, "supports_reasoning")

def supports_vision(model: str, custom_llm_provider: Optional[str] = None) -> bool:
    return _supports_factory(model, custom_llm_provider, "supports_vision")

def supports_function_calling(model: str, custom_llm_provider: Optional[str] = None) -> bool:
    return _supports_factory(model, custom_llm_provider, "supports_function_calling")

def supports_prompt_caching(model: str, custom_llm_provider: Optional[str] = None) -> bool:
    return _supports_factory(model, custom_llm_provider, "supports_prompt_caching")
```

### 4.5 配置文件示例

**文件**：`model_prices_and_context_window.json`

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

### 4.6 用户动态指定：allowed_openai_params

用户可以在请求中通过 `allowed_openai_params` 参数动态指定支持的参数：

```python
supported_params.extend(allowed_openai_params)  # 合并用户指定的参数
```

**使用示例**：
```python
litellm.completion(
    model="claude-3-opus",
    messages=[...],
    some_new_param="value",  # 新参数，静态列表中没有
    allowed_openai_params=["some_new_param"]  # 动态指定支持
)
```

## 5. 阶段 4：不支持参数分流规则

### 5.1 核心函数：_check_valid_arg

**文件**：`litellm/utils.py:4004`（嵌套在 `get_optional_params` 内）

这是分流决策的核心函数，决定不支持的参数是丢弃还是报错。

```python
def _check_valid_arg(supported_params: List[str]):
    """
    检查传入的参数是否被 provider 支持
    
    Args:
        supported_params: 该 provider 支持的参数列表
    """
    unsupported_params = {}
    
    # 1. 遍历所有非默认参数，找出不支持的
    for k in non_default_params.keys():
        if k not in supported_params:
            # 跳过某些特殊参数
            if k == "user" or k == "stream_options" or k == "stream":
                continue
            if k == "n" and n == 1:  # LangChain 默认传 n=1
                continue
            if k == "max_retries":  # 特殊处理
                continue
            else:
                unsupported_params[k] = non_default_params[k]
    
    # 2. 分流决策
    if unsupported_params:
        if litellm.drop_params is True or (
            drop_params is not None and drop_params is True
        ):
            # 策略 A：静默丢弃
            for k in unsupported_params.keys():
                non_default_params.pop(k, None)
        else:
            # 策略 B：抛出异常
            raise UnsupportedParamsError(
                status_code=500,
                message=f"{custom_llm_provider} does not support parameters: "
                        f"{list(unsupported_params.keys())}, for model={model}. "
                        f"To drop these, set `litellm.drop_params=True` or for proxy:\n\n"
                        f"`litellm_settings:\n drop_params: true`\n. \n"
                        f"If you want to use these params dynamically send "
                        f"allowed_openai_params={list(unsupported_params.keys())} "
                        f"in your request.",
            )
```

### 5.2 分流决策流程图

```
                    ┌──────────────────────────────────┐
                    │   检测到 unsupported_params      │
                    └──────────────────────────────────┘
                                    ↓
                    ┌──────────────────────────────────┐
                    │ litellm.drop_params is True?     │
                    │     OR                            │
                    │ drop_params parameter is True?   │
                    └──────────────────────────────────┘
                        │                   │
                       YES                  NO
                        │                   │
                        ↓                   ↓
            ┌─────────────────┐   ┌─────────────────────────┐
            │   静默丢弃参数  │   │  抛出 Unsupported-      │
            │   (从 non_     │   │  ParamsError 异常       │
            │    default_    │   │                          │
            │    params 移除)│   │  错误信息包含：          │
            │                 │   │  - 不支持的参数列表      │
            │                 │   │  - 如何启用 drop_params  │
            │                 │   │  - 如何使用 allowed_     │
            │                 │   │    openai_params         │
            └─────────────────┘   └─────────────────────────┘
```

### 5.3 设置方式

drop_params 可以通过三种方式设置：

#### 方式 1：全局设置

```python
import litellm
litellm.drop_params = True
```

#### 方式 2：单请求设置

```python
litellm.completion(
    model="claude-3-opus",
    messages=[...],
    temperature=0.7,
    some_unsupported_param="value",
    drop_params=True  # 该请求中丢弃不支持的参数
)
```

#### 方式 3：Proxy 配置文件设置

```yaml
litellm_settings:
  drop_params: true
```

### 5.4 特殊参数豁免

以下参数即使不在 `supported_params` 列表中，也不会被标记为不支持：

| 参数 | 豁免原因 |
|------|---------|
| `user` | 通用参数，几乎所有 provider 都支持 |
| `stream_options` | 流式响应相关 |
| `stream` | 流式标志 |
| `n` (当 n=1) | LangChain 默认传 n=1 |
| `max_retries` | LiteLLM 内部使用 |

## 6. 阶段 5：Provider 映射入口衔接

### 6.1 两种实现方式

LiteLLM 中存在两种 provider 映射入口的实现方式：

#### 方式 1：旧方式 - elif 分支链

**文件**：`litellm/utils.py:4066-4250`

在 `get_optional_params` 函数中，通过冗长的 `elif` 链调用对应 provider 的 `map_openai_params`：

```python
if custom_llm_provider == "anthropic":
    optional_params = litellm.AnthropicConfig().map_openai_params(
        model=model,
        non_default_params=non_default_params,
        optional_params=optional_params,
        drop_params=drop_params if drop_params is not None else False,
    )
elif custom_llm_provider == "anthropic_text":
    optional_params = litellm.AnthropicTextConfig().map_openai_params(...)
elif custom_llm_provider in ("cohere_chat", "cohere"):
    optional_params = litellm.CohereChatConfig().map_openai_params(...)
elif custom_llm_provider == "triton":
    optional_params = litellm.TritonConfig().map_openai_params(...)
elif custom_llm_provider == "maritalk":
    optional_params = litellm.MaritalkConfig().map_openai_params(...)
# ... 继续 50+ 个 elif 分支
```

#### 方式 2：新方式 - ProviderConfigManager

**文件**：`litellm/utils.py:8020`

使用工厂模式 + 多态调用，更加优雅和可扩展：

```python
class ProviderConfigManager:
    _PROVIDER_CONFIG_MAP: Optional[dict[LlmProviders, tuple[Callable, bool]]] = None
    
    @staticmethod
    def _build_provider_config_map() -> dict:
        """构建 provider 到 config 工厂的映射"""
        return {
            # 常见 Provider
            LlmProviders.OPENAI: (lambda: litellm.OpenAIGPTConfig(), False),
            LlmProviders.ANTHROPIC: (lambda: litellm.AnthropicConfig(), False),
            
            # 需要 model 参数的 Provider
            LlmProviders.AZURE: (
                lambda model: ProviderConfigManager._get_azure_config(model),
                True,
            ),
            LlmProviders.VERTEX_AI: (
                lambda model: ProviderConfigManager._get_vertex_ai_config(model),
                True,
            ),
            LlmProviders.BEDROCK: (
                lambda model: ProviderConfigManager._get_bedrock_config(model),
                True,
            ),
            LlmProviders.COHERE: (
                lambda model: ProviderConfigManager._get_cohere_config(model),
                True,
            ),
            
            # ... 更多 provider
        }
    
    @staticmethod
    def get_provider_chat_config(
        model: str, provider: LlmProviders
    ) -> Optional[BaseConfig]:
        """获取 provider 对应的 Config 实例"""
        if ProviderConfigManager._PROVIDER_CONFIG_MAP is None:
            ProviderConfigManager._PROVIDER_CONFIG_MAP = (
                ProviderConfigManager._build_provider_config_map()
            )
        
        factory_info = ProviderConfigManager._PROVIDER_CONFIG_MAP.get(provider)
        if factory_info is None:
            return None
        
        factory_fn, needs_model = factory_info
        if needs_model:
            return factory_fn(model)
        return factory_fn()
```

### 6.2 动态 Config 选择

某些 provider 需要根据模型进一步选择具体的 Config：

```python
@staticmethod
def _get_azure_config(model: str) -> BaseConfig:
    if litellm.AzureOpenAIO1Config().is_o_series_model(model=model):
        return litellm.AzureOpenAIO1Config()
    if litellm.AzureOpenAIGPT5Config.is_model_gpt_5_model(model=model):
        return litellm.AzureOpenAIGPT5Config()
    return litellm.AzureOpenAIConfig()

@staticmethod
def _get_vertex_ai_config(model: str) -> BaseConfig:
    if "gemini" in model:
        return litellm.VertexGeminiConfig()
    elif "claude" in model:
        return litellm.VertexAIAnthropicConfig()
    elif "gpt-oss" in model:
        return VertexAIGPTOSSTransformation()
    elif model in litellm.vertex_mistral_models:
        return litellm.MistralConfig()
    else:
        return litellm.VertexAILlama3Config()
```

### 6.3 Handler 层的衔接

**文件**：`litellm/llms/anthropic/chat/handler.py:321`

在具体的 handler 中，通过 `ProviderConfigManager` 获取 Config 并调用转换方法：

```python
def completion(
    self,
    model: str,
    messages: list,
    api_base: str,
    custom_llm_provider: str,
    optional_params: dict,
    litellm_params: dict,
    ...
):
    # 1. 通过 ProviderConfigManager 获取 Config
    config = ProviderConfigManager.get_provider_chat_config(
        model=model,
        provider=LlmProviders(custom_llm_provider),
    )
    
    # 2. 调用 transform_request 转换整个请求体
    data = config.transform_request(
        model=model,
        messages=messages,
        optional_params={**optional_params, "is_vertex_request": is_vertex_request},
        litellm_params=litellm_params,
        headers=headers,
    )
    
    # 3. 调用 transform_response 转换响应
    return config.transform_response(
        model=model,
        raw_response=response,
        model_response=model_response,
        logging_obj=logging_obj,
        api_key=api_key,
        request_data=data,
        messages=messages,
        optional_params=optional_params,
        litellm_params=litellm_params,
        encoding=encoding,
        json_mode=json_mode,
    )
```

### 6.4 完整衔接流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        用户调用 litellm.completion()                          │
│  model="anthropic/claude-3-opus", messages=[...], temperature=0.7, ...     │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                           get_llm_provider()                                  │
│  解析 model 前缀 → custom_llm_provider = "anthropic"                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                         get_optional_params()                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 1. 提取 non_default_params = {"temperature": 0.7, ...}              │    │
│  │ 2. 获取 supported_params = AnthropicConfig.get_supported_openai_    │    │
│  │    params(model)                                                      │    │
│  │ 3. _check_valid_arg() 检查不支持参数                                  │    │
│  │ 4. [旧方式] elif custom_llm_provider == "anthropic":                │    │
│  │    optional_params = AnthropicConfig().map_openai_params(...)        │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AnthropicChatCompletion.completion()                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │ 1. ProviderConfigManager.get_provider_chat_config(                   │    │
│  │      model, LlmProviders.ANTHROPIC                                   │    │
│  │    ) → returns AnthropicConfig() 实例                                │    │
│  │                                                                       │    │
│  │ 2. config.transform_request(                                          │    │
│  │      model, messages, optional_params, ...                           │    │
│  │    ) → 完整请求体转换（含 messages 格式）                             │    │
│  │                                                                       │    │
│  │ 3. 发送 HTTP POST 请求到 Anthropic API                                │    │
│  │                                                                       │    │
│  │ 4. config.transform_response(                                         │    │
│  │      raw_response, model_response, ...                                │    │
│  │    ) → 转换为统一 OpenAI 格式                                         │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                        ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                            返回 ModelResponse 对象                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 7. 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `litellm/utils.py:3938` | `get_optional_params()` - 参数处理核心函数 |
| `litellm/utils.py:3755` | `pre_process_non_default_params()` - 非默认参数提取 |
| `litellm/utils.py:4004` | `_check_valid_arg()` - 不支持参数分流决策 |
| `litellm/utils.py:8020` | `ProviderConfigManager` - 新方式 provider config 工厂 |
| `litellm/litellm_core_utils/get_llm_provider_logic.py:137` | `get_llm_provider()` - Provider 路由核心函数 |
| `litellm/llms/anthropic/chat/handler.py:321` | `AnthropicChatCompletion.completion()` - Handler 层衔接示例 |
| `litellm/llms/base_llm/chat/transformation.py` | `BaseConfig` 抽象基类 |
| `model_prices_and_context_window.json` | 模型能力标志配置文件 |

## 8. 设计要点总结

### 8.1 设计优势

1. **双层参数兼容机制**：静态声明 + 动态检查，兼顾确定性和灵活性
2. **灵活的分流策略**：支持全局/单请求/配置文件三种方式设置 `drop_params`
3. **多态设计**：`ProviderConfigManager` 避免了冗长的 elif 链，易于扩展
4. **用户可定制**：`allowed_openai_params` 允许用户动态指定支持的新参数

### 8.2 扩展指南

新增 provider 时需要：

1. **创建 Config 类**：继承 `BaseConfig`，实现四个核心方法：
   - `get_supported_openai_params(model)` - 声明支持的参数
   - `map_openai_params(...)` - 参数格式映射
   - `transform_request(...)` - 请求体转换
   - `transform_response(...)` - 响应体转换

2. **注册到 ProviderConfigManager**（可选）：
   ```python
   LlmProviders.NEW_PROVIDER: (lambda: NewProviderConfig(), False),
   ```

3. **添加模型到配置**（如需要动态能力检查）：
   - 在 `model_prices_and_context_window.json` 中添加模型条目
   - 包含 `supports_*` 能力标志
