# LiteLLM 跨 Provider 流式响应归一化路径分析

> 分析日期：2026-05-02  
> 分析目标：梳理 LiteLLM 中各 provider 流式响应格式差异的抹平机制、归一化后数据块传递路径、以及 streaming 回调与非流式路径的关键分叉点

---

## 1. 概述

LiteLLM 的核心价值之一是为 100+ 种 LLM provider 提供统一的 OpenAI 兼容接口。对于流式响应（Streaming），LiteLLM 通过一套精心设计的归一化架构，将各 provider 迥异的流式格式统一转换为 OpenAI SSE (Server-Sent Events) 格式。

### 1.1 核心设计目标

1. **格式归一化**：将 Anthropic、Bedrock、Vertex AI 等 provider 的原生流式格式统一转换为 OpenAI 兼容的 `ModelResponseStream`
2. **接口一致性**：同步/异步迭代器使用相同的数据模型
3. **回调支持**：在流式过程中支持自定义钩子（Hooks）和回调（Callbacks）
4. **错误处理**：统一的异常处理和重试机制

---

## 2. 核心组件架构

### 2.1 主要类与模块

| 组件 | 文件位置 | 职责 |
|------|----------|------|
| `CustomStreamWrapper` | `litellm/litellm_core_utils/streaming_handler.py:98` | 核心流式包装器，统一处理所有 provider 的流式响应 |
| `ModelResponseIterator` | `litellm/llms/anthropic/chat/handler.py:522` | Anthropic 流式迭代器，转换 Anthropic 事件流 |
| `ResponsesAPIStreamingIterator` | `litellm/responses/streaming_iterator.py:452` | OpenAI Responses API 流式迭代器 |
| `ChunkProcessor` | `litellm/litellm_core_utils/streaming_chunk_builder_utils.py:35` | 将多个流式块聚合为完整响应 |
| `GenericStreamingChunk` | `litellm/types/utils.py:274` | 中间归一化格式（TypedDict） |
| `ModelResponseStream` | `litellm/types/utils.py` | 最终 OpenAI 兼容流式响应格式 |

### 2.2 整体架构流程图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户调用层                                          │
│  litellm.completion(stream=True) / litellm.acompletion(stream=True)        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           主入口分发层 (main.py)                              │
│  1. 解析 stream 参数                                                          │
│  2. 根据 custom_llm_provider 分发到对应 provider handler                     │
│  3. 流式路径: 调用 xxx_stream_function → 返回 CustomStreamWrapper             │
│  4. 非流式路径: 调用 xxx_function → 返回 ModelResponse                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Provider Handler 层                                    │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────────┐   │
│  │  OpenAI     │  │ Anthropic   │  │  Bedrock    │  │ Vertex AI/Gemini │   │
│  │  (原生SSE)  │  │ (事件流转换) │  │ (AWS事件流) │  │  (ProtoBuf)     │   │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └────────┬─────────┘   │
│         │                │                │                  │               │
│         ▼                ▼                ▼                  ▼               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │              ModelResponseIterator (各 provider 特有)                  │  │
│  │  - 将原生流式格式转换为 GenericStreamingChunk 或 ModelResponseStream   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    归一化层 (CustomStreamWrapper)                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  chunk_creator() - 核心归一化方法 (streaming_handler.py:1121)        │  │
│  │  - 处理 GenericStreamingChunk 格式                                      │  │
│  │  - 处理各 provider 特有格式 (handle_xxx_chunk 系列方法)                 │  │
│  │  - 最终输出 ModelResponseStream                                          │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                      │                                        │
│                                      ▼                                        │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  return_processed_chunk_logic() - 后处理 (streaming_handler.py:943)  │  │
│  │  - 合并推理内容 (thinking/reasoning content)                           │  │
│  │  - 处理特殊 tokens (SageMaker/HF)                                       │  │
│  │  - 检查重复 chunk (防无限循环)                                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          回调与 Hooks 层                                       │
│  - _call_post_streaming_deployment_hook()                                    │
│  - async_post_call_streaming_iterator_hook (Proxy 层)                        │
│  - logging.success_handler / async_success_handler                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 各 Provider 流式响应格式差异

### 3.1 格式对比表

