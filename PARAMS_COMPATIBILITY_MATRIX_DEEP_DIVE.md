# LiteLLM 参数兼容矩阵维护链路深度分析

## 1. 核心路由优先级详解

### 1.1 完整路由优先级链

**文件**：`litellm/litellm_core_utils/get_llm_provider_logic.py:137`

`get_llm_provider()` 函数中的路由优先级按以下顺序依次检查，**一旦匹配立即返回**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 1: LiteLLM Proxy 默认直连分支                                          │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：LiteLLMProxyChatConfig._should_use_litellm_proxy_by_default()    │
│                                                                               │
│ 触发方式（任一满足即可）：                                                    │
│ 1. 环境变量 USE_LITELLM_PROXY=True                                          │
│ 2. litellm_params.use_litellm_proxy=True                                    │
│ 3. litellm.use_litellm_proxy=True                                           │
│                                                                               │
│ 匹配结果：custom_llm_provider = "litellm_proxy"                             │
│         直接跳过后续所有路由规则                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 2: Azure AI Studio 特殊处理                                            │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：model.split("/", 1)[0] == "azure"                                 │
│         AND _is_non_openai_azure_model(model)                                │
│                                                                               │
│ 适用场景：Azure AI Studio 中的非 OpenAI 模型（如 Cohere, Mistral）          │
│                                                                               │
│ 匹配结果：custom_llm_provider = "openai"（使用 OpenAI 兼容路由）            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 3: Cohere / Anthropic Text 模型修正                                    │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：handle_cohere_chat_model_custom_llm_provider()                    │
│         handle_anthropic_text_model_custom_llm_provider()                   │
│                                                                               │
│ 修正逻辑：                                                                    │
│ - "cohere/command-r" → custom_llm_provider = "cohere_chat"（不是 "cohere"）│
│ - 旧版 Claude 模型 → custom_llm_provider = "anthropic_text"                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 4: OpenRouter 前缀处理                                                │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：custom_llm_provider == "openrouter"                               │
│         AND model.startswith("openrouter/")                                  │
│                                                                               │
│ 处理逻辑：                                                                    │
│ - "openrouter/anthropic/claude-3.5-sonnet" → "anthropic/claude-3.5-sonnet" │
│ - "openrouter/auto" → 保持不变                                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 5: JSON 配置 Provider（⚠️ 优先于硬编码 provider_list）               │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 代码注释："Check JSON-configured providers FIRST (before enum-based         │
│           provider_list)"                                                    │
│                                                                               │
│ 检查条件：len(model.split("/")) > 1                                          │
│         AND JSONProviderRegistry.exists(provider_prefix)                     │
│                                                                               │
│ 配置来源：litellm/llms/openai_like/providers.json                          │
│                                                                               │
│ 示例：                                                                        │
│ {                                                                             │
│   "publicai": {                                                               │
│     "base_url": "https://api.publicai.co/v1",                               │
│     "api_key_env": "PUBLICAI_API_KEY",                                       │
│     "base_class": "openai_gpt"                                               │
│   }                                                                           │
│ }                                                                             │
│                                                                               │
│ 使用方式：model = "publicai/gpt-4o"                                          │
│         → custom_llm_provider = "publicai"（JSON 配置的 provider）          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 6: 硬编码 provider_list 前缀匹配                                       │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件 1：                                                                  │
│ model.split("/", 1)[0] in litellm.provider_list                             │
│ AND model.split("/", 1)[0] not in litellm.model_list_set                   │
│ AND len(model.split("/")) > 1                                                │
│                                                                               │
│ 匹配结果：返回 _get_openai_compatible_provider_info()                        │
│                                                                               │
│ 检查条件 2：                                                                  │
│ model.split("/", 1)[0] in litellm.provider_list                             │
│                                                                               │
│ 匹配结果：custom_llm_provider = model.split("/", 1)[0]                      │
│         model = model.split("/", 1)[1]                                       │
│                                                                               │
│ 示例：                                                                        │
│ "anthropic/claude-3-opus" → custom_llm_provider = "anthropic"              │
│ "openai/gpt-4o" → custom_llm_provider = "openai"                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 7: API Base 端点匹配                                                  │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：api_base 匹配已知的 OpenAI-compatible 端点                        │
│                                                                               │
│ 示例端点映射：                                                                │
│ - "api.perplexity.ai" → custom_llm_provider = "perplexity"                 │
│ - "api.mistral.ai/v1" → custom_llm_provider = "mistral"                    │
│ - "api.groq.com/openai/v1" → custom_llm_provider = "groq"                  │
│ - "ollama.com" → custom_llm_provider = "ollama"                             │
│ - 等等（50+ 个已知端点）                                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 8: 已知模型列表匹配                                                    │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：model in litellm.{provider}_models                                 │
│                                                                               │
│ 示例：                                                                        │
│ - model in litellm.open_ai_chat_completion_models → "openai"                │
│ - model in litellm.anthropic_models → "anthropic"                           │
│ - model in litellm.cohere_chat_models → "cohere_chat"                       │
│ - model in litellm.bedrock_models → "bedrock"                               │
│ - 等等（100+ 个内置模型）                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 优先级 9: 模型名称前缀匹配                                                    │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 检查条件：model.startswith("{provider}/")                                    │
│                                                                               │
│ 示例：                                                                        │
│ - "bytez/..." → custom_llm_provider = "bytez"                               │
│ - "oci/..." → custom_llm_provider = "oci"                                   │
│ - "sap/..." → custom_llm_provider = "sap"                                    │
│ - "lemonade/..." → custom_llm_provider = "lemonade"                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓ 未匹配
┌─────────────────────────────────────────────────────────────────────────────┐
│ 最终：抛出 BadRequestError 异常                                               │
│ ─────────────────────────────────────────────────────────────────────────── │
│ 错误信息："LLM Provider NOT provided. Pass in the LLM provider..."          │
│                                                                               │
│ 提示：查看 https://docs.litellm.ai/docs/providers                           │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 JSON Provider 与硬编码 Provider 的优先级关系

