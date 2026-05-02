# LiteLLM 跨 Provider 流式归一化深度分析 (Round 2)

本报告深入分析 Bedrock (Converse/Invoke 两种模式)、Vertex AI/Gemini、Cohere 等主流 provider 的流式响应实现细节，对比它们在格式抹平、数据块传递和回调分叉上的差异。

---

## 目录

1. [Bedrock 双模式深度对比](#1-bedrock-双模式深度对比)
   - 1.1 Converse API 模式
   - 1.2 Invoke API 模式
   - 1.3 模式路由逻辑
2. [Vertex AI/Gemini 流式实现](#2-vertex-aigemini-流式实现)
   - 2.1 直接 ModelResponseStream 路径
   - 2.2 Tool Calls 跨 chunk 处理
   - 2.3 JSON 累积模式
3. [Cohere V1/V2 格式差异](#3-cohere-v1v2-格式差异)
   - 3.1 V1 简单格式
   - 3.2 V2 事件驱动格式
4. [各 Provider 归一化路径对比表](#4-各-provider-归一化路径对比表)
5. [数据块传递机制详解](#5-数据块传递机制详解)
   - 5.1 中间格式选择策略
   - 5.2 两种传递路径 (GenericStreamingChunk vs 直接 ModelResponseStream)
6. [Streaming 回调与非流式分叉点](#6-streaming-回调与非流式分叉点)
   - 6.1 Bedrock 分叉点
   - 6.2 Vertex AI 分叉点
   - 6.3 Cohere 分叉点
7. [关键代码位置速查](#7-关键代码位置速查)

---

## 1. Bedrock 双模式深度对比

Bedrock 提供两种流式 API 模式，它们的实现路径完全不同。

### 1.1 Converse API 模式

**适用模型**: 注册在 `litellm.bedrock_converse_models` 中的模型，或显式指定 `converse/` 前缀

**端点**: 
```
/model/{modelId}/converse-stream
```

**核心实现文件**:
- `litellm/llms/bedrock/chat/converse_handler.py`
- `litellm/llms/bedrock/common_utils.py` (AWSEventStreamDecoder)

#### 数据流转路径

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Bedrock Converse 流式路径                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. HTTP 请求发起                                                            │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ converse_handler.py:358-363                                      │   │
│     │ if (stream is not None and stream is True):                      │   │
│     │     endpoint_url = f"{endpoint_url}/model/{modelId}/converse-stream"│
│     │ else:                                                              │   │
│     │     endpoint_url = f"{endpoint_url}/model/{modelId}/converse"   │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  2. 同步/异步调用分发                                                        │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ converse_handler.py:374-398 (acompletion)                       │   │
│     │ if stream is True:                                                │   │
│     │     return self.async_streaming(...)                              │   │
│     │ else:                                                              │   │
│     │     return self.async_completion(...)                             │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  3. async_streaming() 处理                                                  │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ converse_handler.py:92-161                                       │   │
│     │ - 构造请求: AmazonConverseConfig._async_transform_request()      │   │
│     │ - 签名请求: get_request_headers()                                 │   │
│     │ - 发起调用: make_call()                                           │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  4. make_call() - 响应处理                                                  │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ converse_handler.py:185-276 / invoke_handler.py:185-279        │   │
│     │                                                                     │   │
│     │ 分支逻辑:                                                           │   │
│     │ ┌─────────────────────────────────────────────────────────────┐  │   │
│     │ │ if fake_stream:                                              │  │   │
│     │ │     → MockResponseIterator (非流式转流式)                    │  │   │
│     │ │ elif bedrock_invoke_provider == "anthropic":                │  │   │
│     │ │     → AmazonAnthropicClaudeStreamDecoder (特殊处理)          │  │   │
│     │ │ elif bedrock_invoke_provider == "deepseek_r1":               │  │   │
│     │ │     → AmazonDeepSeekR1StreamDecoder (特殊处理)               │  │   │
│     │ │ else:                                                          │  │   │
│     │ │     → AWSEventStreamDecoder (通用)                            │  │   │
│     │ └─────────────────────────────────────────────────────────────┘  │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  5. AWSEventStreamDecoder - EventStream 解析                               │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ sagemaker/common_utils.py:25-120 (被 Bedrock 复用)              │   │
│     │                                                                     │   │
│     │ 初始化:                                                             │   │
│     │ - self.parser = EventStreamJSONParser() (botocore)              │   │
│     │ - self.is_messages_api = is_messages_api                          │   │
│     │                                                                     │   │
│     │ iter_bytes() 流程:                                                  │   │
│     │ ① 创建 EventStreamBuffer                                           │   │
│     │ ② 循环: for chunk in iterator:                                     │   │
│     │    - event_stream_buffer.add_data(chunk)                          │   │
│     │    - for event in event_stream_buffer:                            │   │
│     │       → message = self._parse_message_from_event(event)           │   │
│     │       → 去除 SSE 格式: _strip_sse_data_from_chunk()              │   │
│     │       → 累积 JSON (处理分片)                                       │   │
│     │       → 解析成功后:                                                 │   │
│     │         if self.is_messages_api:                                    │   │
│     │             yield self._chunk_parser_messages_api()  → OpenAI 格式│   │
│     │         else:                                                       │   │
│     │             yield self._chunk_parser()  → GenericStreamingChunk   │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  6. 包装为 CustomStreamWrapper                                              │
│     ┌─────────────────────────────────────────────────────────────────┐   │
│     │ converse_handler.py:155-160                                       │   │
│     │ streaming_response = CustomStreamWrapper(                         │   │
│     │     completion_stream=completion_stream,  # AWSEventStreamDecoder│   │
│     │     model=model,                                                   │   │
│     │     custom_llm_provider="bedrock",                                │   │
│     │     logging_obj=logging_obj,                                       │   │
│     │ )                                                                  │   │
│     └─────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### AWSEventStreamDecoder 核心方法

**`_chunk_parser()`** (`sagemaker/common_utils.py:43-67`):
```python
def _chunk_parser(self, chunk_data: dict) -> GChunk:
    _token = chunk_data.get("token", {}) or {}
    _index = chunk_data.get("index", None) or 0
    is_finished = False
    finish_reason = ""
    
    _text = _token.get("text", "")
    if _text == "<|endoftext|>":
        # 特殊结束标记
        return GChunk(
            text="",
            index=_index,
            is_finished=True,
            finish_reason="stop",
            usage=None,
        )
    
    return GChunk(
        text=_text,
        index=_index,
        is_finished=is_finished,
        finish_reason=finish_reason,
        usage=None,
    )
```

**`_chunk_parser_messages_api()`** (`sagemaker/common_utils.py:34-41`):
```python
def _chunk_parser_messages_api(
    self, chunk_data: dict
) -> StreamingChatCompletionChunk:
    # 直接返回 OpenAI 格式，跳过 GenericStreamingChunk
    openai_chunk = StreamingChatCompletionChunk(
        **{"model": self.model, **chunk_data}
    )
    return openai_chunk
```

---

### 1.2 Invoke API 模式

**适用模型**: 未注册在 `bedrock_converse_models` 中的模型，或显式指定 `invoke/` 前缀

**端点**:
```
/model/{modelId}/invoke-with-response-stream
```

**核心实现文件**:
- `litellm/llms/bedrock/chat/invoke_handler.py`

#### Invoke 模式的特殊设计

Invoke 模式根据 **底层 provider 类型** 使用不同的解码器，这是 Bedrock 最复杂的部分。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Bedrock Invoke 解码器路由逻辑                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  invoke_handler.py:240-261 (async) / 336-357 (sync)                      │
│                                                                             │
│     if bedrock_invoke_provider == "anthropic":                            │
│         ┌─────────────────────────────────────────────────────────────┐   │
│         │ AmazonAnthropicClaudeStreamDecoder                          │   │
│         │ - 继承 AWSEventStreamDecoder                                 │   │
│         │ - 内部使用 AnthropicModelResponseIterator.chunk_parser()     │   │
│         │ - 直接返回 ModelResponseStream (跳过 GenericStreamingChunk)  │   │
│         └─────────────────────────────────────────────────────────────┘   │
│                                                                             │
│     elif bedrock_invoke_provider == "deepseek_r1":                       │
│         ┌─────────────────────────────────────────────────────────────┐   │
│         │ AmazonDeepSeekR1StreamDecoder                               │   │
│         │ - 继承 AWSEventStreamDecoder                                 │   │
│         │ - 内部使用 AmazonDeepseekR1ResponseIterator.chunk_parser()   │   │
│         └─────────────────────────────────────────────────────────────┘   │
│                                                                             │
│     else:                                                                    │
│         ┌─────────────────────────────────────────────────────────────┐   │
│         │ AWSEventStreamDecoder (基础)                                 │   │
│         │ - 返回 GenericStreamingChunk                                 │   │
│         └─────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### AmazonAnthropicClaudeStreamDecoder 详解

**位置**: `invoke_handler.py:1875-1895`

```python
class AmazonAnthropicClaudeStreamDecoder(AWSEventStreamDecoder):
    """
    处理 Anthropic 模型流式响应的专用解码器
    
    与基础 AWSEventStreamDecoder 的唯一区别是 _chunk_parser 方法
    """
    
    def __init__(
        self,
        model: str,
        sync_stream: bool,
        json_mode: Optional[bool] = None,
    ) -> None:
        super().__init__(model=model)
        # 复用 Anthropic 的 ModelResponseIterator
        self.anthropic_model_response_iterator = AnthropicModelResponseIterator(
            streaming_response=None,
            sync_stream=sync_stream,
            json_mode=json_mode,
        )
    
    def _chunk_parser(self, chunk_data: dict) -> ModelResponseStream:
        # 直接委托给 Anthropic 的 chunk_parser
        # 返回 ModelResponseStream，而非 GenericStreamingChunk
        return self.anthropic_model_response_iterator.chunk_parser(chunk=chunk_data)
```

**关键点**:
1. **复用 Anthropic 迭代器**: Bedrock 上的 Anthropic 模型与直接调用 Anthropic API 使用**相同的 chunk 解析逻辑**
2. **跳过中间格式**: 直接返回 `ModelResponseStream`，不经过 `GenericStreamingChunk`
3. **EventStream 包装**: 外层仍然通过 `AWSEventStreamDecoder.iter_bytes()` 解析 AWS EventStream 格式

---

### 1.3 模式路由逻辑

Bedrock 如何决定使用 Converse 还是 Invoke 模式？

**位置**: `bedrock/common_utils.py:688-746`

```python
@staticmethod
def get_bedrock_route(
    model: str,
) -> Literal["converse", "invoke", "converse_like", ...]:
    """
    获取 Bedrock 路由类型
    """
    route_mappings: Dict[str, Literal[...]] = {
        "invoke/": "invoke",
        "converse_like/": "converse_like",
        "converse/": "converse",
        "agent/": "agent",
        "agentcore/": "agentcore",
        "async_invoke/": "async_invoke",
        "openai/": "openai",
        "mantle/": "mantle",
    }
    
    # 1. 显式路由前缀优先
    for prefix, route_type in route_mappings.items():
        if prefix in model:
            return route_type
    
    # 2. Nova 模型特殊处理
    _model_after_bedrock = model.replace("bedrock/", "", 1)
    if _model_after_bedrock.startswith("nova-2/") or \
       _model_after_bedrock.startswith("nova/"):
        return "converse"
    
    # 3. 检查模型是否在 converse_models 列表中
    base_model = BedrockModelInfo.get_base_model(model)
    alt_model = BedrockModelInfo.get_non_litellm_routing_model_name(model=model)
    if (
        base_model in litellm.bedrock_converse_models
        or alt_model in litellm.bedrock_converse_models
    ):
        return "converse"
    
    # 4. 默认使用 invoke
    return "invoke"
```

**路由决策优先级**:
1. **显式前缀** (最高优先级): `invoke/`, `converse/` 等
2. **Nova 模型**: `nova-2/`, `nova/` → Converse
3. **Converse 模型列表**: 注册在 `litellm.bedrock_converse_models` 中的模型
4. **默认**: Invoke

---

## 2. Vertex AI/Gemini 流式实现

Vertex AI/Gemini 有一个独特的流式实现：**直接返回 ModelResponseStream，完全跳过 GenericStreamingChunk 中间格式**。

**核心实现文件**:
- `litellm/llms/vertex_ai/gemini/vertex_and_google_ai_studio_gemini.py`

---

### 2.1 直接 ModelResponseStream 路径

#### ModelResponseIterator 类

**位置**: `vertex_and_google_ai_studio_gemini.py:3127-3403`

```python
class ModelResponseIterator:
    def __init__(
        self,
        streaming_response,
        sync_stream: bool,
        logging_obj: LoggingClass,
        response_headers: Optional[Dict[str, str]] = None,
    ):
        self.streaming_response = streaming_response
        self.chunk_type: Literal["valid_json", "accumulated_json"] = "valid_json"
        self.accumulated_json = ""
        self.sent_first_chunk = False
        self.logging_obj = logging_obj
        self.response_headers = response_headers or {}
        self.is_function_call = check_is_function_call(logging_obj)
        self.cumulative_tool_call_index: int = 0
        self.has_seen_tool_calls: bool = False  # 关键状态标记
```

#### chunk_parser() 方法

**位置**: `vertex_and_google_ai_studio_gemini.py:3263-3304`

```python
def chunk_parser(self, chunk: dict) -> Optional["ModelResponseStream"]:
    try:
        verbose_logger.debug(f"RAW GEMINI CHUNK: {chunk}")
        from litellm.types.utils import ModelResponseStream
        
        # 1. 类型转换为 Gemini 格式
        processed_chunk = GenerateContentResponseBody(**chunk)
        response_id = processed_chunk.get("responseId")
        
        # 2. 创建 ModelResponseStream (直接创建，不经过 GenericStreamingChunk)
        model_response = ModelResponseStream(choices=[], id=response_id)
        
        # 3. 检查内容过滤
        blocked_response = VertexGeminiConfig._check_prompt_level_content_filter(
            processed_chunk=processed_chunk,
            response_id=response_id,
        )
        if blocked_response is not None:
            model_response = blocked_response
        
        # 4. 处理 candidates
        _candidates: Optional[List[Candidates]] = processed_chunk.get("candidates")
        if _candidates:
            (
                grounding_metadata,
                url_context_metadata,
                safety_ratings,
                citation_metadata,
            ) = self._apply_stream_candidates(_candidates, model_response)
        
        # 5. 处理 usage
        usage = self._apply_stream_usage_metadata(
            processed_chunk, model_response, grounding_metadata
        )
        setattr(model_response, "usage", usage)
        
        # 6. 标记未完成
        model_response._hidden_params["is_finished"] = False
        return model_response
        
    except json.JSONDecodeError:
        raise ValueError(f"Failed to decode JSON from chunk: {chunk}")
```

**关键特点**:
1. **无中间格式**: 直接创建 `ModelResponseStream`，不经过 `GenericStreamingChunk`
2. **丰富的元数据**: 处理 grounding、url_context、safety_ratings、citation 等 Gemini 特有元数据
3. **usage 在每个 chunk**: Gemini 在每个 chunk 中都可能返回 `usageMetadata`

---

### 2.2 Tool Calls 跨 chunk 处理

Gemini 有一个独特的行为：**tool_calls 和 finishReason 放在不同的 chunk 中**。

#### 问题场景

```
Chunk 1: {"candidates": [{"content": {"parts": [{"functionCall": {...}]}}]}
         → 包含 tool_calls，但没有 finishReason

Chunk 2: {"candidates": [{"finishReason": "STOP"}]}
         → 包含 finishReason，但没有 content/tool_calls
```

如果不特殊处理，OpenAI 客户端会看到：
- Chunk 1: `delta.tool_calls = [...]` 
- Chunk 2: `finish_reason = "stop"` (应该是 `"tool_calls"`)

#### 解决方案: has_seen_tool_calls 标记

**位置**: `vertex_and_google_ai_studio_gemini.py:3149-3225`

```python
def _apply_stream_candidates(
    self,
    _candidates: List[Candidates],
    model_response: Any,
) -> Tuple[List[dict], List[dict], List[dict], List[dict]]:
    
    # 处理 candidates...
    (
        grounding_metadata,
        url_context_metadata,
        safety_ratings,
        citation_metadata,
        self.cumulative_tool_call_index,
    ) = VertexGeminiConfig._process_candidates(
        _candidates,
        model_response,
        self.logging_obj.optional_params,
        cumulative_tool_call_index=self.cumulative_tool_call_index,
    )
    
    # 关键: 追踪是否看到过 tool_calls
    if not self.has_seen_tool_calls:
        for choice in model_response.choices:
            if (
                hasattr(choice, "delta")
                and choice.delta
                and choice.delta.tool_calls
            ):
                self.has_seen_tool_calls = True
                break
    
    # 处理空 content 但有 finishReason 的情况
    # _process_candidates 会跳过没有 content 的 candidates
    # 导致最终 chunk 的 finish_reason 丢失
    if not model_response.choices and _candidates:
        from litellm.types.utils import Delta, StreamingChoices
        
        for candidate in _candidates:
            finish_reason_str = candidate.get("finishReason")
            if finish_reason_str is not None:
                # 关键修正: 如果之前看到过 tool_calls，使用 "tool_calls"
                if self.has_seen_tool_calls:
                    mapped_finish_reason = "tool_calls"
                else:
                    mapped_finish_reason = VertexGeminiConfig._check_finish_reason(
                        None, finish_reason_str
                    )
                choice = StreamingChoices(
                    finish_reason=mapped_finish_reason,
                    index=candidate.get("index", 0),
                    delta=Delta(content=None, role=None),
                    logprobs=None,
                    enhancements=None,
                )
                model_response.choices.append(choice)
    
    # 另一个边界情况: 最终 chunk 有空 content (text:"") 但有 finishReason
    # 这种情况下 _process_candidates 会创建 choice，但因为当前 chunk 没有
    # tool_calls，会把 finishReason="STOP" 映射为 "stop"
    if self.has_seen_tool_calls:
        for choice in model_response.choices:
            if choice.finish_reason == "stop":
                choice.finish_reason = "tool_calls"
    
    # ... 设置元数据
    return (
        grounding_metadata,
        url_context_metadata,
        safety_ratings,
        citation_metadata,
    )
```

**状态机逻辑**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Gemini Tool Calls 跨 chunk 追踪                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  初始状态:                                                                    │
│  self.has_seen_tool_calls = False                                           │
│                                                                             │
│  Chunk 1 处理:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ candidate = {"content": {"parts": [{"functionCall": {...}}]}}      │   │
│  │                                                                       │   │
│  │ _process_candidates() 创建:                                          │   │
│  │ choice.delta.tool_calls = [...]                                      │   │
│  │                                                                       │   │
│  │ 检测到 tool_calls:                                                    │   │
│  │ if choice.delta and choice.delta.tool_calls:                        │   │
│  │     self.has_seen_tool_calls = True  ◄──────── 状态改变             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                    │                                        │
│                                    ▼                                        │
│  Chunk 2 处理 (最终 chunk):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ candidate = {"finishReason": "STOP"}  ◄── 没有 content              │   │
│  │                                                                       │   │
│  │ 进入空 content 分支:                                                  │   │
│  │ if not model_response.choices and _candidates:                       │   │
│  │                                                                       │   │
│  │     if self.has_seen_tool_calls:  ◄──────── 检查状态                │   │
│  │         mapped_finish_reason = "tool_calls"  ◄── 正确映射          │   │
│  │     else:                                                             │   │
│  │         mapped_finish_reason = "stop"                                │   │
│  │                                                                       │   │
│  │     choice = StreamingChoices(finish_reason=mapped_finish_reason)   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  最终结果: finish_reason = "tool_calls" (符合 OpenAI 规范)                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 2.3 JSON 累积模式

Gemini 流式响应可能因为网络分片导致 JSON 不完整。Vertex AI 实现了**双模式解析**来处理这种情况。

#### 核心逻辑

**位置**: `vertex_and_google_ai_studio_gemini.py:3346-3403`

```python
def _common_chunk_parsing_logic(
    self, chunk: str
) -> Optional["ModelResponseStream"]:
    try:
        chunk = litellm.CustomStreamWrapper._strip_sse_data_from_chunk(chunk) or ""
        if len(chunk) > 0:
            """
            检查初始 chunk 是否是有效 JSON
            - 如果是部分 JSON → 进入累积模式
            - 如果有效 → 继续
            """
            if self.chunk_type == "valid_json":
                return self.handle_valid_json_chunk(chunk=chunk)
            elif self.chunk_type == "accumulated_json":
                return self.handle_accumulated_json_chunk(chunk=chunk)
        
        return None
    except Exception:
        raise
```

#### handle_valid_json_chunk()

**位置**: `vertex_and_google_ai_studio_gemini.py:3311-3326`

```python
def handle_valid_json_chunk(self, chunk: str) -> Optional["ModelResponseStream"]:
    chunk = chunk.strip()
    try:
        json_chunk = json.loads(chunk)
    except json.JSONDecodeError:
        # 解析失败，切换到累积模式
        # 这可能在任何时候发生，不仅仅是第一个 chunk
        # 见: https://github.com/BerriAI/litellm/issues/16562
        self.chunk_type = "accumulated_json"
        return self.handle_accumulated_json_chunk(chunk=chunk)
    
    if self.sent_first_chunk is False:
        self.sent_first_chunk = True
    
    return self.chunk_parser(chunk=json_chunk)
```

#### handle_accumulated_json_chunk()

**位置**: `vertex_and_google_ai_studio_gemini.py:3328-3344`

```python
def handle_accumulated_json_chunk(
    self, chunk: str
) -> Optional["ModelResponseStream"]:
    chunk = litellm.CustomStreamWrapper._strip_sse_data_from_chunk(chunk) or ""
    message = chunk.replace("\n\n", "")
    
    # 累积 JSON 数据
    self.accumulated_json += message
    
    # 尝试解析累积的 JSON
    try:
        _data = json.loads(self.accumulated_json)
        self.accumulated_json = ""  # 解析成功后重置
        return self.chunk_parser(chunk=_data)
    except json.JSONDecodeError:
        # 还不是有效 JSON，继续等待下一个 chunk
        return None
```

**状态转换图**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Gemini JSON 解析状态机                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        ┌──────────────────┐                                │
│                        │    初始状态       │                                │
│                        │ chunk_type =     │                                │
│                        │ "valid_json"     │                                │
│                        └────────┬─────────┘                                │
│                                 │                                          │
│                                 ▼                                          │
│                    ┌──────────────────────┐                                │
│                    │  handle_valid_json   │                                │
│                    │  _chunk()            │                                │
│                    │                      │                                │
│                    │  尝试 json.loads()   │                                │
│                    └──────────┬───────────┘                                │
│                               │                                             │
│              ┌────────────────┴────────────────┐                           │
│              │                                 │                           │
│              ▼                                 ▼                           │
│    ┌─────────────────┐             ┌─────────────────────┐                 │
│    │   解析成功       │             │    解析失败         │                 │
│    │                 │             │                     │                 │
│    │ 返回            │             │ self.chunk_type =   │                 │
│    │ chunk_parser()  │             │ "accumulated_json" │                 │
│    │                 │             │                     │                 │
│    └─────────────────┘             │ 切换到累积模式      │                 │
│                                    │                     │                 │
│                                    └──────────┬──────────┘                 │
│                                               │                            │
│                                               ▼                            │
│                                    ┌─────────────────────┐                 │
│                                    │  handle_accumulated  │                 │
│                                    │  _json_chunk()       │                 │
│                                    │                     │                 │
│                                    │ 累积:                │                 │
│                                    │ self.accumulated_json│                 │
│                                    │ += message           │                 │
│                                    │                     │                 │
│                                    │ 尝试解析累积数据      │                 │
│                                    └──────────┬──────────┘                 │
│                                               │                            │
│                              ┌────────────────┴────────────────┐           │
│                              │                                 │           │
│                              ▼                                 ▼           │
│                    ┌─────────────────┐             ┌─────────────────────┐ │
│                    │   解析成功       │             │    解析失败         │ │
│                    │                 │             │                     │ │
│                    │ 重置累积:        │             │ 返回 None           │ │
│                    │ accumulated_json│             │ 继续等待下一个 chunk │ │
│                    │ = ""             │             │                     │ │
│                    │                 │             │                     │ │
│                    │ 返回            │             └─────────────────────┘ │
│                    │ chunk_parser()  │                                     │
│                    └─────────────────┘                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Cohere V1/V2 格式差异

Cohere 有两个版本的流式 API，它们的事件格式完全不同。

**核心实现文件**:
- `litellm/llms/cohere/common_utils.py`

---

### 3.1 V1 简单格式

#### ModelResponseIterator (V1)

**位置**: `cohere/common_utils.py:120-218`

```python
class ModelResponseIterator:
    def __init__(
        self, streaming_response, sync_stream: bool, 
        json_mode: Optional[bool] = False
    ):
        self.streaming_response = streaming_response
        self.response_iterator = self.streaming_response
        self.content_blocks: List = []
        self.tool_index = -1
        self.json_mode = json_mode
```

#### V1 chunk_parser()

**位置**: `cohere/common_utils.py:130-163`

```python
def chunk_parser(self, chunk: dict) -> GenericStreamingChunk:
    try:
        text = ""
        tool_use: Optional[ChatCompletionToolCallChunk] = None
        is_finished = False
        finish_reason = ""
        usage: Optional[ChatCompletionUsageBlock] = None
        provider_specific_fields = None
        
        index = int(chunk.get("index", 0))
        
        # V1 格式: 简单的字段提取
        if "text" in chunk:
            text = chunk["text"]
        elif "is_finished" in chunk and chunk["is_finished"] is True:
            is_finished = chunk["is_finished"]
            finish_reason = chunk["finish_reason"]
        
        # Citations (provider_specific)
        if "citations" in chunk:
            provider_specific_fields = {"citations": chunk["citations"]}
        
        return GenericStreamingChunk(
            text=text,
            tool_use=tool_use,
            is_finished=is_finished,
            finish_reason=finish_reason,
            usage=usage,
            index=index,
            provider_specific_fields=provider_specific_fields,
        )
        
    except json.JSONDecodeError:
        raise ValueError(f"Failed to decode JSON from chunk: {chunk}")
```

**V1 格式特点**:
1. **扁平结构**: 直接在 chunk 根级别有 `text`, `is_finished`, `finish_reason`, `citations`
2. **无事件类型**: 没有 `type` 或 `event` 字段
3. **返回 GenericStreamingChunk**: 经过中间格式

---

### 3.2 V2 事件驱动格式

#### CohereV2ModelResponseIterator

**位置**: `cohere/common_utils.py:221-417`

V2 格式基于**事件类型**路由，更加灵活但也更复杂。

```python
class CohereV2ModelResponseIterator:
    """V2-specific response iterator for Cohere streaming"""
    
    def chunk_parser(self, chunk: dict) -> GenericStreamingChunk:
        """
        解析 Cohere v2 流式 chunks
        
        v2 格式:
        - 内容: chunk.type == "content-delta" → chunk.delta.message.content.text
        - Tool calls: chunk.type == "tool-call-delta" → chunk.delta.tool_calls
        - Tool plan: chunk.event == "tool-plan-delta" → chunk.data.delta.message.tool_plan
        - Citations: chunk.event == "citation-start" → chunk.data.delta.message.citations
        - 结束: chunk.event == "message-end" → chunk.data.delta.finish_reason
        """
        try:
            text = ""
            tool_use: Optional[ChatCompletionToolCallChunk] = None
            is_finished = False
            finish_reason = ""
            usage: Optional[ChatCompletionUsageBlock] = None
            provider_specific_fields = None
            
            index = int(chunk.get("index", 0))
            chunk_type = chunk.get("type", "")      # 用于 content/tool-call
            event_type = chunk.get("event", "")      # 用于 plan/citation/end
            
            # 根据类型/事件路由到不同的解析方法
            if chunk_type == "content-delta":
                text = self._parse_content_delta(chunk)
            elif chunk_type == "tool-call-delta":
                tool_use = self._parse_tool_call_delta(chunk)
            elif event_type == "tool-plan-delta":
                provider_specific_fields = self._parse_tool_plan_delta(chunk)
            elif event_type == "citation-start":
                provider_specific_fields = self._parse_citation_start(chunk)
            elif event_type == "message-end":
                is_finished, finish_reason, usage = self._parse_message_end(chunk)
            
            # 兼容: 任何 chunk 中的 citations
            if "citations" in chunk:
                if provider_specific_fields is None:
                    provider_specific_fields = {}
                provider_specific_fields["citations"] = chunk["citations"]
            
            return GenericStreamingChunk(
                text=text,
                tool_use=tool_use,
                is_finished=is_finished,
                finish_reason=finish_reason,
                usage=usage,
                index=index,
                provider_specific_fields=provider_specific_fields,
            )
            
        except Exception as e:
            raise ValueError(f"Failed to parse v2 chunk: {e}, chunk: {chunk}")
```

#### V2 各事件类型解析方法

**`_parse_content_delta()`** (`cohere/common_utils.py:233-242`):
```python
def _parse_content_delta(self, chunk: dict) -> str:
    """解析 content-delta chunks 提取文本"""
    delta = chunk.get("delta", {})
    message = delta.get("message", {})
    content = message.get("content", {})
    if isinstance(content, dict) and "text" in content:
        return content["text"]
    elif isinstance(content, str):
        return content
    return ""
```

**`_parse_tool_call_delta()`** (`cohere/common_utils.py:244-259`):
```python
def _parse_tool_call_delta(
    self, chunk: dict
) -> Optional[ChatCompletionToolCallChunk]:
    """解析 tool-call-delta chunks 提取 tool calls"""
    delta = chunk.get("delta", {})
    tool_calls = delta.get("tool_calls", [])
    if tool_calls:
        return {
            "id": tool_calls[0].get("id", ""),
            "type": "function",
            "function": {
                "name": tool_calls[0].get("name", ""),
                "arguments": tool_calls[0].get("arguments", ""),
            },
        }  # type: ignore
    return None
```

**`_parse_message_end()`** (`cohere/common_utils.py:288-308`):
```python
def _parse_message_end(
    self, chunk: dict
) -> Tuple[bool, str, Optional[ChatCompletionUsageBlock]]:
    """解析 message-end 事件提取结束信息和 usage"""
    data = chunk.get("data", {})
    delta = data.get("delta", {})
    is_finished = True
    finish_reason = delta.get("finish_reason", "stop")
    
    usage = None
    usage_data = delta.get("usage", {})
    if usage_data:
        tokens_data = usage_data.get("tokens", {})
        usage = ChatCompletionUsageBlock(
            prompt_tokens=tokens_data.get("input_tokens", 0),
            completion_tokens=tokens_data.get("output_tokens", 0),
            total_tokens=tokens_data.get("input_tokens", 0)
            + tokens_data.get("output_tokens", 0),
        )
    
    return is_finished, finish_reason, usage
```

#### V2 事件类型完整列表

| 事件类型 | chunk 字段 | 解析方法 | 输出字段 |
|---------|-----------|---------|---------|
| **文本内容** | `type: "content-delta"` | `_parse_content_delta()` | `text` |
| **Tool Call** | `type: "tool-call-delta"` | `_parse_tool_call_delta()` | `tool_use` |
| **Tool Plan** | `event: "tool-plan-delta"` | `_parse_tool_plan_delta()` | `provider_specific_fields.tool_plan` |
| **Citation** | `event: "citation-start"` | `_parse_citation_start()` | `provider_specific_fields.citations` |
| **结束** | `event: "message-end"` | `_parse_message_end()` | `is_finished`, `finish_reason`, `usage` |

---

## 4. 各 Provider 归一化路径对比表

### 4.1 核心数据结构对比

| Provider | 原生流式格式 | 中间格式 | 最终格式 | 归一化位置 |
|---------|------------|---------|---------|-----------|
| **OpenAI/Azure** | SSE + `ChatCompletionChunk` | 无 | `ModelResponseStream` | `handle_openai_chat_completion_chunk()` |
| **Anthropic** | 事件流 (message_start, content_block_delta 等) | `GenericStreamingChunk` | `ModelResponseStream` | `ModelResponseIterator.chunk_parser()` |
| **Bedrock Converse** | AWS EventStream | `GenericStreamingChunk` \| `StreamingChatCompletionChunk` | `ModelResponseStream` | `AWSEventStreamDecoder` |
| **Bedrock Invoke (Anthropic)** | AWS EventStream + Anthropic 事件 | 无 (直接 ModelResponseStream) | `ModelResponseStream` | `AmazonAnthropicClaudeStreamDecoder` |
| **Bedrock Invoke (其他)** | AWS EventStream | `GenericStreamingChunk` | `ModelResponseStream` | `AWSEventStreamDecoder` |
| **Vertex AI/Gemini** | JSON (可能分片) | 无 (直接 ModelResponseStream) | `ModelResponseStream` | `ModelResponseIterator.chunk_parser()` |
| **Cohere V1** | 扁平 JSON | `GenericStreamingChunk` | `ModelResponseStream` | `ModelResponseIterator.chunk_parser()` |
| **Cohere V2** | 事件驱动 JSON | `GenericStreamingChunk` | `ModelResponseStream` | `CohereV2ModelResponseIterator.chunk_parser()` |

### 4.2 两条归一化路径

LiteLLM 实现了**两条不同的归一化路径**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        路径 A: 经过 GenericStreamingChunk                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provider 原生格式                                                           │
│        │                                                                    │
│        ▼                                                                    │
│  ┌──────────────────┐                                                       │
│  │ Provider 特有    │                                                       │
│  │ ModelResponse    │                                                       │
│  │ Iterator         │                                                       │
│  │                  │                                                       │
│  │ 例如:             │                                                       │
│  │ - Anthropic      │                                                       │
│  │ - Cohere V1/V2   │                                                       │
│  │ - Bedrock (通用) │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    GenericStreamingChunk (中间格式)                   │  │
│  │                                                                       │  │
│  │ class GenericStreamingChunk(TypedDict, total=False):                │  │
│  │     text: Required[str]                                               │  │
│  │     tool_use: Optional[ChatCompletionToolCallChunk]                  │  │
│  │     is_finished: Required[bool]                                       │  │
│  │     finish_reason: Required[str]                                      │  │
│  │     usage: Required[Optional[ChatCompletionUsageBlock]]               │  │
│  │     index: int                                                         │  │
│  │     provider_specific_fields: Optional[dict]                          │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │          CustomStreamWrapper.chunk_creator()                         │  │
│  │                                                                       │  │
│  │ 检测是否是 GenericStreamingChunk:                                     │  │
│  │ if isinstance(chunk, dict) and                                        │  │
│  │    generic_chunk_has_all_required_fields(chunk=chunk):               │  │
│  │     → 从 GenericStreamingChunk 构建 ModelResponseStream               │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    ModelResponseStream (最终格式)                      │  │
│  │                                                                       │  │
│  │ (OpenAI 兼容的流式响应格式)                                           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────┐
│                   路径 B: 直接 ModelResponseStream (跳过中间格式)             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Provider 原生格式                                                           │
│        │                                                                    │
│        ▼                                                                    │
│  ┌──────────────────┐                                                       │
│  │ Provider 特有    │                                                       │
│  │ ModelResponse    │                                                       │
│  │ Iterator         │                                                       │
│  │                  │                                                       │
│  │ 例如:             │                                                       │
│  │ - Vertex AI      │                                                       │
│  │ - Bedrock Invoke │                                                       │
│  │   (Anthropic)    │                                                       │
│  └────────┬─────────┘                                                       │
│           │                                                                 │
│           │ chunk_parser() 返回                                             │
│           │ ModelResponseStream                                             │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │          CustomStreamWrapper.chunk_creator()                         │  │
│  │                                                                       │  │
│  │ 检测是否已经是 ModelResponseStream:                                   │  │
│  │ elif isinstance(chunk, ModelResponseStream):                         │  │
│  │     → 直接返回，跳过 GenericStreamingChunk 转换                       │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    ModelResponseStream (最终格式)                      │  │
│  │                                                                       │  │
│  │ (OpenAI 兼容的流式响应格式)                                           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.3 路径选择策略

为什么有些 provider 选择路径 A，有些选择路径 B？

**选择路径 B (直接 ModelResponseStream) 的情况**:

1. **复杂的元数据需求**:
   - Vertex AI/Gemini: 需要处理 grounding_metadata、url_context_metadata、safety_ratings、citation_metadata 等
   - 这些元数据无法完全放入 `GenericStreamingChunk.provider_specific_fields`

2. **复用现有解析逻辑**:
   - Bedrock Invoke (Anthropic): 复用 `AnthropicModelResponseIterator.chunk_parser()`
   - 这个方法本来就返回 `ModelResponseStream`

3. **Tool calls 跨 chunk 状态追踪**:
   - Vertex AI 需要 `has_seen_tool_calls` 状态
   - 这种状态追踪在 `GenericStreamingChunk` 层难以实现

**选择路径 A (经过 GenericStreamingChunk) 的情况**:

1. **简单格式**:
   - Cohere V1/V2、Bedrock 通用模式
   - 格式相对扁平，容易映射到 `GenericStreamingChunk`

2. **统一中间层抽象**:
   - 多个 provider 可以共享相同的后续处理逻辑
   - `return_processed_chunk_logic()` 可以基于统一的中间格式工作

---

## 5. 数据块传递机制详解

### 5.1 CustomStreamWrapper 核心流转

**位置**: `litellm/litellm_core_utils/streaming_handler.py`

#### 迭代器协议

```python
class CustomStreamWrapper:
    """
    核心流式包装器，实现 __next__ 和 __anext__
    所有 provider 的流式响应最终都被包装为这个类
    """
    
    def __next__(self) -> ModelResponseStream:
        while True:
            # 1. 从底层迭代器获取原始 chunk
            chunk = self.response_iterator.__next__()
            
            # 2. 通过 chunk_creator 归一化
            model_response_chunk = self.chunk_creator(chunk)
            if model_response_chunk is None:
                continue
            
            # 3. 后处理 (合并推理内容等)
            processed_chunk = self.return_processed_chunk_logic(
                model_response_chunk
            )
            
            # 4. 触发流式回调
            if self.callbacks:
                asyncio.run(
                    self._call_post_streaming_deployment_hook(
                        model_response=processed_chunk
                    )
                )
            
            # 5. 累积 usage
            if processed_chunk.usage is not None:
                self.accumulated_usage = processed_chunk.usage
            
            return processed_chunk
```

#### chunk_creator() - 路径分叉点

**位置**: `streaming_handler.py:1121` (约)

```python
def chunk_creator(self, chunk: Any):
    """
    核心归一化方法: 根据 chunk 类型选择不同的转换路径
    """
    
    # 路径 1: GenericStreamingChunk 格式
    if isinstance(chunk, dict) and generic_chunk_has_all_required_fields(chunk=chunk):
        # 从 GenericStreamingChunk 构建 ModelResponseStream
        completion_obj = ModelResponseStream()
        completion_obj.model = self.model
        
        # 处理 Anthropic 格式的 GenericStreamingChunk
        anthropic_response_obj = chunk
        if "text" in anthropic_response_obj:
            completion_obj["content"] = anthropic_response_obj["text"]
        
        if anthropic_response_obj.get("is_finished"):
            self.received_finish_reason = anthropic_response_obj["finish_reason"]
        
        # 处理 tool_use
        if anthropic_response_obj.get("tool_use"):
            # ... 构建 delta.tool_calls
        
        return completion_obj
    
    # 路径 2: 已经是 ModelResponseStream (Vertex AI、Bedrock Invoke Anthropic)
    elif isinstance(chunk, ModelResponseStream):
        # 直接返回，跳过转换
        return chunk
    
    # 路径 3: OpenAI/Azure 原生格式
    else:
        response_obj = self.handle_openai_chat_completion_chunk(chunk)
        return response_obj
```

#### generic_chunk_has_all_required_fields()

**位置**: `litellm/types/utils.py` (约)

```python
def generic_chunk_has_all_required_fields(chunk: dict) -> bool:
    """
    检测一个 dict 是否是有效的 GenericStreamingChunk
    
    GenericStreamingChunk 必需字段:
    - text: str
    - is_finished: bool
    - finish_reason: str
    - usage: Optional[ChatCompletionUsageBlock]
    
    注意: 使用 `get()` 检查，允许字段存在但为 None/""
    """
    required_fields = ["text", "is_finished", "finish_reason", "usage"]
    for field in required_fields:
        if field not in chunk:
            return False
    return True
```

---

### 5.2 Provider 特有迭代器注册

LiteLLM 如何知道哪个 provider 使用哪个迭代器？

#### 方式 1: 在 handler 中直接包装

**Bedrock 示例** (`converse_handler.py:155-160`):
```python
# 1. 获取 provider 特有迭代器
decoder = AWSEventStreamDecoder(model=model, json_mode=json_mode)
completion_stream = decoder.iter_bytes(
    response.iter_bytes(chunk_size=stream_chunk_size)
)

# 2. 包装为 CustomStreamWrapper
streaming_response = CustomStreamWrapper(
    completion_stream=completion_stream,
    model=model,
    custom_llm_provider="bedrock",
    logging_obj=logging_obj,
)
```

**Vertex AI 示例**:
```python
# 1. 创建 Vertex 特有迭代器
model_response_iterator = ModelResponseIterator(
    streaming_response=response.aiter_lines(),
    sync_stream=False,
    logging_obj=logging_obj,
    response_headers=dict(response.headers),
)

# 2. 包装为 CustomStreamWrapper
return CustomStreamWrapper(
    completion_stream=model_response_iterator,
    model=model,
    custom_llm_provider="vertex_ai",
    logging_obj=logging_obj,
)
```

#### 方式 2: 通过 Config 类的 get_model_response_iterator()

**Cohere 示例** (`cohere/chat/transformation.py:358-368`):
```python
class CohereChatConfig(BaseConfig):
    
    def get_model_response_iterator(
        self,
        streaming_response: Union[Iterator[str], AsyncIterator[str], ModelResponse],
        sync_stream: bool,
        json_mode: Optional[bool] = False,
    ):
        # 根据版本选择不同的迭代器
        # V1: ModelResponseIterator
        # V2: CohereV2ModelResponseIterator
        return CohereModelResponseIterator(
            streaming_response=streaming_response,
            sync_stream=sync_stream,
            json_mode=json_mode,
        )
```

---

## 6. Streaming 回调与非流式分叉点

### 6.1 分叉点总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    流式 vs 非流式路径分叉点总览                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  litellm.completion()                                                       │
│        │                                                                    │
│        ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 分叉点 1: main.py - 主入口                                           │  │
│  │                                                                       │  │
│  │ if stream is True:                                                    │  │
│  │     # 流式路径                                                        │  │
│  │     model_response = ModelResponseStream()                           │  │
│  │     # 调用 provider 的 xxx_stream_function                           │  │
│  │ else:                                                                 │  │
│  │     # 非流式路径                                                      │  │
│  │     model_response = ModelResponse()                                 │  │
│  │     # 调用 provider 的 xxx_function                                  │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                               │
│          ┌───────────────────┴───────────────────┐                       │
│          │                                       │                       │
│          ▼                                       ▼                       │
│  ┌──────────────────┐              ┌──────────────────┐                 │
│  │    流式路径       │              │    非流式路径     │                 │
│  └────────┬─────────┘              └────────┬─────────┘                 │
│           │                                   │                           │
│           ▼                                   ▼                           │
│  ┌─────────────────────────────────────┐    ┌─────────────────────────┐ │
│  │ 分叉点 2: Provider Handler          │    │ 非流式响应处理          │ │
│  │                                      │    │                         │ │
│  │ Bedrock:                             │    │ - 直接调用 /converse    │ │
│  │ if stream is True:                   │    │   或 /invoke            │ │
│  │     endpoint = "/converse-stream"    │    │                         │ │
│  │     return async_streaming()         │    │ - 返回 ModelResponse    │ │
│  │ else:                                │    │ - 触发 success_handler  │ │
│  │     endpoint = "/converse"           │    │   (非流式)             │ │
│  │     return async_completion()        │    └─────────────────────────┘ │
│  │                                      │                                  │
│  │ Vertex AI:                           │                                  │
│  │ if "stream" in optional_params:      │                                  │
│  │     url += ":streamGenerateContent"   │                                  │
│  └───────────────────┬──────────────────┘                                  │
│                      │                                                      │
│                      ▼                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 分叉点 3: 响应包装                                                    │  │
│  │                                                                       │  │
│  │ 流式:                                                                 │  │
│  │ - 包装为 CustomStreamWrapper                                         │  │
│  │ - 包含: completion_stream (迭代器), logging_obj, callbacks          │  │
│  │                                                                       │  │
│  │ 非流式:                                                               │  │
│  │ - 直接返回 ModelResponse                                              │  │
│  │ - 无迭代器                                                            │  │
│  └───────────────────────────┬───────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 分叉点 4: 回调触发时机                                                │  │
│  │                                                                       │  │
│  │ 流式:                                                                 │  │
│  │ - 每个 chunk: _call_post_streaming_deployment_hook()                │  │
│  │ - 流结束时: _handle_logging_completed_response()                     │  │
│  │                                                                       │  │
│  │ 非流式:                                                               │  │
│  │ - 响应返回后: 触发 success_handler (一次性)                          │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

### 6.2 Bedrock 分叉点详解

#### 端点选择

**位置**: `converse_handler.py:358-363`

```python
### SET RUNTIME ENDPOINT ###
endpoint_url, proxy_endpoint_url = self.get_runtime_endpoint(...)

if (stream is not None and stream is True) and not fake_stream:
    # 流式端点
    endpoint_url = f"{endpoint_url}/model/{modelId}/converse-stream"
    proxy_endpoint_url = f"{proxy_endpoint_url}/model/{modelId}/converse-stream"
else:
    # 非流式端点
    endpoint_url = f"{endpoint_url}/model/{modelId}/converse"
    proxy_endpoint_url = f"{proxy_endpoint_url}/model/{modelId}/converse"
```

#### 方法分发

**位置**: `converse_handler.py:374-416` (acompletion 分支)

```python
### ROUTING (ASYNC, STREAMING, SYNC)
if acompletion:
    if isinstance(client, HTTPHandler):
        client = None
    if stream is True:
        # 流式路径
        return self.async_streaming(
            model=model,
            messages=messages,
            api_base=proxy_endpoint_url,
            model_response=model_response,
            encoding=encoding,
            logging_obj=logging_obj,
            optional_params=optional_params,
            stream=True,
            litellm_params=litellm_params,
            logger_fn=logger_fn,
            headers=headers,
            timeout=timeout,
            client=client,
            json_mode=json_mode,
            fake_stream=fake_stream,
            credentials=credentials,
            api_key=api_key,
            stream_chunk_size=stream_chunk_size,
        )
    ### ASYNC COMPLETION (非流式)
    return self.async_completion(
        model=model,
        messages=messages,
        api_base=proxy_endpoint_url,
        model_response=model_response,
        encoding=encoding,
        logging_obj=logging_obj,
        optional_params=optional_params,
        stream=stream,
        litellm_params=litellm_params,
        logger_fn=logger_fn,
        headers=headers,
        timeout=timeout,
        client=client,
        credentials=credentials,
        api_key=api_key,
    )
```

#### async_streaming() vs async_completion()

**async_streaming()** (`converse_handler.py:92-161`):
```python
async def async_streaming(
    self,
    model: str,
    messages: list,
    ...
) -> CustomStreamWrapper:
    # 1. 转换请求
    request_data = await litellm.AmazonConverseConfig()._async_transform_request(...)
    data = json.dumps(request_data)
    
    # 2. 签名请求
    prepped = self.get_request_headers(...)
    
    # 3. 发起调用，获取流式迭代器
    completion_stream = await make_call(...)
    
    # 4. 包装为 CustomStreamWrapper (关键区别)
    streaming_response = CustomStreamWrapper(
        completion_stream=completion_stream,
        model=model,
        custom_llm_provider="bedrock",
        logging_obj=logging_obj,
    )
    return streaming_response  # 返回迭代器包装器
```

**async_completion()** (`converse_handler.py:163-249`):
```python
async def async_completion(
    self,
    model: str,
    messages: list,
    ...
) -> Union[ModelResponse, CustomStreamWrapper]:
    # 1. 转换请求
    request_data = await litellm.AmazonConverseConfig()._async_transform_request(...)
    data = json.dumps(request_data)
    
    # 2. 签名请求
    prepped = self.get_request_headers(...)
    
    # 3. 发起 POST，等待完整响应
    response = await client.post(url=api_base, headers=headers, data=data, ...)
    response.raise_for_status()
    
    # 4. 转换响应为 ModelResponse
    return litellm.AmazonConverseConfig()._transform_response(
        model=model,
        response=response,
        model_response=model_response,
        stream=stream if isinstance(stream, bool) else False,
        ...
    )  # 返回完整的 ModelResponse
```

---

### 6.3 Vertex AI 分叉点详解

#### URL 构造

**位置**: `vertex_and_google_ai_studio_gemini.py` (约)

```python
# 构建 Vertex AI URL
url = f"{endpoint}/publishers/google/models/{model}:{method}"

# 流式 vs 非流式 method
if stream:
    method = "streamGenerateContent"  # 流式
else:
    method = "generateContent"       # 非流式
```

#### 响应处理

**流式路径**:
```python
# 1. 获取流式响应
response = await client.post(
    url,
    headers=headers,
    json=request_body,
    stream=True,  # 关键: httpx 流式
)

# 2. 创建流式迭代器
model_response_iterator = ModelResponseIterator(
    streaming_response=response.aiter_lines(),
    sync_stream=False,
    logging_obj=logging_obj,
    response_headers=dict(response.headers),
)

# 3. 包装为 CustomStreamWrapper
return CustomStreamWrapper(
    completion_stream=model_response_iterator,
    model=model,
    custom_llm_provider="vertex_ai",
    logging_obj=logging_obj,
)
```

**非流式路径**:
```python
# 1. 获取完整响应
response = await client.post(
    url,
    headers=headers,
    json=request_body,
    stream=False,  # 关键: 非流式
)

# 2. 直接转换为 ModelResponse
return VertexGeminiConfig().transform_response(
    model=model,
    raw_response=response,
    model_response=model_response,
    logging_obj=logging_obj,
    optional_params=optional_params,
    litellm_params=litellm_params,
    request_data=request_body,
    messages=messages,
    encoding=encoding,
)
```

---

### 6.4 Cohere 分叉点详解

#### 参数传递

**位置**: `cohere/chat/transformation.py:156-180`

```python
def map_openai_params(
    self,
    non_default_params: dict,
    optional_params: dict,
    model: str,
    drop_params: bool,
) -> dict:
    for param, value in non_default_params.items():
        if param == "stream":
            # 传递 stream 参数到 Cohere API
            optional_params["stream"] = value
        # ... 其他参数
    return optional_params
```

#### 端点选择

Cohere V1/V2 端点不同，但 stream 参数是通过 request body 传递的：

```python
# V1: /chat
# V2: /v2/chat

# 请求体中的 stream 参数
data = json.dumps({
    "message": most_recent_message,
    "stream": stream,  # 在请求体中
    "chat_history": chat_history,
    ...
})
```

---

### 6.5 流式回调机制

#### _call_post_streaming_deployment_hook()

**位置**: `streaming_handler.py` (约)

```python
async def _call_post_streaming_deployment_hook(
    self,
    model_response: ModelResponseStream,
    start_time: Optional[float] = None,
    end_time: Optional[float] = None,
):
    """
    每个流式 chunk 返回前调用的钩子
    
    触发时机:
    - 在 __next__ / __anext__ 中，return_processed_chunk_logic() 之后
    - 在 return 给用户之前
    """
    if not self.callbacks:
        return
    
    for callback in self.callbacks:
        if hasattr(callback, "async_post_call_streaming_hook"):
            await callback.async_post_call_streaming_hook(
                model_response=model_response,
                model=self.model,
                custom_llm_provider=self.custom_llm_provider,
                start_time=start_time,
                end_time=end_time,
            )
```

#### _handle_logging_completed_response()

**位置**: `streaming_handler.py` (约)

```python
def _handle_logging_completed_response(self):
    """
    流式结束时的处理
    
    触发时机:
    - 当 StopIteration / StopAsyncIteration 被捕获时
    - 或者当 received_finish_reason 被设置时
    """
    if self.logging_obj is not None:
        # 1. 构建完整的 model_response (用于 logging)
        complete_streamed_response = ModelResponse()
        complete_streamed_response.choices = self._hidden_params.get(
            "complete_streamed_response", []
        )
        complete_streamed_response.usage = self.accumulated_usage
        
        # 2. 触发 success_handler (非流式风格的回调)
        self.logging_obj.model_response = complete_streamed_response
        asyncio.run(self.logging_obj.success_handler())
```

#### Proxy 层流式钩子

**位置**: `litellm/proxy/utils.py:2262` (约)

```python
async def async_post_call_streaming_iterator_hook(
    self,
    response: Union[CustomStreamWrapper, ModelResponse, ModelResponseStream],
    start_time: float,
    user_connected_model: str,
    original_data: dict,
    call_type: str = "completion",
    *args,
    **kwargs,
) -> AsyncIterator:
    """
    Proxy 层的流式迭代器钩子
    
    功能:
    1. Guardrails 检查
    2. 自定义回调执行
    3. Usage 累积和报告
    """
    if isinstance(response, CustomStreamWrapper):
        async for chunk in response:
            # 每个 chunk 的处理
            # - Guardrails 检查
            # - 自定义流式回调
            yield chunk
        
        # 流结束后的处理
        # - 触发 success_handler
        # - 记录 usage
    else:
        # 非流式响应
        yield response
```

---

## 7. 关键代码位置速查

### 7.1 Bedrock

| 功能 | 文件 | 行号 (约) |
|-----|------|----------|
| Converse Handler | `litellm/llms/bedrock/chat/converse_handler