| Provider | 原生流式格式 | 转换方式 | 关键文件 |
|----------|-------------|----------|----------|
| **OpenAI** | SSE (Server-Sent Events) + `ChatCompletionChunk` 对象 | 直接使用（无需转换） | `litellm/llms/openai/openai.py` |
| **Anthropic** | 事件流 (message_start, content_block_start, content_block_delta, content_block_stop, message_delta, message_stop) | `ModelResponseIterator` → `GenericStreamingChunk` | `litellm/llms/anthropic/chat/handler.py:522` |
| **Bedrock (Converse)** | AWS EventStream 格式 (contentBlockDelta, messageStop 等) | `ModelResponseIterator` (Bedrock 版本) | `litellm/llms/bedrock/common_utils.py` |
| **Bedrock (Invoke)** | 各模型特有 JSON 格式 (Anthropic/Cohere/Titan 等) | 按模型类型分别解析 | `litellm/llms/bedrock/chat/invoke_handler.py` |
| **Vertex AI/Gemini** | Protobuf 格式 (`candidates[0].content.parts`) | 直接在 `chunk_creator()` 中解析 | `litellm/litellm_core_utils/streaming_handler.py:1236` |
| **Cohere** | 事件流格式 | 转换为 `GenericStreamingChunk` | `litellm/llms/cohere/chat/handler.py` |
| **Mistral** | OpenAI 兼容 SSE | 直接使用 | `litellm/llms/mistral/chat/handler.py` |

### 3.2 Anthropic 流式事件类型详解

Anthropic 使用完全不同的事件类型系统，`ModelResponseIterator` 的 `_content_block_delta_helper` 方法负责转换：

```python
# Anthropic 原生事件类型 (litellm/llms/anthropic/chat/handler.py:596)
- "message_start"    → 包含 usage 信息
- "content_block_start" → 标记新内容块开始
- "content_block_delta" → 实际内容增量
  ├── "text_delta" → 普通文本
  ├── "input_json_delta" → 工具调用参数
  ├── "thinking_delta" → 推理过程文本
  └── "signature" → 推理签名 (用于签名验证)
- "content_block_stop" → 内容块结束
- "message_delta" → 消息级增量
- "message_stop" → 消息结束，包含最终 usage
```

转换逻辑示例 (`_content_block_delta_helper`):

```python
# litellm/llms/anthropic/chat/handler.py:596-650
def _content_block_delta_helper(self, chunk: dict) -> Tuple[
    str,
    Optional[ChatCompletionToolCallChunk],
    List[Union[ChatCompletionThinkingBlock, ChatCompletionRedactedThinkingBlock]],
    Dict[str, Any],
]:
    text = ""
    tool_use: Optional[ChatCompletionToolCallChunk] = None
    thinking_blocks = []
    # ...
    if "text" in content_block["delta"]:
        text = content_block["delta"]["text"]
    elif "partial_json" in content_block["delta"]:
        # 工具调用参数
        tool_use = {
            "id": None,
            "type": "function", 
            "function": {
                "name": None,
                "arguments": content_block["delta"]["partial_json"],
            },
            "index": self.tool_index,
        }
    elif "thinking" in content_block["delta"] or "signature" in content_block["delta"]:
        # 推理内容块
        thinking_blocks = [
            ChatCompletionThinkingBlock(
                type="thinking",
                thinking=content_block["delta"].get("thinking") or "",
                signature=str(content_block["delta"].get("signature") or ""),
            )
        ]
```

### 3.3 Vertex AI/Gemini 流式格式

Vertex AI 使用 Protobuf 格式，`chunk_creator()` 中直接处理：

```python
# litellm/litellm_core_utils/streaming_handler.py:1236-1303
elif self.custom_llm_provider == "vertex_ai" and not isinstance(
    chunk, ModelResponseStream
):
    import proto
    if hasattr(chunk, "candidates") is True:
        try:
            completion_obj["content"] = chunk.text  # 直接访问 text 属性
        except Exception as e:
            if "Part has no text." in str(e):
                # 函数调用处理
                function_call = chunk.candidates[0].content.parts[0].function_call
                args_dict = {}
                for key, val in function_call.args.items():
                    if isinstance(val, proto.marshal.collections.repeated.RepeatedComposite):
                        args_dict[key] = [v for v in val]
                    else:
                        args_dict[key] = val
                
                _delta_obj = litellm.utils.Delta(
                    content=None,
                    tool_calls=[{
                        "id": f"call_{str(uuid.uuid4())}",
                        "function": {
                            "arguments": json.dumps(args_dict),
                            "name": function_call.name,
                        },
                        "type": "function",
                    }],
                )
```