**关键发现**：JSON 配置的 Provider **优先级高于**硬编码的 provider_list。

**代码证据**（`get_llm_provider_logic.py:210-218`）：

```python
# Check JSON-configured providers FIRST (before enum-based provider_list)
provider_prefix = model.split("/", 1)[0]
if len(model.split("/")) > 1 and JSONProviderRegistry.exists(provider_prefix):
    return _get_openai_compatible_provider_info(
        model=model,
        api_base=api_base,
        api_key=api_key,
        dynamic_api_key=dynamic_api_key,
    )

# check if llm provider part of model name
# ... 硬编码 provider_list 检查（在 JSON 之后）
```

**设计意图**：
- 允许用户通过 JSON 配置覆盖或扩展内置 provider
- 无需修改代码即可添加新的 OpenAI-compatible provider
- 配置文件：`litellm/llms/openai_like/providers.json`

### 1.3 LiteLLM Proxy 默认直连分支详解

**文件**：`litellm/llms/litellm_proxy/chat/transformation.py`

#### 1.3.1 触发条件

```python
@staticmethod
def _should_use_litellm_proxy_by_default(
    litellm_params: Optional[LiteLLM_Params] = None,
) -> bool:
    """
    返回 True 时，所有请求都将直连到 LiteLLM Proxy
    
    适用场景：Google ADK 用户希望动态控制请求是否走 LiteLLM Proxy
    允许使用原始格式的模型名称：
    - "gemini/gemini-1.5-pro"
    - "openai/gpt-4"
    - "mistral/llama-2-70b-chat"
    """
    import litellm

    if get_secret_bool("USE_LITELLM_PROXY") is True:
        return True
    if litellm_params and litellm_params.use_litellm_proxy is True:
        return True
    if litellm.use_litellm_proxy is True:
        return True
    return False
```

#### 1.3.2 触发后的路由结果

