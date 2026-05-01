# LiteLLM 多模型供应商适配架构技术分析报告

## 目录

1. [概述](#1-概述)
2. [基类抽象与统一接口设计](#2-基类抽象与统一接口设计)
3. [模型路由机制](#3-模型路由机制)
4. [请求参数归一化](#4-请求参数归一化)
5. [响应结构归一化](#5-响应结构归一化)
6. [异常类型归一化](#6-异常类型归一化)
7. [流式输出归一化](#7-流式输出归一化)
8. [供应商特有参数透传机制](#8-供应商特有参数透传机制)
9. [架构总结与设计亮点](#9-架构总结与设计亮点)

---

## 1. 概述

LiteLLM 是一个统一的 100+ 大语言模型接口库，其核心价值在于：

- **统一接口**：将所有 AI 服务供应商的 API 统一为 OpenAI 兼容格式
- **多态实现**：通过基类抽象支持不同供应商的差异化实现
- **路由机制**：根据模型标识符自动路由到对应供应商
- **参数透传**：支持供应商特有参数的灵活透传

本文档深入分析 LiteLLM 的多模型供应商适配架构，揭示其如何实现上述核心功能。

---

## 2. 基类抽象与统一接口设计

### 2.1 核心抽象类：BaseConfig

LiteLLM 的核心设计思想是通过 `BaseConfig` 抽象类定义统一接口，所有供应商配置类继承该基类并实现抽象方法。

**关键代码位置**：`litellm/llms/base_llm/chat/transformation.py:81`

```python
class BaseConfig(ABC):
    """所有 LLM 供应商配置的通用基类"""
    
    @abstractmethod
    def get_supported_openai_params(self, model: str) -> list:
        """返回该供应商支持的 OpenAI 参数列表"""
        pass

    @abstractmethod
    def map_openai_params(
        self,
        non_default_params: dict,
        optional_params: dict,
        model: str,
        drop_params: bool,
    ) -> dict:
        """将 OpenAI 参数映射为供应商特有参数"""
        pass

    @abstractmethod
    def validate_environment(
        self,
        headers: dict,
        model: str,
        messages: List[AllMessageValues],
        optional_params: dict,
        litellm_params: dict,
        api_key: Optional[str] = None,
        api_base: Optional[str] = None,
    ) -> dict:
        """验证环境并设置请求头"""
        pass

    @abstractmethod
    def transform_request(
        self,
        model: str,
        messages: List[AllMessageValues],
        optional_params: dict,
        litellm_params: dict,
        headers: dict,
    ) -> dict:
        """转换请求体为供应商特定格式"""
        pass

    @abstractmethod
    def transform_response(
        self,
        model: str,
        raw_response: httpx.Response,
        model_response: "ModelResponse",
        logging_obj: LiteLLMLoggingObj,
        request_data: dict,
        messages: List[AllMessageValues],
        optional_params: dict,
        litellm_params: dict,
        encoding: Any,
        api_key: Optional[str] = None,
        json_mode: Optional[bool] = None,
    ) -> "ModelResponse":
        """转换供应商响应为统一的 ModelResponse 格式"""
        pass

    @abstractmethod
    def get_error_class(
        self, error_message: str, status_code: int, headers: Union[dict, httpx.Headers]
    ) -> BaseLLMException:
        """获取供应商特定的错误类"""
        pass
```

### 2.2 统一入口：completion() 函数

**关键代码位置**：`litellm/main.py:379`

`completion()` 和 `acompletion()` 是 LiteLLM 对外暴露的统一调用入口，负责：

1. 参数收集与预处理
2. 模型路由解析
3. 供应商配置实例化
4. 请求发送与响应处理
5. 异常捕获与归一化

### 2.3 架构层次图

```
┌─────────────────────────────────────────────────────────────────┐
│                      应用层 (Application)                         │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  litellm.completion() / acompletion()                    │    │
│  │  - 统一参数签名                                            │    │
│  │  - OpenAI 兼容格式                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      路由层 (Routing)                             │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  get_llm_provider()                                      │    │
│  │  - 模型前缀匹配 (e.g., "anthropic/claude-3-opus")       │    │
│  │  - 已知模型列表匹配                                       │    │
│  │  - API Base 端点匹配                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    转换层 (Transformation)                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  BaseConfig (抽象基类)                                    │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │  OpenAIGPTConfig              │    │    │
│  │  │  - 直接传递 OpenAI 参数                                │    │    │
│  │  │  - 最小化转换                                          │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │  AnthropicConfig            │    │    │
│  │  │  - 消息格式转换 (OpenAI → Anthropic)               │    │    │
│  │  │  - 工具调用转换                                      │    │    │
│  │  │  - 响应格式转换 (Anthropic → OpenAI)               │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  │  ┌─────────────────────────────────────────────────┐    │    │
│  │  │  BedrockConfig / VertexAIConfig / ...           │    │    │
│  │  │  - 各供应商特定实现                                │    │    │
│  │  └─────────────────────────────────────────────────┘    │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    归一化层 (Normalization)                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │ 参数映射    │  │ 响应转换    │  │ 异常映射    │             │
│  │ map_openai_ │  │ transform_  │  │ exception_  │             │
│  │ params()    │  │ response()  │  │ type()      │             │
│  └─────────────┘  └─────────────┘  └─────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. 模型路由机制

### 3.1 核心路由函数：get_llm_provider()

**关键代码位置**：`litellm/litellm_core_utils/get_llm_provider_logic.py:137`

```python
def get_llm_provider(
    model: str,
    custom_llm_provider: Optional[str] = None,
    api_base: Optional[str] = None,
    api_key: Optional[str] = None,
    litellm_params: Optional[LiteLLM_Params] = None,
) -> Tuple[str, str, Optional[str], Optional[str]]:
    """
    根据模型名称返回供应商信息
    
    返回: (model, custom_llm_provider, dynamic_api_key, api_base)
    """
```

### 3.2 三种路由策略

#### 策略 1：模型前缀路由

当模型名称包含 `/` 分隔符时，前缀被识别为供应商标识：

```python
# 示例
model = "anthropic/claude-3-opus-20240229"
# 解析结果:
#   model = "claude-3-opus-20240229"
#   custom_llm_provider = "anthropic"

model = "azure/gpt-4"
# 解析结果:
#   model = "gpt-4"
#   custom_llm_provider = "azure"
```

**关键代码**：`litellm/litellm_core_utils/get_llm_provider_logic.py:222-237`

```python
if (
    model.split("/", 1)[0] in litellm.provider_list
    and model.split("/", 1)[0] not in litellm.model_list_set
    and len(model.split("/")) > 1
):
    return _get_openai_compatible_provider_info(...)
```

#### 策略 2：已知模型列表匹配

LiteLLM 维护了各供应商的已知模型列表，通过精确匹配或前缀匹配识别：

```python
# 示例已知模型列表
litellm.open_ai_chat_completion_models = ["gpt-4", "gpt-3.5-turbo", ...]
litellm.anthropic_models = ["claude-3-opus-20240229", "claude-3-sonnet-20240229", ...]
litellm.bedrock_models = ["amazon.titan-tg1-large", "anthropic.claude-v2", ...]
```

**关键代码**：`litellm/litellm_core_utils/get_llm_provider_logic.py:384-499`

```python
if model in litellm.open_ai_chat_completion_models:
    custom_llm_provider = "openai"
elif model in litellm.anthropic_models:
    custom_llm_provider = "anthropic"
elif model in litellm.bedrock_models:
    custom_llm_provider = "bedrock"
# ... 更多供应商判断
```

#### 策略 3：API Base 端点匹配

当用户提供 `api_base` 时，LiteLLM 通过匹配已知端点来识别供应商：

**关键代码**：`litellm/litellm_core_utils/get_llm_provider_logic.py:249-380`

```python
if api_base:
    for endpoint in litellm.openai_compatible_endpoints:
        if _endpoint_matches_api_base(endpoint, api_base):
            # 根据端点设置对应供应商
            if endpoint == "api.perplexity.ai":
                custom_llm_provider = "perplexity"
            elif endpoint == "api.groq.com/openai/v1":
                custom_llm_provider = "groq"
            elif endpoint == "https://api.mistral.ai/v1":
                custom_llm_provider = "mistral"
            # ... 更多端点匹配
```

### 3.3 路由策略的优先级与适用边界

LiteLLM 的三种路由策略并非平等，而是有明确的**优先级顺序**：

| 优先级 | 路由策略 | 触发条件 | 适用边界 |
|:---:|---------|---------|---------|
| **1 (最高)** | **显式供应商标识** | 用户显式指定 `custom_llm_provider` 参数 | 所有场景 |
| **2** | **JSON 配置供应商** | 模型前缀匹配 `JSONProviderRegistry` 中的供应商 | 动态配置的供应商 |
| **3** | **模型前缀路由** | 模型名称以已知供应商为前缀（如 `"anthropic/claude-3-opus"`） | 需要精确控制供应商时 |
| **4** | **API Base 端点匹配** | `api_base` 匹配已知 OpenAI 兼容端点 | OpenAI 兼容的 API 服务 |
| **5 (最低)** | **已知模型列表匹配** | 模型名称在 `litellm.model_list` 中 | 常用模型的便捷使用 |

**关键代码**：`litellm/litellm_core_utils/get_llm_provider_logic.py:153-500`

```python
# 优先级 1: 显式指定 custom_llm_provider
if litellm_params and litellm_params.custom_llm_provider:
    custom_llm_provider = litellm_params.custom_llm_provider

# 优先级 2: JSON 配置供应商（优先于 enum-based provider_list）
provider_prefix = model.split("/", 1)[0]
if len(model.split("/")) > 1 and JSONProviderRegistry.exists(provider_prefix):
    return _get_openai_compatible_provider_info(...)

# 优先级 3: 模型前缀路由
if (
    model.split("/", 1)[0] in litellm.provider_list
    and model.split("/", 1)[0] not in litellm.model_list_set
    and len(model.split("/")) > 1
):
    return _get_openai_compatible_provider_info(...)

# 优先级 4: API Base 端点匹配
if api_base:
    for endpoint in litellm.openai_compatible_endpoints:
        if _endpoint_matches_api_base(endpoint, api_base):
            # 设置对应供应商...

# 优先级 5: 已知模型列表匹配
if model in litellm.open_ai_chat_completion_models:
    custom_llm_provider = "openai"
elif model in litellm.anthropic_models:
    custom_llm_provider = "anthropic"
# ...
```

**设计权衡**：

| 路由策略 | 优点 | 缺点 |
|---------|-----|-----|
| **显式供应商标识** | 最明确，不会有歧义 | 需要额外参数 |
| **模型前缀路由** | 自描述，无需额外配置 | 模型名称较长 |
| **API Base 端点匹配** | 适合 OpenAI 兼容服务 | 可能受 URL 注入攻击（已防护） |
| **已知模型列表匹配** | 使用便捷，名称最短 | 新模型可能未及时更新 |

### 3.4 端点匹配的安全实现

LiteLLM 使用解析后的 URL 进行匹配，而非简单的子字符串搜索，以防止安全漏洞：

**关键代码**：`litellm/litellm_core_utils/get_llm_provider_logic.py:12-46`

```python
def _endpoint_matches_api_base(endpoint: str, api_base: str) -> bool:
    """
    安全的端点匹配，防止 URL 注入攻击
    
    问题场景：攻击者传递 "https://attacker.com/api.groq.com/openai/v1"
    可能被错误识别为 Groq 供应商，导致 API Key 泄露
    """
    def _parse(value: str):
        normalized = value if "://" in value else f"https://{value}"
        return urlparse(normalized)
    
    parsed_endpoint = _parse(endpoint)
    parsed_url = _parse(api_base)
    
    # 主机名必须精确匹配
    endpoint_host = (parsed_endpoint.hostname or "").lower()
    url_host = (parsed_url.hostname or "").lower()
    if not endpoint_host or endpoint_host != url_host:
        return False
    
    # 路径必须以注册端点路径开头
    endpoint_path = parsed_endpoint.path.rstrip("/")
    if not endpoint_path:
        return True
    url_path = parsed_url.path.rstrip("/")
    return url_path == endpoint_path or url_path.startswith(endpoint_path + "/")
```

---

## 4. 请求参数归一化

### 4.1 参数映射流程

请求参数归一化的核心是 `map_openai_params()` 方法，将 OpenAI 格式的参数转换为各供应商特有格式。

#### 示例 1：OpenAI 供应商（最小转换）

**关键代码位置**：`litellm/llms/openai/chat/gpt_transformation.py:189-225`

```python
def get_supported_openai_params(self, model: str) -> list:
    """OpenAI 支持的参数列表"""
    base_params = [
        "frequency_penalty", "logit_bias", "logprobs", 
        "max_tokens", "max_completion_tokens", "modalities",
        "n", "presence_penalty", "seed", "stop",
        "stream", "stream_options", "temperature", "top_p",
        "tools", "tool_choice", "function_call", "functions",
        "parallel_tool_calls", "audio", "web_search_options",
        "service_tier", "safety_identifier",
        "prompt_cache_key", "prompt_cache_retention", "store",
    ]
    # ... 模型特定参数
    return base_params + model_specific_params

def _map_openai_params(
    self,
    non_default_params: dict,
    optional_params: dict,
    model: str,
    drop_params: bool,
) -> dict:
    """OpenAI 参数直接传递"""
    supported_openai_params = self.get_supported_openai_params(model)
    for param, value in non_default_params.items():
        if param in supported_openai_params:
            optional_params[param] = value
    return optional_params
```

#### 示例 2：Anthropic 供应商（复杂转换）

**关键代码位置**：`litellm/llms/anthropic/chat/transformation.py:989-1140`

Anthropic 需要进行大量格式转换，因为其 API 设计与 OpenAI 有显著差异：

```python
def map_openai_params(
    self,
    non_default_params: dict,
    optional_params: dict,
    model: str,
    drop_params: bool,
) -> dict:
    """将 OpenAI 参数映射为 Anthropic 格式"""
    
    for param, value in non_default_params.items():
        # OpenAI max_tokens → Anthropic max_tokens (但有默认值要求)
        if param == "max_tokens":
            optional_params["max_tokens"] = value
        
        # OpenAI tools → Anthropic tools (需要格式转换)
        elif param == "tools":
            anthropic_tools, mcp_servers = self._map_tools(value)
            optional_params = self._add_tools_to_optional_params(
                optional_params=optional_params, tools=anthropic_tools
            )
            if mcp_servers:
                optional_params["mcp_servers"] = mcp_servers
        
        # OpenAI tool_choice → Anthropic tool_choice
        elif param == "tool_choice" or param == "parallel_tool_calls":
            _tool_choice = self._map_tool_choice(
                tool_choice=non_default_params.get("tool_choice"),
                parallel_tool_use=non_default_params.get("parallel_tool_calls"),
            )
            if _tool_choice is not None:
                optional_params["tool_choice"] = _tool_choice
        
        # OpenAI stop → Anthropic stop_sequences
        elif param == "stop":
            _value = self._map_stop_sequences(value)
            if _value is not None:
                optional_params["stop_sequences"] = _value
        
        # OpenAI response_format → Anthropic tool (JSON 模式)
        elif param == "response_format" and isinstance(value, dict):
            if any(substring in model for substring in {"sonnet-4.5", ...}):
                # 新版本支持 output_format
                _output_format = self.map_response_format_to_anthropic_output_format(value)
                optional_params["output_format"] = _output_format
            else:
                # 旧版本通过工具调用模拟
                _tool = self.map_response_format_to_anthropic_tool(
                    value, optional_params, is_thinking_enabled
                )
                optional_params = self._add_tools_to_optional_params(
                    optional_params=optional_params, tools=[_tool]
                )
                optional_params["json_mode"] = True
        
        # OpenAI user → Anthropic metadata.user_id
        elif param == "user" and value is not None:
            optional_params["metadata"] = {"user_id": value}
        
        # OpenAI reasoning_effort → Anthropic thinking
        elif param == "reasoning_effort" and isinstance(value, str):
            optional_params["thinking"] = AnthropicConfig._map_reasoning_effort(
                reasoning_effort=value, model=model
            )
        
        # ... 更多参数映射
```

### 4.2 工具调用转换

Anthropic 的工具调用格式与 OpenAI 不同，需要专门转换：

**关键代码位置**：`litellm/llms/anthropic/chat/transformation.py:421-644`

```python
def _map_tool_helper(
    self, tool: ChatCompletionToolParam
) -> Tuple[Optional[AllAnthropicToolsValues], Optional[AnthropicMcpServerTool]]:
    """将 OpenAI 工具格式转换为 Anthropic 格式"""
    
    if tool["type"] == "function" or tool["type"] == "custom":
        # OpenAI 格式:
        # {
        #   "type": "function",
        #   "function": {
        #     "name": "get_weather",
        #     "description": "获取天气",
        #     "parameters": {...}
        #   }
        # }
        
        _input_schema = tool["function"].get("parameters", {...})
        
        # Anthropic 格式:
        # {
        #   "name": "get_weather",
        #   "description": "获取天气",
        #   "input_schema": {...}
        # }
        
        _tool = AnthropicMessagesTool(
            name=tool["function"]["name"],
            input_schema=_input_schema,
            type="custom",
        )
        
        if tool["function"].get("description"):
            _tool["description"] = tool["function"]["description"]
        
        return _tool, None
    
    # ... 其他工具类型处理
```

### 4.3 请求体构建：transform_request()

参数映射完成后，`transform_request()` 方法构建最终的请求体：

**OpenAI 示例**：`litellm/llms/openai/chat/gpt_transformation.py:423-450`

```python
def transform_request(
    self,
    model: str,
    messages: List[AllMessageValues],
    optional_params: dict,
    litellm_params: dict,
    headers: dict,
) -> dict:
    """构建 OpenAI 格式的请求体"""
    messages = self._transform_messages(messages=messages, model=model)
    messages, tools = self.remove_cache_control_flag_from_messages_and_tools(...)
    
    if tools is not None and len(tools) > 0:
        optional_params["tools"] = tools
    
    optional_params.pop("max_retries", None)
    
    return {
        "model": model,
        "messages": messages,
        **optional_params,
    }
```

**Anthropic 示例**（消息格式转换）：`litellm/llms/anthropic/chat/transformation.py:1172-1231`

```python
def translate_system_message(
    self, messages: List[AllMessageValues]
) -> List[AnthropicSystemMessageContent]:
    """
    将 OpenAI 系统消息转换为 Anthropic 格式
    
    OpenAI: 系统消息在 messages 列表中，role="system"
    Anthropic: 系统消息通过独立的 system 参数传递
    """
    system_prompt_indices = []
    anthropic_system_message_list = []
    
    for idx, message in enumerate(messages):
        if message["role"] == "system":
            system_prompt_indices.append(idx)
            # 转换为 Anthropic 格式的系统消息
            if isinstance(message["content"], str):
                anthropic_system_message_content = AnthropicSystemMessageContent(
                    type="text",
                    text=message["content"],
                )
                anthropic_system_message_list.append(anthropic_system_message_content)
            elif isinstance(message["content"], list):
                # 处理多模态系统消息
                for _content in message["content"]:
                    anthropic_system_message_content = AnthropicSystemMessageContent(
                        type=_content.get("type"),
                        text=_content.get("text"),
                    )
                    anthropic_system_message_list.append(anthropic_system_message_content)
    
    # 从原消息列表中移除系统消息
    for idx in reversed(system_prompt_indices):
        del messages[idx]
    
    return anthropic_system_message_list
```

---

## 5. 响应结构归一化

### 5.1 统一响应类型：ModelResponse

LiteLLM 定义了统一的 `ModelResponse` 类型，所有供应商的响应都转换为此格式。

**关键代码位置**：`litellm/types/utils.py`（简化示意）

```python
class ModelResponse(BaseModel):
    """统一的模型响应格式"""
    id: Optional[str] = None
    object: str = "chat.completion"
    created: Optional[int] = None
    model: Optional[str] = None
    choices: List[Choices] = []
    usage: Optional[Usage] = None
    _response_ms: Optional[int] = None
    _hidden_params: Optional[HiddenParams] = None

class Choices(BaseModel):
    finish_reason: Optional[str] = None
    index: Optional[int] = None
    message: Optional[Message] = None
    logprobs: Optional[Any] = None

class Message(BaseModel):
    role: Optional[str] = None
    content: Optional[Union[str, List]] = None
    tool_calls: Optional[List[ChatCompletionMessageToolCall]] = None
    reasoning_content: Optional[str] = None  # 思考过程内容
```

### 5.2 响应转换：transform_response()

#### OpenAI 响应转换（最小转换）

**关键代码位置**：`litellm/llms/openai/chat/gpt_transformation.py:605-654`

```python
def transform_response(
    self,
    model: str,
    raw_response: httpx.Response,
    model_response: ModelResponse,
    logging_obj: LiteLLMLoggingObj,
    request_data: dict,
    messages: List[AllMessageValues],
    optional_params: dict,
    litellm_params: dict,
    encoding: Any,
    api_key: Optional[str] = None,
    json_mode: Optional[bool] = None,
) -> ModelResponse:
    """OpenAI 响应直接转换为 ModelResponse"""
    
    # 日志记录
    logging_obj.post_call(...)
    
    # 解析 JSON 响应
    try:
        completion_response = raw_response.json()
    except Exception as e:
        raise OpenAIError(...)
    
    # 转换为 ModelResponse
    raw_response_headers = dict(raw_response.headers)
    final_response_obj = convert_to_model_response_object(
        response_object=completion_response,
        model_response_object=model_response,
        hidden_params={"headers": raw_response_headers},
        _response_headers=raw_response_headers,
    )
    
    return cast(ModelResponse, final_response_obj)
```

#### Anthropic 响应转换（复杂格式映射）

Anthropic 的响应格式与 OpenAI 差异较大，需要进行多方面转换：

**关键代码位置**：`litellm/llms/anthropic/chat/transformation.py`（相关方法）

1. **工具调用响应转换**：

```python
@staticmethod
def convert_tool_use_to_openai_format(
    anthropic_tool_content: Dict[str, Any],
    index: int,
) -> ChatCompletionToolCallChunk:
    """
    将 Anthropic 工具调用响应转换为 OpenAI 格式
    
    Anthropic 格式:
    {
      "type": "tool_use",
      "id": "toolu_012345",
      "name": "get_weather",
      "input": {"location": "北京"}
    }
    
    OpenAI 格式:
    {
      "id": "toolu_012345",
      "type": "function",
      "function": {
        "name": "get_weather",
        "arguments": "{\"location\": \"北京\"}"
      },
      "index": 0
    }
    """
    tool_call = ChatCompletionToolCallChunk(
        id=anthropic_tool_content["id"],
        type="function",
        function=ChatCompletionToolCallFunctionChunk(
            name=anthropic_tool_content["name"],
            arguments=json.dumps(anthropic_tool_content["input"]),
        ),
        index=index,
    )
    return tool_call
```

2. **结束原因映射**：

```python
def _get_finish_reason(self, message: Message, received_finish_reason: str) -> str:
    """将 Anthropic 结束原因映射为 OpenAI 格式"""
    if message.tool_calls is not None:
        return "tool_calls"
    else:
        return received_finish_reason
```

### 5.3 响应转换流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    原始响应 (Raw Response)                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  OpenAI 响应    │  │  Anthropic 响应 │  │  Bedrock 响应   │ │
│  │  (原生格式)     │  │  (原生格式)     │  │  (原生格式)     │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              transform_response() 方法                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  1. 解析 JSON 响应                                        │    │
│  │  2. 消息格式转换 (如 Anthropic tool_use → OpenAI tool_calls)│    │
│  │  3. 结束原因映射 (如 Anthropic "tool_use" → "tool_calls") │    │
│  │  4. 思考内容提取 (reasoning_content)                       │    │
│  │  5. 构建统一的 ModelResponse 对象                          │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    统一响应 (ModelResponse)                       │
│  {                                                               │
│    "id": "chatcmpl-xxx",                                         │
│    "object": "chat.completion",                                  │
│    "created": 1700000000,                                        │
│    "model": "gpt-4",                                             │
│    "choices": [                                                  │
│      {                                                            │
│        "index": 0,                                                │
│        "message": {                                               │
│          "role": "assistant",                                     │
│          "content": "你好！",                                     │
│          "tool_calls": [...],  // 统一格式                       │
│          "reasoning_content": "..."  // 思考过程                 │
│        },                                                         │
│        "finish_reason": "stop"                                   │
│      }                                                            │
│    ],                                                             │
│    "usage": {                                                     │
│      "prompt_tokens": 10,                                         │
│      "completion_tokens": 5,                                      │
│      "total_tokens": 15                                           │
│    }                                                              │
│  }                                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. 异常类型归一化

### 6.1 统一异常类型体系

LiteLLM 定义了一套统一的异常类型，继承自 OpenAI 的异常类型：

**关键代码位置**：`litellm/exceptions.py`

```python
# 主要异常类型
class APIError(OpenAIError): ...
class APIConnectionError(APIError): ...
class AuthenticationError(APIError): ...
class BadRequestError(APIError): ...
class NotFoundError(APIError): ...
class PermissionDeniedError(APIError): ...
class RateLimitError(APIError): ...
class Timeout(APIError): ...
class BadGatewayError(APIError): ...
class ServiceUnavailableError(APIError): ...
class InternalServerError(APIError): ...
class UnprocessableEntityError(APIError): ...

# 业务特定异常
class ContextWindowExceededError(BadRequestError): ...
class ContentPolicyViolationError(BadRequestError): ...
class BudgetExceededError(BadRequestError): ...
class RouterNotFoundError(NotFoundError): ...
```

### 6.2 异常映射核心函数：exception_type()

**关键代码位置**：`litellm/litellm_core_utils/exception_mapping_utils.py:235`

```python
def exception_type(
    model,
    original_exception,
    custom_llm_provider,
    completion_kwargs={},
    extra_kwargs={},
):
    """
    将各供应商的异常映射为统一的 LiteLLM 异常类型
    
    映射策略:
    1. 如果已经是 LiteLLM 异常类型，直接返回
    2. 按供应商类型进行特定映射
    3. 按 HTTP 状态码映射
    4. 按错误字符串模式匹配
    """
    
    # 1. 已经是 LiteLLM 异常类型，直接返回
    if any(
        isinstance(original_exception, exc_type)
        for exc_type in litellm.LITELLM_EXCEPTION_TYPES
    ):
        return original_exception
    
    # 提取错误信息
    error_str = str(original_exception)
    
    # 2. 通用超时错误检测
    if (
        "Request Timeout Error" in error_str
        or "Request timed out" in error_str
        or "Timed out generating response" in error_str
        or "The read operation timed out" in error_str
    ):
        raise Timeout(...)
    
    # 3. 按供应商类型进行特定映射
    if (
        custom_llm_provider == "openai"
        or custom_llm_provider == "text-completion-openai"
        or custom_llm_provider == "custom_openai"
        or custom_llm_provider in litellm.openai_compatible_providers
    ):
        # OpenAI 兼容供应商异常映射
        message = get_error_message(error_obj=original_exception)
        
        # 速率限制错误
        if ExceptionCheckers.is_error_str_rate_limit(error_str):
            raise RateLimitError(...)
        
        # 上下文窗口超限错误
        elif ExceptionCheckers.is_error_str_context_window_exceeded(error_str):
            raise ContextWindowExceededError(...)
        
        # 模型未找到
        elif "invalid_request_error" in error_str and "model_not_found" in error_str:
            raise NotFoundError(...)
        
        # 内容策略违规
        elif (
            "invalid_request_error" in error_str
            and "content_policy_violation" in error_str
        ) or (
            "request was rejected as a result of the safety system"
            in error_str.lower()
        ):
            raise ContentPolicyViolationError(...)
        
        # 按 HTTP 状态码映射
        elif hasattr(original_exception, "status_code"):
            if original_exception.status_code == 400:
                raise BadRequestError(...)
            elif original_exception.status_code == 401:
                raise AuthenticationError(...)
            elif original_exception.status_code == 404:
                raise NotFoundError(...)
            elif original_exception.status_code == 429:
                raise RateLimitError(...)
            elif original_exception.status_code == 500:
                raise InternalServerError(...)
            # ... 更多状态码映射
    
    # 4. Anthropic 供应商特定映射
    elif custom_llm_provider == "anthropic" or custom_llm_provider == "anthropic_text":
        if "prompt is too long" in error_str or "prompt: length" in error_str:
            raise ContextWindowExceededError(...)
        elif "overloaded_error" in error_str or "Overloaded" in error_str:
            raise InternalServerError(...)
        elif "Invalid API Key" in error_str:
            raise AuthenticationError(...)
        elif "content filtering policy" in error_str:
            raise ContentPolicyViolationError(...)
        # ... 按状态码映射
    
    # 5. Bedrock、Replicate 等其他供应商映射
    elif custom_llm_provider == "bedrock":
        if "too many tokens" in error_str or "Input is too long" in error_str:
            raise ContextWindowExceededError(...)
        # ...
    
    # ... 更多供应商特定映射
```

### 6.3 错误条件检测器：ExceptionCheckers

**关键代码位置**：`litellm/litellm_core_utils/exception_mapping_utils.py:30`

```python
class ExceptionCheckers:
    """
    辅助类，用于检测异常字符串中的各种错误条件
    """
    
    @staticmethod
    def is_error_str_rate_limit(error_str: str) -> bool:
        """检测是否为速率限制错误"""
        if not isinstance(error_str, str):
            return False
        
        # 状态码 429
        if re.search(r"\b429\b", error_str):
            return True
        
        _error_str_lower = error_str.lower()
        
        # 关键字匹配: "rate limit", "rate-limit", "rate_limit"
        if re.search(r"rate[\s_\-]*limit", _error_str_lower):
            return True
        
        # Mistral 特定错误
        if "service tier capacity exceeded" in _error_str_lower:
            return True
        
        return False
    
    @staticmethod
    def is_error_str_context_window_exceeded(error_str: str) -> bool:
        """检测是否为上下文窗口超限错误"""
        _error_str_lowercase = error_str.lower()
        
        # 排除参数验证错误 (如 OpenAI "user" 参数长度限制)
        if "string_above_max_length" in _error_str_lowercase:
            return False
        if (
            "invalid 'user'" in _error_str_lowercase
            and "string too long" in _error_str_lowercase
        ):
            return False
        
        # 已知的上下文超限错误子串
        known_exception_substrings = [
            "exceed context limit",
            "this model's maximum context length is",
            "string too long. expected a string with maximum length",
            "model's maximum context limit",
            "is longer than the model's context length",
            "input tokens exceed the configured limit",
            "`inputs` tokens + `max_new_tokens` must be",
            "exceeds the maximum number of tokens allowed",
        ]
        for substring in known_exception_substrings:
            if substring in _error_str_lowercase:
                return True
        
        # Cerebras 特定模式
        if (
            "current length is" in _error_str_lowercase
            and "while limit is" in _error_str_lowercase
        ):
            return True
        
        return False
    
    @staticmethod
    def is_azure_content_policy_violation_error(error_str: str) -> bool:
        """检测 Azure 内容策略违规错误"""
        _lower = error_str.lower()
        known_exception_substrings = [
            "content_policy_violation",
            "responsibleaipolicyviolation",
            "the response was filtered due to the prompt triggering azure openai's content management",
            "your task failed as a result of our safety system",
            "the model produced invalid content",
            "content_filter_policy",
            "your request was rejected as a result of our safety system",
        ]
        for substring in known_exception_substrings:
            if substring in _lower:
                return True
        return False
```

### 6.4 异常映射流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    原始异常 (Original Exception)                   │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  OpenAIError    │  │  AnthropicError │  │  AWS Error      │ │
│  │  (openai SDK)   │  │  (anthropic SDK)│  │  (boto3)        │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   exception_type() 函数                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  步骤 1: 检查是否已是 LiteLLM 异常类型                      │    │
│  │     if isinstance(e, LiteLLMException): return e         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  步骤 2: 提取错误信息                                      │    │
│  │     error_str = str(original_exception)                   │    │
│  │     message = get_error_message(original_exception)       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  步骤 3: 按模式匹配检测错误类型                            │    │
│  │  ┌───────────────────────────────────────────────────┐  │    │
│  │  │ ExceptionCheckers.is_error_str_rate_limit()       │  │    │
│  │  │ ExceptionCheckers.is_error_str_context_window_    │  │    │
│  │  │   exceeded()                                       │  │    │
│  │  │ ExceptionCheckers.is_azure_content_policy_        │  │    │
│  │  │   violation_error()                                │  │    │
│  │  └───────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  步骤 4: 按供应商类型进行特定映射                          │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │    │
│  │  │ OpenAI 兼容  │  │  Anthropic   │  │  Bedrock     │  │    │
│  │  │  供应商映射  │  │   特定映射   │  │  特定映射    │  │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  步骤 5: 按 HTTP 状态码映射 (兜底)                        │    │
│  │     400 → BadRequestError                                │    │
│  │     401 → AuthenticationError                            │    │
│  │     404 → NotFoundError                                  │    │
│  │     429 → RateLimitError                                 │    │
│  │     500 → InternalServerError                            │    │
│  │     502 → BadGatewayError                                │    │
│  │     503 → ServiceUnavailableError                        │    │
│  │     504 → Timeout                                        │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    统一异常 (LiteLLM Exception)                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  AuthenticationError      (认证失败)                     │    │
│  │  BadRequestError          (请求参数错误)                  │    │
│  │  NotFoundError            (资源不存在)                    │    │
│  │  RateLimitError           (速率限制)                      │    │
│  │  ContextWindowExceededError (上下文超限)                 │    │
│  │  ContentPolicyViolationError (内容策略违规)              │    │
│  │  Timeout                  (超时)                         │    │
│  │  InternalServerError      (服务端错误)                   │    │
│  │  APIConnectionError       (连接错误)                     │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. 流式输出归一化

### 7.1 基类迭代器：BaseModelResponseIterator

**关键代码位置**：`litellm/llms/base_llm/base_model_iterator.py:60`

LiteLLM 使用 `BaseModelResponseIterator` 基类统一处理所有供应商的流式输出：

```python
class BaseModelResponseIterator:
    """
    流式响应迭代器基类
    支持同步迭代 (__next__) 和异步迭代 (__anext__)
    """
    
    def __init__(
        self, streaming_response, sync_stream: bool, json_mode: Optional[bool] = False
    ):
        self.streaming_response = streaming_response
        self.response_iterator = self.streaming_response
        self.json_mode = json_mode
    
    @abstractmethod
    def chunk_parser(self, chunk: dict) -> Union[GenericStreamingChunk, ModelResponseStream]:
        """
        各供应商实现自己的 chunk 解析逻辑
        将原始响应 chunk 转换为统一的 ModelResponseStream 格式
        """
        pass
    
    # SSE 格式解析
    @staticmethod
    def _string_to_dict_parser(str_line: str) -> Optional[dict]:
        """
        解析 SSE (Server-Sent Events) 格式的行
        
        SSE 格式示例:
        data: {"id": "chatcmpl-xxx", "choices": [...]}
        data: [DONE]
        """
        stripped_json_chunk: Optional[dict] = None
        # 移除 "data: " 前缀
        stripped_chunk = litellm.CustomStreamWrapper._strip_sse_data_from_chunk(str_line)
        try:
            if stripped_chunk is not None:
                stripped_json_chunk = json.loads(stripped_chunk)
            else:
                stripped_json_chunk = None
        except json.JSONDecodeError:
            stripped_json_chunk = None
        return stripped_json_chunk
    
    # 同步迭代
    def __next__(self):
        while True:
            try:
                chunk = self.response_iterator.__next__()
            except StopIteration:
                raise StopIteration
            
            # 处理二进制数据
            if isinstance(chunk, bytes):
                str_line = chunk.decode("utf-8")
                index = str_line.find("data:")
                if index != -1:
                    str_line = str_line[index:]
            else:
                str_line = chunk
            
            # 跳过空行 (SSE 流中事件之间的空行)
            if isinstance(str_line, str) and (not str_line or not str_line.strip()):
                continue
            
            return self._handle_string_chunk(str_line=str_line)
    
    # 异步迭代
    async def __anext__(self):
        while True:
            try:
                chunk = await self.async_response_iterator.__anext__()
            except StopAsyncIteration:
                raise StopAsyncIteration
            
            # 相同的处理逻辑...
            return self._handle_string_chunk(str_line=str_line)
    
    def _handle_string_chunk(
        self, str_line: str
    ) -> Union[GenericStreamingChunk, ModelResponseStream]:
        """处理单个字符串 chunk"""
        stripped_json_chunk = BaseModelResponseIterator._string_to_dict_parser(str_line=str_line)
        
        # 检测流结束标记
        if "[DONE]" in str_line:
            return GenericStreamingChunk(
                text="",
                is_finished=True,
                finish_reason="stop",
                usage=None,
                index=0,
                tool_use=None,
            )
        elif stripped_json_chunk:
            # 调用各供应商实现的 chunk_parser
            return self.chunk_parser(chunk=stripped_json_chunk)
        else:
            # 无法解析的 chunk，返回空
            return GenericStreamingChunk(
                text="",
                is_finished=False,
                finish_reason="",
                usage=None,
                index=0,
                tool_use=None,
            )
```

### 7.2 OpenAI 流式处理器：OpenAIChatCompletionStreamingHandler

**关键代码位置**：`litellm/llms/openai/chat/gpt_transformation.py:784`

```python
class OpenAIChatCompletionStreamingHandler(BaseModelResponseIterator):
    """
    OpenAI 流式响应处理器
    """
    
    def _map_reasoning_to_reasoning_content(self, choices: list) -> list:
        """
        将 'reasoning' 字段映射为 'reasoning_content' 字段
        
        某些 OpenAI 兼容供应商 (如 GLM-5, hosted_vllm) 返回 delta.reasoning，
        但 LiteLLM 期望 delta.reasoning_content
        """
        for choice in choices:
            delta = choice.get("delta", {})
            if "reasoning" in delta:
                delta["reasoning_content"] = delta.pop("reasoning")
        return choices
    
    def chunk_parser(self, chunk: dict) -> ModelResponseStream:
        """
        解析 OpenAI 流式 chunk
        
        OpenAI 流式响应格式:
        {
          "id": "chatcmpl-xxx",
          "object": "chat.completion.chunk",
          "created": 1700000000,
          "model": "gpt-4",
          "choices": [
            {
              "index": 0,
              "delta": {
                "role": "assistant",
                "content": "你",
                "reasoning_content": "我需要思考一下..."  // 思考过程
              },
              "finish_reason": null
            }
          ],
          "usage": {...}  // 某些流包含 usage
        }
        """
        try:
            choices = chunk.get("choices", [])
            choices = self._map_reasoning_to_reasoning_content(choices)
            
            kwargs: Dict[str, Any] = {
                "id": chunk.get("id"),
                "object": "chat.completion.chunk",
                "created": chunk.get("created"),
                "model": chunk.get("model"),
                "choices": choices,
            }
            # 某些流包含 usage 信息
            if "usage" in chunk and chunk["usage"] is not None:
                kwargs["usage"] = chunk["usage"]
            
            return ModelResponseStream(**kwargs)
        except Exception as e:
            raise e
```

### 7.3 统一流式响应类型：ModelResponseStream

```python
class ModelResponseStream(BaseModel):
    """统一的流式响应格式"""
    id: Optional[str] = None
    object: str = "chat.completion.chunk"
    created: Optional[int] = None
    model: Optional[str] = None
    choices: List[StreamingChoices] = []
    usage: Optional[Usage] = None

class StreamingChoices(BaseModel):
    finish_reason: Optional[str] = None
    index: Optional[int] = None
    delta: Optional[Delta] = None

class Delta(BaseModel):
    role: Optional[str] = None
    content: Optional[str] = None
    tool_calls: Optional[List[ChatCompletionMessageToolCallChunk]] = None
    reasoning_content: Optional[str] = None  # 思考过程内容
```

### 7.4 流式处理流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    原始流式响应 (Raw Stream)                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  SSE (Server-Sent Events) 格式                            │    │
│  │  data: {"id": "chatcmpl-1", "choices": [...]}           │    │
│  │  data: {"id": "chatcmpl-1", "choices": [...]}           │    │
│  │  data: [DONE]                                             │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│              BaseModelResponseIterator 基类                       │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  1. SSE 格式解析                                          │    │
│  │     - 移除 "data: " 前缀                                  │    │
│  │     - 检测 "[DONE]" 结束标记                              │    │
│  │     - JSON 解析                                           │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  2. 各供应商特定 chunk_parser() 实现                      │    │
│  │  ┌─────────────────┐  ┌─────────────────┐              │    │
│  │  │ OpenAI 实现     │  │ Anthropic 实现  │              │    │
│  │  │  - 直接解析     │  │  - 工具调用转换 │              │    │
│  │  │  - reasoning    │  │  - 结束原因映射 │              │    │
│  │  │    字段映射     │  │                 │              │    │
│  │  └─────────────────┘  └─────────────────┘              │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    统一流式响应 (ModelResponseStream)             │
│  {                                                               │
│    "id": "chatcmpl-xxx",                                         │
│    "object": "chat.completion.chunk",                            │
│    "created": 1700000000,                                        │
│    "model": "gpt-4",                                             │
│    "choices": [                                                  │
│      {                                                            │
│        "index": 0,                                                │
│        "delta": {                                                 │
│          "role": "assistant",                                     │
│          "content": "你",                                         │
│          "reasoning_content": "...",  // 思考过程                │
│          "tool_calls": [...]       // 工具调用                   │
│        },                                                         │
│        "finish_reason": null                                      │
│      }                                                            │
│    ],                                                             │
│    "usage": {...}  // 可选，某些流包含 token 使用量              │
│  }                                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 8. 供应商特有参数透传机制

### 8.1 透传机制概述

LiteLLM 支持两种方式透传供应商特有参数：

1. **`extra_body` 参数**：传递请求体中的额外参数
2. **`extra_headers` 参数**：传递请求头中的额外参数
3. **可选参数直接传递**：通过 `map_openai_params()` 支持的参数

### 8.2 extra_body 透传机制

**关键代码位置**：`litellm/llms/cometapi/chat/transformation.py:44-88`（示例实现）

```python
def map_openai_params(
    self,
    non_default_params: dict,
    optional_params: dict,
    model: str,
    drop_params: bool,
) -> dict:
    """
    CometAPI 参数映射示例
    
    展示如何处理 extra_body 参数
    """
    extra_body: dict[str, Any] = {}
    
    # 示例：供应商特有参数可以通过 extra_body 传递
    #     extra_body["custom_param"] = custom_param
    
    if extra_body:
        # 将 extra_body 合并到映射后的参数中
        mapped_openai_params["extra_body"] = extra_body
    
    # ... 其他参数映射
    
    return mapped_openai_params

def transform_request(
    self,
    model: str,
    messages: List[AllMessageValues],
    optional_params: dict,
    litellm_params: dict,
    headers: dict,
) -> dict:
    """
    在 transform_request 中应用 extra_body
    """
    # 从 optional_params 中提取 extra_body
    extra_body = optional_params.pop("extra_body", {})
    
    # 构建基础响应
    response = {
        "model": model,
        "messages": messages,
        **optional_params,
    }
    
    # 合并 extra_body 参数
    if extra_body:
        response.update(extra_body)
    
    return response
```

### 8.3 视频生成 API 的 extra_body 处理示例

**关键代码位置**：`litellm/llms/openai/videos/transformation.py:241-554`

```python
def _create_videos_body(
    self,
    model: str,
    prompt: str,
    size: Optional[str],
    fps: Optional[int],
    duration: Optional[int],
    resolution: Optional[str],
    aspect_ratio: Optional[str],
    extra_body: Optional[Dict[str, Any]] = None,
) -> Dict[str, Any]:
    """
    构建视频生成请求体
    
    extra_body 参数会被合并到最终请求中
    """
    data = {
        "model": model,
        "prompt": prompt,
    }
    
    # 添加可选参数
    if size:
        data["size"] = size
    if fps:
        data["fps"] = fps
    if duration:
        data["duration"] = duration
    if resolution:
        data["resolution"] = resolution
    if aspect_ratio:
        data["aspect_ratio"] = aspect_ratio
    
    # 合并 extra_body - 供应商特有参数
    if extra_body:
        data.update(extra_body)
    
    return data
```

### 8.4 extra_headers 透传机制

**关键代码位置**：`litellm/main.py:421`（completion 函数签名）

```python
async def acompletion(
    ...
    extra_headers: Optional[dict] = None,  # 额外请求头
    **kwargs,
) -> Union[ModelResponse, CustomStreamWrapper]:
    """
    extra_headers 参数允许用户传递供应商特有的请求头
    
    这些请求头会被合并到最终的 HTTP 请求中
    """
```

### 8.5 透传机制流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    用户调用 (User Call)                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  litellm.completion(                                      │    │
│  │      model="anthropic/claude-3-opus",                    │    │
│  │      messages=[...],                                      │    │
│  │      extra_body={                                         │    │
│  │          "anthropic_beta": "prompt-caching-2024-07-31", │    │
│  │          "custom_param": "value"                          │    │
│  │      },                                                    │    │
│  │      extra_headers={                                       │    │
│  │          "X-Custom-Header": "value"                       │    │
│  │      }                                                     │    │
│  │  )                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    参数收集阶段                                    │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  get_non_default_completion_params()                      │    │
│  │  - 收集所有非默认参数                                      │    │
│  │  - 包括 extra_body, extra_headers                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    参数映射阶段 (map_openai_params)               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  1. 标准 OpenAI 参数 → 供应商格式                          │    │
│  │     tools → anthropic_tools                              │    │
│  │     tool_choice → anthropic_tool_choice                  │    │
│  │     response_format → output_format 或 json_mode tool    │    │
│  │                                                           │    │
│  │  2. extra_body → 暂存到 optional_params["extra_body"]    │    │
│  │  3. extra_headers → 暂存到 optional_params["extra_headers"]│  │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    请求构建阶段 (transform_request)               │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  1. 构建基础请求体                                        │    │
│  │     {                                                      │    │
│  │       "model": "claude-3-opus",                          │    │
│  │       "messages": [...],                                  │    │
│  │       "max_tokens": 1024,                                 │    │
│  │       ...                                                  │    │
│  │     }                                                      │    │
│  │                                                           │    │
│  │  2. 合并 extra_body                                        │    │
│  │     response.update(extra_body)                           │    │
│  │     结果:                                                   │    │
│  │     {                                                      │    │
│  │       "model": "claude-3-opus",                          │    │
│  │       "messages": [...],                                  │    │
│  │       "max_tokens": 1024,                                 │    │
│  │       "anthropic_beta": "prompt-caching-2024-07-31",    │    │
│  │       "custom_param": "value"                             │    │
│  │     }                                                      │    │
│  │                                                           │    │
│  │  3. 合并 extra_headers 到请求头                            │    │
│  │     headers.update(extra_headers)                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP 请求发送                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  POST https://api.anthropic.com/v1/messages              │    │
│  │                                                           │    │
│  │  Headers:                                                 │    │
│  │    x-api-key: sk-ant-xxx                                  │    │
│  │    anthropic-version: 2023-06-01                         │    │
│  │    X-Custom-Header: value  ← extra_headers                │    │
│  │                                                           │    │
│  │  Body:                                                    │    │
│  │    {                                                      │    │
│  │      "model": "claude-3-opus-20240229",                 │    │
│  │      "max_tokens": 1024,                                 │    │
│  │      "messages": [...],                                   │    │
│  │      "anthropic_beta": "prompt-caching-2024-07-31",  ← │    │
│  │      "custom_param": "value"  ← extra_body               │    │
│  │    }                                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. 架构总结与设计亮点

### 9.1 核心设计模式

LiteLLM 的多供应商适配架构巧妙运用了多种经典设计模式：

| 设计模式 | 应用场景 | 关键代码 |
|---------|---------|---------|
| **抽象工厂** | 供应商配置类的创建 | `BaseConfig` 抽象类 |
| **模板方法** | 参数映射、请求转换流程 | `map_openai_params()`, `transform_request()` |
| **策略模式** | 不同供应商的差异化实现 | `OpenAIGPTConfig`, `AnthropicConfig`, `BedrockConfig` 等 |
| **适配器模式** | API 格式转换 | `transform_request()`, `transform_response()` |
| **责任链模式** | 异常类型映射 | `exception_type()` 中的多层检测 |
| **迭代器模式** | 流式响应处理 | `BaseModelResponseIterator` |

### 9.2 架构层次总结

```
┌──────────────────────────────────────────────────────────────────────┐
│                         应用层 (Application)                           │
│  litellm.completion() / acompletion()                                │
│  - 统一的 OpenAI 兼容接口                                             │
│  - 简化的参数签名                                                     │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          路由层 (Routing)                              │
│  get_llm_provider()                                                   │
│  ┌─────────────────┬─────────────────┬─────────────────┐           │
│  │ 模型前缀路由    │ 已知模型匹配    │ API Base 匹配   │           │
│  │ "anthropic/..." │ "gpt-4"        │ "api.groq.com" │           │
│  └─────────────────┴─────────────────┴─────────────────┘           │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        转换层 (Transformation)                         │
│  BaseConfig (抽象基类)                                                │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  参数映射 (map_openai_params)                                   │ │
│  │  请求转换 (transform_request)                                   │ │
│  │  响应转换 (transform_response)                                  │ │
│  │  环境验证 (validate_environment)                                │ │
│  └────────────────────────────────────────────────────────────────┘ │
│  具体实现:                                                            │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐               │
│  │ OpenAI 实现  │ │ Anthropic    │ │ Bedrock/     │               │
│  │ (最小转换)   │ │ 实现(复杂)  │ │ VertexAI 等 │               │
│  └──────────────┘ └──────────────┘ └──────────────┘               │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       归一化层 (Normalization)                         │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐               │
│  │ 参数归一化   │ │ 响应归一化   │ │ 异常归一化   │               │
│  │ map_openai_  │ │ ModelResponse│ │ exception_   │               │
│  │ params()     │ │ / Stream     │ │ type()       │               │
│  └──────────────┘ └──────────────┘ └──────────────┘               │
│  ┌──────────────┐ ┌──────────────┐                                │
│  │ 流式归一化   │ │ 透传机制     │                                │
│  │ BaseModel    │ │ extra_body/  │                                │
│  │ ResponseIte- │ │ extra_headers│                                │
│  │ rator        │ │              │                                │
│  └──────────────┘ └──────────────┘                                │
└──────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          传输层 (Transport)                             │
│  HTTPHandler / AsyncHTTPHandler                                       │
│  - httpx 客户端封装                                                   │
│  - 同步/异步支持                                                       │
│  - 超时、重试、连接池管理                                              │
└──────────────────────────────────────────────────────────────────────┘
```

### 9.3 关键设计亮点

#### 1. 渐进式转换策略

LiteLLM 采用**渐进式转换**策略，对不同供应商的转换程度不同：

- **OpenAI 兼容供应商**：最小化转换，参数直接传递
- **非 OpenAI 供应商**：完整的格式转换（消息、工具、响应）
- **混合模式**：部分参数支持直接透传 (`extra_body`)

这种策略平衡了代码复用和灵活性。

#### 2. 安全的端点匹配

**关键代码**：`_endpoint_matches_api_base()`

LiteLLM 没有使用简单的子字符串匹配来识别 API 端点，而是采用了 URL 解析的安全方式：

```python
# ❌ 不安全的方式（可能被利用）
if "api.groq.com" in api_base:
    # 攻击者可以构造: "https://attacker.com/api.groq.com/..."
    # 导致 API Key 被发送到攻击者服务器

# ✅ 安全的方式（LiteLLM 采用）
def _endpoint_matches_api_base(endpoint: str, api_base: str) -> bool:
    parsed_endpoint = _parse(endpoint)
    parsed_url = _parse(api_base)
    
    # 主机名必须精确匹配
    if parsed_endpoint.hostname != parsed_url.hostname:
        return False
    
    # 路径必须以端点路径开头
    return url_path.startswith(endpoint_path)
```

这种设计防止了 URL 注入攻击，保护用户的 API Key 安全。

#### 3. 可扩展的异常检测

`ExceptionCheckers` 类采用了**组合式**的异常检测策略：

- **状态码检测**：基于 HTTP 状态码的映射
- **字符串模式匹配**：基于错误消息中的关键字
- **供应商特定规则**：不同供应商有不同的错误消息格式

这种设计使得异常检测可以独立于供应商实现进行扩展。

#### 4. 统一的流式抽象

`BaseModelResponseIterator` 基类统一处理了：

- **SSE 格式解析**：所有供应商都使用 SSE 或类似格式
- **同步/异步双支持**：`__next__` 和 `__anext__` 方法
- **结束标记检测**：`[DONE]` 标记的统一处理
- **Chunk 解析抽象**：各供应商实现 `chunk_parser()` 方法

### 9.4 关键文件索引

| 功能模块 | 文件路径 | 关键类/函数 |
|---------|---------|------------|
| 基类抽象 | `litellm/llms/base_llm/chat/transformation.py` | `BaseConfig` |
| 模型路由 | `litellm/litellm_core_utils/get_llm_provider_logic.py` | `get_llm_provider()`, `_endpoint_matches_api_base()` |
| OpenAI 实现 | `litellm/llms/openai/chat/gpt_transformation.py` | `OpenAIGPTConfig` |
| Anthropic 实现 | `litellm/llms/anthropic/chat/transformation.py` | `AnthropicConfig` |
| 异常映射 | `litellm/litellm_core_utils/exception_mapping_utils.py` | `exception_type()`, `ExceptionCheckers` |
| 流式处理 | `litellm/llms/base_llm/base_model_iterator.py` | `BaseModelResponseIterator` |
| 主入口 | `litellm/main.py` | `completion()`, `acompletion()` |
| 异常类型 | `litellm/exceptions.py` | `RateLimitError`, `ContextWindowExceededError` 等 |

### 9.5 扩展新供应商的步骤

基于架构分析，添加新供应商需要以下步骤：

1. **创建配置类**：继承 `BaseConfig`，实现所有抽象方法
2. **添加模型列表**：将新供应商的模型添加到对应的模型列表中
3. **更新路由逻辑**：在 `get_llm_provider()` 中添加供应商识别逻辑
4. **实现异常映射**：在 `exception_type()` 中添加供应商特定的异常检测
5. **添加测试**：在 `tests/` 目录中添加单元测试和集成测试

### 9.6 总结

LiteLLM 的多模型供应商适配架构展现了优秀的软件设计：

1. **面向抽象编程**：通过 `BaseConfig` 抽象类定义统一接口
2. **多态实现**：各供应商根据自身 API 特性实现差异化逻辑
3. **分层架构**：路由层、转换层、归一化层职责清晰
4. **安全优先**：URL 匹配、API Key 处理等方面都考虑了安全性
5. **可扩展性**：新供应商的添加遵循明确的扩展点

这种架构使得 LiteLLM 能够支持 100+ 不同的 AI 服务供应商，同时对外提供一致的 OpenAI 兼容接口，大大简化了多模型应用的开发和维护。

---

## 附录

### A. 核心类继承关系

```
BaseConfig (ABC)
├── OpenAIGPTConfig
│   ├── AzureOpenAIConfig
│   └── OpenAILikeConfig
├── AnthropicConfig
│   ├── BedrockAnthropicConfig
│   └── VertexAIAnthropicConfig
├── BedrockConverseConfig
├── VertexAIConfig
├── GeminiConfig
├── CohereConfig
├── MistralConfig
└── ... (100+ 其他供应商)
```

### B. 异常类型层次结构

```
APIError (基类)
├── APIConnectionError
├── AuthenticationError
├── BadRequestError
│   ├── ContextWindowExceededError
│   ├── ContentPolicyViolationError
│   └── BudgetExceededError
├── NotFoundError
├── PermissionDeniedError
├── RateLimitError
├── Timeout
├── BadGatewayError
├── ServiceUnavailableError
├── InternalServerError
└── UnprocessableEntityError
```

### C. 参考资源

- LiteLLM 官方文档：https://docs.litellm.ai/
- LiteLLM GitHub：https://github.com/BerriAI/litellm
- OpenAI API 文档：https://platform.openai.com/docs/api-reference