---

## 4. 归一化处理流程（三步转换）

### 4.1 第一步：Provider 特有迭代器 → 中间格式

各 provider 通过自己的 `ModelResponseIterator` 将原生流式格式转换为 `GenericStreamingChunk`：

```python
# GenericStreamingChunk 定义 (litellm/types/utils.py:274)
class GenericStreamingChunk(TypedDict, total=False):
    text: Required[str]                    # 文本内容
    tool_use: Optional[ChatCompletionToolCallChunk]  # 工具调用
    is_finished: Required[bool]            # 是否结束
    finish_reason: Required[str]           # 结束原因
    usage: Required[Optional[ChatCompletionUsageBlock]]  # Token 使用量
    provider_specific_fields: Optional[Dict]  # 特有字段扩展
```

### 4.2 第二步：`CustomStreamWrapper.chunk_creator()` → 统一格式

这是**最核心的归一化方法**，处理所有 provider 的格式差异：

```python
# litellm/litellm_core_utils/streaming_handler.py:1121-1420
def chunk_creator(self, chunk: Any):
    # 1. 初始化响应对象
    model_response = self.model_response_creator()
    completion_obj: Dict[str, Any] = {"content": ""}
    
    # 2. 分支处理：按 provider 类型分发
    if (
        isinstance(chunk, dict)
        and generic_chunk_has_all_required_fields(chunk=chunk)
    ) or (
        self.custom_llm_provider
        and self.custom_llm_provider in litellm._custom_providers
    ):
        # 路径 A: GenericStreamingChunk 格式 (Anthropic/Bedrock 等)
        anthropic_response_obj: GChunk = cast(GChunk, chunk)
        completion_obj["content"] = anthropic_response_obj["text"]
        if anthropic_response_obj["is_finished"]:
            self.received_finish_reason = anthropic_response_obj["finish_reason"]
        if anthropic_response_obj["usage"] is not None:
            setattr(model_response, "usage", litellm.Usage(**anthropic_response_obj["usage"]))
        if anthropic_response_obj["tool_use"] is not None:
            completion_obj["tool_calls"] = [anthropic_response_obj["tool_use"]]
        response_obj = cast(Dict[str, Any], anthropic_response_obj)
    
    elif self.model == "replicate" or self.custom_llm_provider == "replicate":
        # 路径 B: Replicate 特有格式
        response_obj = self.handle_replicate_chunk(chunk)
        completion_obj["content"] = response_obj["text"]
    
    elif self.custom_llm_provider == "vertex_ai" and not isinstance(chunk, ModelResponseStream):
        # 路径 C: Vertex AI Protobuf 格式
        # ... 见 3.3 节
    
    else:
        # 路径 D: OpenAI/Azure 原生格式 (默认)
        response_obj = self.handle_openai_chat_completion_chunk(chunk)
        completion_obj["content"] = response_obj["text"]
```

### 4.3 第三步：`return_processed_chunk_logic()` → 后处理

对归一化后的 chunk 进行最终处理：

```python
# litellm/litellm_core_utils/streaming_handler.py:943-1074
def return_processed_chunk_logic(
    self,
    completion_obj: Dict[str, Any],
    model_response: ModelResponseStream,
    response_obj: Dict[str, Any],
):
    # 1. 检查 chunk 是否非空
    is_chunk_non_empty = self.is_chunk_non_empty(...)
    
    if is_chunk_non_empty:
        # 2. 处理 SageMaker/HF 特殊 tokens (bos/eos)
        hold, model_response_str = self.check_special_tokens(...)
        
        if hold is False:
            # 3. 处理原始 chunk (如果是 OpenAI 格式)
            original_chunk = response_obj.get("original_chunk", None)
            if original_chunk:
                # 保留原始属性
                model_response.system_fingerprint = original_chunk.system_fingerprint
                preserve_upstream_non_openai_attributes(...)
                model_response = self.strip_role_from_delta(model_response)
            else:
                # 组装 delta 对象
                completion_obj["content"] = model_response_str
                if self.sent_first_chunk is False:
                    completion_obj["role"] = "assistant"
                    self.sent_first_chunk = True
                model_response.choices[0].delta = Delta(**completion_obj)
            
            # 4. 可选：合并推理内容块
            self._optional_combine_thinking_block_in_choices(
                model_response=model_response
            )
            return model_response
    
    elif self.received_finish_reason is not None:
        # 处理结束块
        # ...
```