```python
@staticmethod
def litellm_proxy_get_custom_llm_provider_info(
    model: str, api_base: Optional[str] = None, api_key: Optional[str] = None
) -> Tuple[str, str, Optional[str], Optional[str]]:
    """
    强制使用 LiteLLM Proxy 处理所有模型
    
    返回结果：
    - custom_llm_provider = "litellm_proxy"（固定值）
    - api_base = api_base 或 LITELLM_PROXY_API_BASE 环境变量
    - api_key = api_key 或 LITELLM_PROXY_API_KEY 环境变量
    """
    custom_llm_provider = "litellm_proxy"
    
    # 移除 "litellm_proxy/" 前缀（如果有）
    if model.startswith("litellm_proxy/"):
        model = model.split("/", 1)[1]
    
    # 获取 API 配置
    api_base, api_key = LiteLLMProxyChatConfig()._get_openai_compatible_provider_info(
        api_base=api_base, api_key=api_key
    )
    
    return model, custom_llm_provider, api_key, api_base
```

#### 1.3.3 LiteLLMProxyChatConfig 特点

```python
class LiteLLMProxyChatConfig(OpenAIGPTConfig):
    def get_supported_openai_params(self, model: str) -> List:
        """支持所有 OpenAI 参数"""
        params_list = super().get_supported_openai_params(model)
        params_list.extend(OPENAI_CHAT_COMPLETION_PARAMS)
        return params_list
    
    def transform_request(
        self,
        model: str,
        messages: List["AllMessageValues"],
        optional_params: dict,
        litellm_params: dict,
        headers: dict,
    ) -> dict:
        """不做任何转换，直接透传"""
        return {
            "model": model,
            "messages": messages,
            **optional_params,
        }
```

**设计特点**：
- **参数支持最大化**：`get_supported_openai_params` 返回所有 OpenAI 参数
- **请求透传**：`transform_request` 不做任何转换，直接透传给下游 proxy
- **适用场景**：当 LiteLLM 作为 SDK 使用，而实际的路由和转换由另一个 LiteLLM Proxy 实例处理

## 2. 额外丢弃参数（additional_drop_params）嵌套路径处理

### 2.1 功能概述

**文件**：`litellm/litellm_core_utils/dot_notation_indexing.py`

`additional_drop_params` 不仅支持丢弃顶层参数，还支持通过 JSONPath-like 语法丢弃嵌套结构中的参数。

### 2.2 处理流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段 1: pre_process_non_default_params 中的处理                               │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                               │
│ 仅处理简单的顶层参数路径（不包含 "." 或 "["）                                │
│                                                                               │
│ def _should_drop_param(k, additional_drop_params) -> bool:                   │
│     if additional_drop_params is not None:                                   │
│         if k in additional_drop_params:  # 仅简单的 k in list 检查          │
│             return True                                                       │
│     return False                                                              │
│                                                                               │
│ 适用示例：                                                                     │
│ additional_drop_params = ["logit_bias", "user"]  # 仅顶层参数               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段 2: get_optional_params 末尾的嵌套路径处理                                │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                               │
│ # Apply nested drops from additional_drop_params                             │
│ if additional_drop_params:                                                    │
│     # 识别嵌套路径（包含 "." 或 "["）                                         │
│     nested_paths = [p for p in additional_drop_params if is_nested_path(p)] │
│                                                                               │
│     # 逐个删除嵌套路径的值                                                     │
│     for path in nested_paths:                                                 │
│         optional_params = delete_nested_value(optional_params, path)         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 支持的路径语法

```python
def is_nested_path(path: str) -> bool:
    """
    检查路径是否需要嵌套处理
    返回 True 如果路径包含 '.' 或 '['（数组表示法）
    """
    return "." in path or "[" in path
```

| 路径语法 | 说明 | 示例 |
|---------|------|------|
| `"field"` | 顶层字段 | `"temperature"` |
| `"parent.child"` | 嵌套字段 | `"tools[0].function.parameters"` |
| `"array[*]"` | 所有数组元素（通配符） | `"tools[*]"` |
| `"array[0]"` | 特定数组元素（索引） | `"messages[0]"` |
| `"array[*].field"` | 所有数组元素中的字段 | `"tools[*].function.name"` |
| `"parent\\.child"` | 包含点号的键名（转义） | `"kubernetes\\.io.namespace"` |

### 2.4 实际使用示例

#### 示例 1: 删除所有 tools 中的 input_examples

```python
litellm.completion(
    model="claude-3-opus",
    messages=[...],
    tools=[
        {
            "type": "function",
            "function": {
                "name": "get_weather",
                "parameters": {...},
                "input_examples": [...]  # 某些 provider 不支持此字段
            }
        }
    ],
    additional_drop_params=[
        "tools[*].function.input_examples"  # 删除所有工具中的 input_examples
    ]
)
```

**处理逻辑**：
```python
# 解析路径 "tools[*].function.input_examples"
segments = ["tools", "[*]", "function", "input_examples"]

# 遍历 optional_params["tools"] 数组中的每个元素
for tool in optional_params["tools"]:
    # 删除 tool["function"]["input_examples"]
    tool["function"].pop("input_examples", None)
```

#### 示例 2: 删除特定工具的参数

```python
additional_drop_params=[
    "tools[0].function.parameters.additionalProperties"  # 删除第一个工具的 additionalProperties
]
```

#### 示例 3: 混合使用顶层和嵌套路径

```python
additional_drop_params=[
    "logit_bias",                              # 顶层参数
    "user",                                    # 顶层参数
    "tools[*].function.input_examples",       # 嵌套路径
    "messages[0].metadata"                     # 嵌套路径
]
```

### 2.5 核心实现函数

#### delete_nested_value

```python
def delete_nested_value(
    data: Dict[str, Any],
    path: str,
) -> Dict[str, Any]:
    """
    使用 JSONPath 表示法从嵌套数据中删除字段
    
    支持：
    - "field" - 顶层字段
    - "parent.child" - 嵌套字段
    - "array[*]" - 所有数组元素（通配符）
    - "array[0]" - 特定数组元素（索引）
    - "array[*].field" - 所有数组元素中的字段
    
    返回：修改后的新字典（深拷贝，不修改原数据）
    """
    import copy
    result = copy.deepcopy(data)
    
    # 解析路径段
    segments = _parse_path_segments(path)
    # "tools[*].function.input_examples" 
    # → ["tools", "[*]", "function", "input_examples"]
    
    # 递归删除
    _delete_nested_value_custom(result, segments, 0)
    
    return result
```

#### _delete_nested_value_custom（递归实现）

```python
def _delete_nested_value_custom(
    data: Union[Dict[str, Any], List[Any]],
    segments: list,
    segment_index: int = 0,
) -> None:
    """
    递归删除嵌套数据中的字段
    """
    if segment_index >= len(segments):
        return
    
    segment = segments[segment_index]
    is_last = segment_index == len(segments) - 1
    
    # 处理数组通配符: [*]
    if segment == "[*]":
        if isinstance(data, list):
            for item in data:
                if is_last:
                    pass  # 不能删除数组元素本身
                else:
                    if isinstance(item, (dict, list)):
                        _delete_nested_value_custom(item, segments, segment_index + 1)
        return
    
    # 处理数组索引: [0], [1], 等
    if segment.startswith("[") and segment.endswith("]"):
        try:
            index = int(segment[1:-1])
            if isinstance(data, list) and 0 <= index < len(data):
                if is_last:
                    pass  # 不能删除数组元素本身
                else:
                    element = data[index]
                    if isinstance(element, (dict, list)):
                        _delete_nested_value_custom(element, segments, segment_index + 1)
        except (ValueError, IndexError):
            pass
        return
    
    # 处理普通字段导航
    if isinstance(data, dict):
        if is_last:
            # 删除该字段
            data.pop(segment, None)
        else:
            # 继续深入导航
            if segment in data:
                next_segment = segments[segment_index + 1] if segment_index + 1 < len(segments) else None
                
                # 如果下一段是数组表示法，当前字段应该是 list
                if next_segment and next_segment.startswith("["):
                    if isinstance(data[segment], list):
                        _delete_nested_value_custom(data[segment], segments, segment_index + 1)
                # 否则导航到 dict
                elif isinstance(data[segment], dict):
                    _delete_nested_value_custom(data[segment], segments, segment_index + 1)
```

## 3. 用户动态放行参数（allowed_openai_params）回填机制