---

## 5. 数据块传递机制与模块间接口

### 5.1 数据流转图

```
┌──────────────┐
│  HTTP 响应流 │
│ (SSE/EventStream)
└──────┬───────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  Provider ModelResponseIterator                          │
│  - 逐行读取 HTTP 响应流                                    │
│  - 解析事件类型 (如 Anthropic 的 event: xxx)              │
│  - 输出: GenericStreamingChunk 或 ModelResponseStream     │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  CustomStreamWrapper (作为 Iterator)                     │
│  - __next__() / __anext__() 逐个获取 chunk                │
│  - 调用 chunk_creator() 进行归一化                         │
│  - 输出: ModelResponseStream                              │
└─────────────────────────────────────────────────────────┘
       │
       ▼
┌─────────────────────────────────────────────────────────┐
│  用户代码层                                               │
│  for chunk in response:                                  │
│      print(chunk.choices[0].delta.content)              │
└─────────────────────────────────────────────────────────┘
```

### 5.2 关键接口定义

#### 5.2.1 `ModelResponseStream` (最终输出格式)

```python
# 基于 OpenAI ChatCompletionChunk
class ModelResponseStream(BaseModel):
    id: str                                    # 响应 ID
    object: Literal["chat.completion.chunk"]   # 固定值
    created: int                                # 创建时间戳
    model: str                                  # 模型名称
    choices: List[StreamingChoices]            # 选择列表
    usage: Optional[Usage]                     # Token 使用量 (stream_options 启用时)
    system_fingerprint: Optional[str]          # 系统指纹
    _hidden_params: Dict[str, Any]             # 内部隐藏参数
```

#### 5.2.2 `StreamingChoices` / `Delta`

```python
class StreamingChoices(BaseModel):
    delta: Delta                # 增量内容
    finish_reason: Optional[str] # 结束原因
    index: int                  # 索引

class Delta(BaseModel):
    role: Optional[str]                     # 角色
    content: Optional[str]                  # 文本内容
    tool_calls: Optional[List[ChatCompletionMessageToolCall]]  # 工具调用
    function_call: Optional[FunctionCall]   # 函数调用 (旧格式)
    reasoning_content: Optional[str]        # 推理内容 (Claude)
    thinking_blocks: Optional[List]         # 推理块 (Gemini)
    annotations: Optional[List]             # 引用标注
```

### 5.3 流式块聚合（非流式场景）

当需要将流式响应聚合为完整响应时，使用 `ChunkProcessor`：

```python
# litellm/litellm_core_utils/streaming_chunk_builder_utils.py:35-731
class ChunkProcessor:
    def __init__(self, chunks: List, messages: Optional[list] = None):
        self.chunks = self._sort_chunks(chunks)  # 按时间排序
    
    def build_base_response(self, chunks: List[Dict[str, Any]]) -> ModelResponse:
        # 从多个 chunk 构建基础 ModelResponse
        id = ChunkProcessor._get_chunk_id(chunks)
        model = ChunkProcessor._get_model_from_chunks(chunks, first_chunk_model)
        # ...
    
    def get_combined_content(self, chunks: List[Dict[str, Any]]) -> str:
        # 合并所有 content delta
        content_list: List[str] = []
        for chunk in chunks:
            delta = chunk["choices"][0].get("delta", {})
            content = delta.get("content", "")
            content_list.append(content)
        return "".join(content_list)
    
    def get_combined_tool_content(self, tool_call_chunks: List) -> List:
        # 合并工具调用参数 (分块的 JSON 需要拼接)
        # ...
    
    def calculate_usage(self, chunks: List, ...) -> Usage:
        # 计算总 Token 使用量
        # ...
```

---

## 6. Streaming 回调与非流式路径关键分叉点

### 6.1 主入口分叉点 (`main.py`)

#### 6.1.1 `stream` 参数解析

```python
# litellm/main.py:1493
optional_param_args = {
    # ...
    "stream": stream,                    # 关键参数
    "stream_options": stream_options,    # 流式选项 (如 include_usage)
    # ...
}
```