### 3.1 功能概述

`allowed_openai_params` 允许用户动态指定哪些参数应该被视为"支持"的参数，并最终传递到 LLM API 请求中。

### 3.2 完整处理流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段 1: 参数兼容检查（_check_valid_arg）                                      │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                               │
│ # 获取 provider 静态声明的支持参数                                            │
│ supported_params = get_supported_openai_params(model, custom_llm_provider)  │
│                                                                               │
│ # 合并用户动态指定的放行参数（关键步骤！）                                    │
│ allowed_openai_params = allowed_openai_params or []                          │
│ supported_params.extend(allowed_openai_params)  # ← 合并                   │
│                                                                               │
│ # 检查不支持参数                                                              │
│ _check_valid_arg(supported_params=supported_params)                          │
│                                                                               │
│ 效果：                                                                        │
│ - 原本不支持的参数不会触发 UnsupportedParamsError                            │
│ - 但这只是"检查通过"，参数还没有被添加到 optional_params 中                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段 2: Provider 映射（map_openai_params）                                    │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                               │
│ # 这是旧方式的处理，在 elif 分支中调用                                        │
│ if custom_llm_provider == "anthropic":                                       │
│     optional_params = AnthropicConfig().map_openai_params(                   │
│         non_default_params=non_default_params,                                │
│         optional_params=optional_params,                                      │
│         ...                                                                   │
│     )                                                                         │
│                                                                               │
│ 问题：                                                                        │
│ - map_openai_params 只会处理 provider 已知的参数                             │
│ - allowed_openai_params 中的"新参数"不会被处理                              │
│ - 这些参数仍然留在 non_default_params 中                                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 阶段 3: 动态参数回填（_apply_openai_param_overrides）                        │
│ ─────────────────────────────────────────────────────────────────────────── │
│                                                                               │
│ # 在 get_optional_params 函数的最后调用                                      │
│ optional_params = _apply_openai_param_overrides(                             │
│     optional_params=optional_params,                                          │
│     non_default_params=non_default_params,                                    │
│     allowed_openai_params=allowed_openai_params,                              │
│ )                                                                             │
│                                                                               │
│ 这是 allowed_openai_params 真正生效的地方！                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 核心实现函数

**文件**：`litellm/utils.py:4811`

```python
def _apply_openai_param_overrides(
    optional_params: dict,
    non_default_params: dict,
    allowed_openai_params: list
):
    """
    如果用户传入 allowed_openai_params，将它们应用到 optional_params
    
    这些参数会原样传递给 LLM API，因为用户明确选择要传递它们
    
    历史修复（Issue #25697）：
    之前此函数会无条件地为任何缺少的 allowed param 写入 None，
    这导致 provider SDK 收到不识别的顶层 kwargs（例如 OpenAI SDK 报错：
    `AsyncCompletions.create() got an unexpected keyword argument 'enable_thinking'`）
    
    现在只转发调用者实际发送的参数。
    """
    if allowed_openai_params:
        for param in allowed_openai_params:
            # 如果参数已经在 optional_params 中，跳过
            if param in optional_params:
                continue
            
            # 如果参数不在 non_default_params 中（用户没有实际传递），跳过
            if param not in non_default_params:
                continue
            
            # 从 non_default_params 移动到 optional_params
            # 这样参数就会被传递到最终的请求中
            optional_params[param] = non_default_params.pop(param)
    
    return optional_params
```

### 3.4 完整示例

```python
# 场景：使用新的参数 "enable_thinking"，但 AnthropicConfig 不支持此参数

# 不使用 allowed_openai_params（会报错）
try:
    litellm.completion(
        model="claude-3-opus",
        messages=[...],
        enable_thinking=True  # 新参数
    )
except UnsupportedParamsError as e:
    print(e)
    # "anthropic does not support parameters: ['enable_thinking'], for model=claude-3-opus"

# 使用 allowed_openai_params（成功）
response = litellm.completion(
    model="claude-3-opus",
    messages=[...],
    enable_thinking=True,  # 新参数
    allowed_openai_params=["enable_thinking"]  # 动态放行
)

# 此时 optional_params 中会包含 "enable_thinking": True
# 请求会原样传递给 Anthropic API
```

### 3.5 处理流程详解

```python
# 假设用户调用
litellm.completion(
    model="claude-3-opus",
    messages=[...],
    temperature=0.7,           # 标准参数
    enable_thinking=True,      # 新参数（AnthropicConfig 不支持）
    allowed_openai_params=["enable_thinking"]
)

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 1: 参数分类                                                          │
# └─────────────────────────────────────────────────────────────────────────┘
non_default_params = {
    "temperature": 0.7,
    "enable_thinking": True
}
allowed_openai_params = ["enable_thinking"]

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 2: 获取支持参数列表                                                  │
# └─────────────────────────────────────────────────────────────────────────┘
supported_params = AnthropicConfig().get_supported_openai_params("claude-3-opus")
# ["stream", "stop", "temperature", "top_p", "max_tokens", ..., "thinking"]
# 注意："enable_thinking" 不在列表中

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 3: 合并 allowed_openai_params                                        │
# └─────────────────────────────────────────────────────────────────────────┘
supported_params.extend(allowed_openai_params)
# 现在 supported_params 包含 "enable_thinking"

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 4: 检查不支持参数（_check_valid_arg）                                │
# └─────────────────────────────────────────────────────────────────────────┘
# "enable_thinking" 现在在 supported_params 中
# 不会触发 UnsupportedParamsError

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 5: Provider 映射（map_openai_params）                                │
# └─────────────────────────────────────────────────────────────────────────┘
optional_params = AnthropicConfig().map_openai_params(
    non_default_params=non_default_params,
    optional_params={},
    model="claude-3-opus",
    drop_params=False
)
# 结果：
# optional_params = {"temperature": 0.7}  # 只有已知参数
# non_default_params = {"enable_thinking": True}  # 未知参数还在这里

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 6: 动态参数回填（_apply_openai_param_overrides）                    │
# └─────────────────────────────────────────────────────────────────────────┘
optional_params = _apply_openai_param_overrides(
    optional_params={"temperature": 0.7},
    non_default_params={"enable_thinking": True},
    allowed_openai_params=["enable_thinking"]
)
# 处理过程：
# for param in ["enable_thinking"]:
#     if param not in optional_params:  # True
#     if param in non_default_params:  # True
#         optional_params[param] = non_default_params.pop(param)

# 最终结果：
# optional_params = {
#     "temperature": 0.7,
#     "enable_thinking": True  # ← 从 non_default_params 移动过来
# }
# non_default_params = {}  # 空了

# ┌─────────────────────────────────────────────────────────────────────────┐
# │ 步骤 7: 请求发送                                                          │
# └─────────────────────────────────────────────────────────────────────────┘
# optional_params 中的所有参数都会传递到最终的 API 请求中
# 包括 "enable_thinking": True
```

### 3.6 关键设计要点

| 要点 | 说明 |
|------|------|
| **两层作用** | `allowed_openai_params` 在两个阶段生效：<br>1. 参数兼容检查（避免报错）<br>2. 参数回填（添加到请求） |
| **安全设计** | 只转发用户**实际传递**的参数（`param in non_default_params`）<br>避免了之前无条件写入 `None` 导致的 SDK 错误 |
| **优先级** | `optional_params` 中已存在的参数不会被覆盖<br>（`if param in optional_params: continue`） |
| **移动语义** | 参数从 `non_default_params` 移动到 `optional_params`<br>使用 `pop()` 确保不会重复处理 |

## 4. 完整链路总结

### 4.1 端到端流程图