#### 6.1.2 Provider Handler 中的分叉

以 Anthropic 为例：

```python
# litellm/llms/anthropic/chat/handler.py:394-447
if acompletion is True:
    if stream is True:
        # 流式路径
        print_verbose("makes async anthropic streaming POST request")
        data["stream"] = stream
        return self.acompletion_stream_function(...)  # → 返回 CustomStreamWrapper
    else:
        # 非流式路径
        return self.acompletion_function(...)  # → 返回 ModelResponse
else:
    if stream is True:
        # 同步流式路径
        data["stream"] = stream
        completion_stream, headers = make_sync_call(...)
        return CustomStreamWrapper(...)
    else:
        # 同步非流式路径
        response = client.post(...)
        return config.transform_response(...)
```

### 6.2 核心分叉逻辑表

| 位置 | 文件 | 关键判断 | 流式路径 | 非流式路径 |
|------|------|----------|----------|------------|
| 1 | `main.py:856` | `if stream is True` | `model_response = ModelResponseStream()` | `model_response = ModelResponse()` |
| 2 | `anthropic/handler.py:396` | `if stream is True` | 调用 `acompletion_stream_function` | 调用 `acompletion_function` |
| 3 | `openai/openai.py` | SDK 原生支持 | 返回 SDK 原生 stream | 调用 SDK 的非流式方法 |
| 4 | `streaming_handler.py:1131` | `isinstance(chunk, GenericStreamingChunk)` | 归一化处理 | 按 provider 类型分发 |

### 6.3 Streaming 回调机制

#### 6.3.1 Core 层回调 (`CustomStreamWrapper`)

```python
# litellm/litellm_core_utils/streaming_handler.py 中的迭代器方法
async def __anext__(self) -> ModelResponseStream:
    # ...
    result = self._process_chunk(chunk)
    # 调用流式部署钩子
    result = await self._call_post_streaming_deployment_hook(chunk=result)
    return result

# litellm/responses/streaming_iterator.py:499-504 (Responses API)
async def __anext__(self) -> ResponsesAPIStreamingResponse:
    # ...
    result = self._process_chunk(chunk)
    result = await self._call_post_streaming_deployment_hook(chunk=result)
    return result
```

#### 6.3.2 部署钩子实现

```python
# litellm/responses/streaming_iterator.py:299-338
async def _call_post_streaming_deployment_hook(self, chunk):
    """
    Allow callbacks to modify streaming chunks before returning
    """
    callbacks = getattr(litellm, "callbacks", None) or []
    for callback in callbacks:
        if hasattr(callback, "async_post_call_streaming_deployment_hook"):
            result = await callback.async_post_call_streaming_deployment_hook(
                request_data=request_data,
                response_chunk=chunk,
                call_type=typed_call_type,
            )
            if result is not None:
                chunk = result
    return chunk
```

#### 6.3.3 Proxy 层回调 (`async_post_call_streaming_iterator_hook`)

```python
# litellm/proxy/utils.py:2262-2324
async def async_post_call_streaming_iterator_hook(
    self,
    response,
    user_api_key_dict: UserAPIKeyAuth,
    request_data: dict,
):
    """
    Proxy 层的流式迭代器钩子 - 支持 Guardrail 和自定义回调
    """
    current_response = response
    for callback in self.callbacks:
        if isinstance(callback, CustomLogger):
            if "async_post_call_streaming_iterator_hook" in type(callback).__dict__:
                current_response = callback.async_post_call_streaming_iterator_hook(
                    user_api_key_dict=user_api_key_dict,
                    response=current_response,
                    request_data=request_data,
                )
    return current_response
```

### 6.4 流式完成后的日志回调

当流式响应结束时，触发成功/失败处理：

```python
# litellm/responses/streaming_iterator.py:521-556
def _handle_logging_completed_response(self):
    """Handle logging for completed responses in async context"""
    # 创建日志副本
    logging_response = self.completed_response
    # ...
    
    # 异步日志处理 (不阻塞主流程)
    asyncio.create_task(
        self.logging_obj.async_success_handler(
            result=logging_response,
            start_time=self.start_time,
            end_time=datetime.now(),
            cache_hit=None,
        )
    )
    
    # 同步日志处理 (线程池)
    executor.submit(
        self.logging_obj.success_handler,
        result=logging_response,
        cache_hit=None,
        start_time=self.start_time,
        end_time=datetime.now(),
    )
    
    self._run_post_success_hooks(end_time=datetime.now())
```

---

## 7. 关键代码位置索引

### 7.1 核心文件速查

| 功能 | 文件路径 | 关键行号 |
|------|----------|----------|
| **主入口分发** | `litellm/main.py` | `completion()` L1060, `acompletion()` L379 |
| **流式包装器** | `litellm/litellm_core_utils/streaming_handler.py` | `CustomStreamWrapper` L98 |
| **核心归一化** | `litellm/litellm_core_utils/streaming_handler.py` | `chunk_creator()` L1121 |
| **后处理逻辑** | `litellm/litellm_core_utils/streaming_handler.py` | `return_processed_chunk_logic()` L943 |
| **Anthropic 转换** | `litellm/llms/anthropic/chat/handler.py` | `ModelResponseIterator` L522 |
| **Responses API 流式** | `litellm/responses/streaming_iterator.py` | `ResponsesAPIStreamingIterator` L452 |
| **Chunk 聚合** | `litellm/litellm_core_utils/streaming_chunk_builder_utils.py` | `ChunkProcessor` L35 |
| **Proxy 流式回调** | `litellm/proxy/utils.py` | `async_post_call_streaming_iterator_hook` L2262 |
| **通用流式格式** | `litellm/types/utils.py` | `GenericStreamingChunk` L274 |

### 7.2 关键方法速查

| 方法 | 功能 | 位置 |
|------|------|------|
| `CustomStreamWrapper.__next__()` | 同步迭代器入口 | `streaming_handler.py` |
| `CustomStreamWrapper.__anext__()` | 异步迭代器入口 | `streaming_handler.py` |
| `chunk_creator()` | 核心归一化方法 | `streaming_handler.py:1121` |
| `_content_block_delta_helper()` | Anthropic 事件转换 | `anthropic/handler.py:596` |
| `handle_openai_chat_completion_chunk()` | OpenAI 格式处理 | `streaming_handler.py:510` |
| `_call_post_streaming_deployment_hook()` | 流式部署钩子 | `streaming_iterator.py:299` |
| `_handle_logging_completed_response()` | 完成日志处理 | `streaming_iterator.py:521` |

---

## 8. 流程图总结

### 8.1 完整调用栈（流式路径）

```
litellm.completion(model="claude-3-opus", stream=True, ...)
  │
  ▼
main.py:completion()
  ├── 解析 stream=True
  ├── 确定 custom_llm_provider="anthropic"
  └── 调用 anthropic_chat_completions.completion(...)
        │
        ▼
anthropic/chat/handler.py:AnthropicChatCompletion.completion()
  ├── 检查 stream=True
  ├── 调用 make_sync_call() / make_call()
  │       ├── HTTP POST with stream=True
  │       └── 返回 ModelResponseIterator
  └── 包装为 CustomStreamWrapper(completion_stream, ...)
        │
        ▼
返回 CustomStreamWrapper 对象
  │
  ▼
用户代码: for chunk in response:
  │
  ▼
streaming_handler.py:CustomStreamWrapper.__next__()
  ├── next(self.completion_stream)  # 从 ModelResponseIterator 获取
  ├── self.chunk_creator(chunk)     # 归一化
  │       ├── 识别为 GenericStreamingChunk
  │       ├── 提取 text, tool_use, usage, finish_reason
  │       └── 组装为 ModelResponseStream
  └── self.return_processed_chunk_logic(...)  # 后处理
        │
        ▼
返回 ModelResponseStream (OpenAI 兼容格式)
```

### 8.2 非流式路径对比

```
litellm.completion(model="claude-3-opus", stream=False, ...)
  │
  ▼
main.py:completion()
  │
  ▼
anthropic/chat/handler.py:AnthropicChatCompletion.completion()
  ├── stream=False
  ├── HTTP POST (非流式)
  └── 调用 config.transform_response()
        │
        ├── 解析完整 JSON 响应
        ├── 一次性转换为 ModelResponse
        └── 调用 logging.success_handler() (同步)
              │
              ▼
        返回 ModelResponse
```

---

## 9. 扩展点与自定义

### 9.1 自定义 Provider 流式实现