```
用户调用 litellm.completion(
    model="provider/model-name",
    messages=[...],
    param1=value1,           # 标准参数
    param2=value2,           # 新参数（provider 不支持）
    allowed_openai_params=["param2"],  # 动态放行
    additional_drop_params=["tools[*].input_examples"],  # 嵌套丢弃
    drop_params=False,
)
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. get_llm_provider() - Provider 路由                                       │
│    优先级 1: LiteLLM Proxy 默认直连?                                        │
│    优先级 2: Azure AI Studio 特殊处理?                                      │
│    优先级 3: JSON Provider 配置?                                            │
│    优先级 4: 硬编码 provider_list?                                           │
│    ...                                                                       │
│    → custom_llm_provider = "provider"                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. pre_process_non_default_params() - 参数预处理                            │
│    - 提取 non_default_params = {"param1": value1, "param2": value2}       │
│    - additional_drop_params 中的顶层参数被过滤掉                            │
│    - Pydantic response_format → JSON Schema 转换                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. get_optional_params() - 核心参数处理                                     │
│                                                                               │
│    3.1 支持参数判定                                                           │
│        supported_params = provider_config.get_supported_openai_params()    │
│        supported_params.extend(allowed_openai_params)  # 合并              │
│                                                                               │
│    3.2 不支持参数分流                                                         │
│        if drop_params:                                                        │
│            从 non_default_params 中移除不支持参数                            │
│        else:                                                                  │
│            抛出 UnsupportedParamsError                                        │
│                                                                               │
│    3.3 Provider 参数映射                                                      │
│        optional_params = provider_config.map_openai_params(                 │
│            non_default_params, optional_params, ...                         │
│        )                                                                      │
│                                                                               │
│    3.4 动态参数回填（关键！）                                                 │
│        optional_params = _apply_openai_param_overrides(                     │
│            optional_params, non_default_params, allowed_openai_params      │
│        )                                                                      │
│        → "param2" 从 non_default_params 移动到 optional_params              │
│                                                                               │
│    3.5 嵌套路径丢弃                                                           │
│        for path in additional_drop_params:                                   │
│            if is_nested_path(path):                                          │
│                optional_params = delete_nested_value(optional_params, path) │
│        → "tools[*].input_examples" 被删除                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. Handler.completion() - API 调用                                          │
│    - provider_config.transform_request(model, messages, optional_params)   │
│    - 发送 HTTP 请求到 provider API                                           │
│    - provider_config.transform_response(raw_response, ...)                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    ↓
                           返回 ModelResponse 对象
```

### 4.2 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| `litellm/litellm_core_utils/get_llm_provider_logic.py:137` | `get_llm_provider()` - Provider 路由核心函数 |
| `litellm/llms/litellm_proxy/chat/transformation.py` | `LiteLLMProxyChatConfig` - Proxy 直连分支 |
| `litellm/llms/openai_like/json_loader.py` | `JSONProviderRegistry` - JSON Provider 加载 |
| `litellm/litellm_core_utils/dot_notation_indexing.py` | 嵌套路径处理工具（`delete_nested_value`, `is_nested_path`） |
| `litellm/utils.py:4811` | `_apply_openai_param_overrides()` - 动态参数回填 |
| `litellm/utils.py:2953` | `_should_drop_param()` - 额外丢弃参数判断 |
| `litellm/utils.py:4004` | `_check_valid_arg()` - 不支持参数分流决策 |

### 4.3 核心机制总结表

| 机制 | 触发参数 | 处理阶段 | 关键函数 |
|------|---------|---------|---------|
| **Provider 路由** | `model` 前缀, `api_base`, 环境变量 | `get_llm_provider()` | 按优先级 1-9 依次检查 |
| **Proxy 直连** | `USE_LITELLM_PROXY`, `use_litellm_proxy` | 路由优先级 1 | `_should_use_litellm_proxy_by_default()` |
| **参数兼容检查** | `get_supported_openai_params()` | `get_optional_params()` 前期 | `_check_valid_arg()` |
| **不支持参数丢弃** | `drop_params`, `litellm.drop_params` | `_check_valid_arg()` 内 | 静默移除或抛异常 |
| **动态放行参数** | `allowed_openai_params` | 检查阶段 + 回填阶段 | `supported_params.extend()` + `_apply_openai_param_overrides()` |
| **额外丢弃参数** | `additional_drop_params` | 预处理阶段 + 末尾嵌套处理 | `_should_drop_param()` + `delete_nested_value()` |
| **嵌套路径处理** | 包含 `.` 或 `[` 的路径 | `get_optional_params()` 末尾 | `is_nested_path()` + `delete_nested_value()` |