要添加新的 provider 支持，需要实现：

1. **`ModelResponseIterator`**：将原生流式格式转换为 `GenericStreamingChunk`
2. **或直接返回 `ModelResponseStream`**：如果是 OpenAI 兼容格式

```python
# 参考 litellm/llms/anthropic/chat/handler.py:522
class ModelResponseIterator:
    def __init__(self, streaming_response, sync_stream: bool, ...):
        self.streaming_response = streaming_response
        self.response_id = _generate_id()  # 统一生成响应 ID
    
    def __iter__(self) -> Iterator[GenericStreamingChunk]:
        for line in self.streaming_response:
            # 解析 provider 特有格式
            # 转换为 GenericStreamingChunk
            yield {
                "text": extracted_text,
                "tool_use": extracted_tool_call,
                "is_finished": is_last_chunk,
                "finish_reason": finish_reason,
                "usage": usage_dict,
            }
```

### 9.2 自定义流式回调

```python
from litellm.integrations.custom_logger import CustomLogger

class MyStreamingCallback(CustomLogger):
    async def async_post_call_streaming_deployment_hook(
        self, 
        request_data: dict, 
        response_chunk, 
        call_type
    ):
        # 在每个 chunk 返回前修改/监控
        print(f"Streaming chunk: {response_chunk}")
        return response_chunk  # 可返回修改后的 chunk
    
    async def async_post_call_streaming_hook(
        self,
        data: dict,
        response,
        user_api_key_dict,
    ):
        # Proxy 层的流式钩子
        pass
```

---

## 10. 注意事项与常见陷阱

### 10.1 关键陷阱

1. **`GenericStreamingChunk` vs `ModelResponseStream` 混淆**
   - `GenericStreamingChunk` 是**中间格式**（TypedDict）
   - `ModelResponseStream` 是**最终输出格式**（Pydantic BaseModel）

2. **流式响应的 `usage` 字段**
   - 默认情况下流式响应**不返回** `usage`
   - 需要设置 `stream_options={"include_usage": True}` 才会在最后一个 chunk 返回

3. **异步迭代器的事件循环**
   - `CustomStreamWrapper.set_logging_event_loop()` 用于同步流式时设置 async 日志的事件循环
   - 见 `main.py:634-637`

4. **HTTP 客户端关闭**
   - 不要在缓存驱逐时关闭 HTTP 客户端（见 `AGENTS.md` 中的规则）
   - 使用 `CustomStreamWrapper.aclose()` 显式关闭

### 10.2 调试技巧

```python
# 启用详细日志
litellm.set_verbose = True

# 查看 chunk 处理过程
# 在 streaming_handler.py 的 chunk_creator 中添加断点
# 或查看 CustomStreamWrapper.chunks 列表（已处理的 chunk 缓存）
```

---

## 附录：类型定义参考

### A.1 `GenericStreamingChunk`

```python
# litellm/types/utils.py:274
class GenericStreamingChunk(TypedDict, total=False):
    text: Required[str]
    tool_use: Optional[ChatCompletionToolCallChunk]
    is_finished: Required[bool]
    finish_reason: Required[str]
    usage: Required[Optional[ChatCompletionUsageBlock]]
    logprobs: Optional[BaseModel]
    original_chunk: Optional[BaseModel]
    index: Required[Optional[int]]
    provider_specific_fields: Optional[Dict[str, Any]]
```

### A.2 关键回调接口

```python
# CustomLogger 中的流式相关方法
class CustomLogger:
    # 每个 chunk 的钩子 (Core 层)
    async def async_post_call_streaming_deployment_hook(
        self, request_data, response_chunk, call_type
    ): ...
    
    # 每个 chunk 的钩子 (Proxy 层)
    async def async_post_call_streaming_hook(
        self, data, response, user_api_key_dict
    ): ...
    
    # 迭代器级别钩子 (Proxy 层，用于 Guardrails)
    async def async_post_call_streaming_iterator_hook(
        self, user_api_key_dict, response, request_data
    ): ...
```

---

*本报告基于 LiteLLM 代码库分析生成，分析范围包括 `litellm/main.py`、`litellm/litellm_core_utils/streaming_handler.py`、`litellm/llms/anthropic/chat/handler.py`、`litellm/responses/streaming_iterator.py` 等核心文件。*
