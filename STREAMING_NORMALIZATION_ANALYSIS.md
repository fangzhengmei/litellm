# LiteLLM 跨 Provider 流式响应归一化分析报告

> 分析日期：2026-05-02  
> 分析范围：LiteLLM 流式响应处理机制

---

## 1. 概述

LiteLLM 作为一个统一的 LLM 接口库，需要处理来自 100+ 不同 provider 的流式响应格式差异。本报告深入分析 LiteLLM 如何：

1. **抹平各 provider 流式格式差异** - 将 Anthropic、Bedrock、Vertex AI、Cohere 等不同格式的流式响应统一转换为 OpenAI 兼容格式
2. **归一化数据块传递路径** - 追踪数据从原始响应到用户可迭代对象的完整流程
3. **streaming 与非流式路径分叉点** - 分析代码中决定走流式还是非流式路径的关键决策点

---

## 2. 核心组件架构

### 2.1 主要组件层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户代码层                                 │
│  for chunk in litellm.completion(model="...", stream=True):     │
│      print(chunk)                                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│              CustomStreamWrapper (核心流式包装器)                │
│  - __next__() / __anext__() - 同步/异步迭代                     │
│  - chunk_creator() - 统一格式转换入口                            │
│  - handle_*_chunk() - 各 provider 特定处理                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ModelResponse  │   │ModelResponse  │   │ModelResponse  │
│Iterator       │   │Iterator       │   │Iterator       │
│(Anthropic)    │   │(OpenAI)       │   │(Bedrock)      │
└───────────────┘   └───────────────┘   └───────────────┘
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│Anthropic SSE  │   │OpenAI SDK    │   │AWS Bedrock   │
│原生响应       │   │流式响应      │   │EventStream   │
└───────────────┘   └───────────────┘   └───────────────┘
```

### 2.2 核心类与文件位置

| 组件 | 文件路径 | 主要职责 |
|------|----------|----------|
| `CustomStreamWrapper` | `litellm/litellm_core_utils/streaming_handler.py:98` | 统一流式包装器，实现同步/异步迭代协议 |
| `ChunkProcessor` | `litellm/litellm_core_utils/streaming_chunk_builder_utils.py:35` | 流式 chunk 处理，用于合并多个 chunk 为完整响应 |
| `ModelResponseIterator` (Anthropic) | `litellm/llms/anthropic/chat/handler.py:522` | Anthropic 特定的流式迭代器 |
| `ModelResponseStream` | `litellm/types/utils.py` | 归一化后的流式响应类型（OpenAI 格式） |
| `GenericStreamingChunk` | `litellm/types/utils.py` | 中间格式，用于 provider 与 `CustomStreamWrapper` 之间的通信 |

---

## 3. 各 Provider 流式格式差异及抹平策略

### 3.1 主要 Provider 原生流式格式对比

| Provider | 原生格式 | 数据块结构特点 | 关键差异 |
|----------|----------|----------------|----------|
| **OpenAI** | Server-Sent Events (SSE) | `{"choices": [{"delta": {"content": "..."}}]}` | 标准格式，其他 provider 都需要向此对齐 |
| **Anthropic** | Server-Sent Events (SSE) | `{"type": "content_block_delta", "delta": {"type": "text_delta", "text": "..."}}` | 使用 `content_block_delta` 而非 `delta`，需要转换为 OpenAI 格式 |
| **Bedrock (Converse)** | AWS EventStream | `{"contentBlockDelta": {"delta": {"text": "..."}}}` | AWS 特有事件流格式，字段命名不同 |
| **Vertex AI (Gemini)** | gRPC Stream | `{"candidates": [{"content": {"parts": [{"text": "..."}]}}]}` | 使用 `candidates` 而非 `choices`，`parts` 而非 `delta` |
| **Cohere** | JSON Stream | `{"text": "...", "is_finished": false}` | 极简格式，需要包装为 OpenAI 结构 |
| **Replicate** | Polling-based | `{"output": "...", "status": "succeeded"}` | 非真正流式，通过轮询模拟 |

### 3.2 Anthropic 格式转换详解

Anthropic 使用与 OpenAI 完全不同的流式事件类型，需要进行多层转换。

#### 原生 Anthropic 流式事件类型：

```typescript
// Anthropic 消息开始事件
{
  "type": "message_start",
  "message": {
    "id": "msg_xxx",
    "type": "message",
    "role": "assistant",
    "content": [],
    "model": "claude-3-opus-20240229",
    "usage": {"input_tokens": 10, "output_tokens": 1}
  }
}

// Anthropic 内容块开始事件
{
  "type": "content_block_start",
  "index": 0,
  "content_block": {"type": "text", "text": ""}
}

// Anthropic 内容块增量事件（核心数据）
{
  "type": "content_block_delta",
  "index": 0,
  "delta": {"type": "text_delta", "text": "Hello"}
}

// Anthropic 内容块结束事件
{
  "type": "content_block_stop",
  "index": 0
}

// Anthropic 消息增量事件（包含 usage）
{
  "type": "message_delta",
  "delta": {"stop_reason": "end_turn"},
  "usage": {"output_tokens": 15}
}
```

#### 转换为 OpenAI 格式的过程：

**文件位置**: `litellm/llms/anthropic/chat/handler.py:522` - `ModelResponseIterator` 类

```python
class ModelResponseIterator:
    """
    Anthropic 流式响应迭代器，负责将 Anthropic SSE 事件转换为 OpenAI 格式
    """
    
    def _content_block_delta_helper(self, chunk: dict) -> Tuple[
        str,
        Optional[ChatCompletionToolCallChunk],
        List[Union[ChatCompletionThinkingBlock, ChatCompletionRedactedThinkingBlock]],
        Dict[str, Any],
    ]:
        """
        处理 Anthropic 的 content_block_delta 事件
        
        转换逻辑：
        - text_delta → delta.content
        - input_json_delta → delta.tool_calls[].function.arguments
        - thinking/signature → thinking_blocks (provider_specific_fields)
        """
        text = ""
        tool_use: Optional[ChatCompletionToolCallChunk] = None
        thinking_blocks: List[...] = []
        provider_specific_fields = {}
        
        content_block = ContentBlockDelta(**chunk)
        self.content_blocks.append(content_block)
        
        if "text" in content_block["delta"]:
            # 普通文本增量 → OpenAI delta.content
            text = content_block["delta"]["text"]
        elif "partial_json" in content_block["delta"]:
            # 工具调用参数增量 → OpenAI tool_calls
            if self.current_content_block_type in ("tool_use", "server_tool_use"):
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
            # 推理内容 → thinking_blocks (Anthropic 特有)
            thinking_blocks = [
                ChatCompletionThinkingBlock(
                    type="thinking",
                    thinking=content_block["delta"].get("thinking") or "",
                    signature=str(content_block["delta"].get("signature") or ""),
                )
            ]
            provider_specific_fields["thinking_blocks"] = thinking_blocks
        
        return text, tool_use, thinking_blocks, provider_specific_fields
```

**转换结果示例**：

```typescript
// 转换后的 OpenAI 格式
{
  "id": "chatcmpl-xxx",
  "object": "chat.completion.chunk",
  "created": 1714567890,
  "model": "claude-3-opus-20240229",
  "choices": [{
    "index": 0,
    "delta": {
      "role": "assistant",
      "content": "Hello"
    },
    "finish_reason": null
  }]
}
```

### 3.3 CustomStreamWrapper 中的 Provider 分发

**文件位置**: `litellm/litellm_core_utils/streaming_handler.py:1121` - `chunk_creator` 方法

```python
def chunk_creator(self, chunk: Any):
    """
    核心分发方法：根据 custom_llm_provider 选择对应的 chunk 处理器
    """
    model_response = self.model_response_creator()
    completion_obj: Dict[str, Any] = {"content": ""}
    
    # 检查是否是通用格式的 chunk（GenericStreamingChunk）
    if (
        isinstance(chunk, dict)
        and generic_chunk_has_all_required_fields(chunk=chunk)
    ) or (
        self.custom_llm_provider
        and self.custom_llm_provider in litellm._custom_providers
    ):
        # 通用格式：直接读取标准字段
        anthropic_response_obj: GChunk = cast(GChunk, chunk)
        completion_obj["content"] = anthropic_response_obj["text"]
        if anthropic_response_obj["is_finished"]:
            self.received_finish_reason = anthropic_response_obj["finish_reason"]
        if anthropic_response_obj["usage"] is not None:
            setattr(model_response, "usage", litellm.Usage(...))
        if anthropic_response_obj["tool_use"] is not None:
            completion_obj["tool_calls"] = [anthropic_response_obj["tool_use"]]
    
    # Provider 特定处理分支
    elif self.custom_llm_provider == "replicate":
        response_obj = self.handle_replicate_chunk(chunk)
        completion_obj["content"] = response_obj["text"]
        
    elif self.custom_llm_provider == "predibase":
        response_obj = self.handle_predibase_chunk(chunk)
        completion_obj["content"] = response_obj["text"]
        
    elif self.custom_llm_provider == "cohere":
        response_obj = self.handle_cohere_chunk(chunk)
        completion_obj["content"] = response_obj["text"]
        
    elif self.custom_llm_provider == "vertex_ai" and not isinstance(chunk, ModelResponseStream):
        # Vertex AI Gemini 原生格式处理
        if hasattr(chunk, "candidates"):
            completion_obj["content"] = chunk.text
            
    # ... 更多 provider 分支
    
    else:  # OpenAI / Azure 兼容格式（默认分支）
        response_obj = self.handle_openai_chat_completion_chunk(chunk)
        completion_obj["content"] = response_obj["text"]
```

### 3.4 GenericStreamingChunk 中间格式

**文件位置**: `litellm/types/utils.py`

为了简化各 provider 的转换工作，LiteLLM 定义了 `GenericStreamingChunk` 作为中间格式：

```python
GenericStreamingChunk = TypedDict(
    "GenericStreamingChunk",
    {
        "text": str,                    # 文本内容
        "is_finished": bool,            # 是否结束
        "finish_reason": Optional[str], # 结束原因
        "usage": Optional[dict],        # token 使用量
        "tool_use": Optional[dict],     # 工具调用
        "provider_specific_fields": Optional[dict],  # provider 特有字段
    },
    total=False,
)
```

各 provider 只需将其原生格式转换为此中间格式，`CustomStreamWrapper` 会自动处理后续的 OpenAI 格式转换。

---

## 4. 归一化数据块的传递路径

### 4.1 完整数据流图

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           请求发起阶段                                      │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  litellm/completion() / acompletion()                                     │
│  - 解析 stream 参数                                                        │
│  - 根据 model 确定 custom_llm_provider                                    │
│  - 分发到对应 provider 的 completion 方法                                   │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  Provider 特定 completion 方法 (如 AnthropicChatCompletion.completion())  │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ if stream is True:                                                    │ │
│  │     → 调用 acompletion_stream_function() / 同步版本                   │ │
│  │     → 创建 ModelResponseIterator (provider 特定迭代器)               │ │
│  │     → 包装为 CustomStreamWrapper 并返回                               │ │
│  │ else:                                                                 │ │
│  │     → 调用 acompletion_function()                                     │ │
│  │     → 等待完整响应后转换为 ModelResponse                               │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼ (stream=True)
┌──────────────────────────────────────────────────────────────────────────┐
│  CustomStreamWrapper 初始化                                                │
│  - completion_stream = ModelResponseIterator (provider 迭代器)            │
│  - model = 模型名称                                                        │
│  - custom_llm_provider = provider 标识                                    │
│  - logging_obj = 日志对象                                                  │
│  - stream_options = 流式选项 (如 include_usage)                           │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼ (用户开始迭代)
┌──────────────────────────────────────────────────────────────────────────┐
│  CustomStreamWrapper.__next__() / __anext__()                            │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │ 1. 从 completion_stream 获取下一个原始 chunk                          │ │
│  │    chunk = next(self.completion_stream)                               │ │
│  │                                                                         │ │
│  │ 2. 调用 chunk_creator() 进行格式转换                                   │ │
│  │    response = self.chunk_creator(chunk=chunk)                        │ │
│  │                                                                         │ │
│  │ 3. 处理 stream_options (如 include_usage)                              │ │
│  │    - 如果有 usage，从 chunk 中剥离，存到 self.chunks                  │ │
│  │                                                                         │ │
│  │ 4. 日志记录和回调                                                       │ │
│  │    executor.submit(self.run_success_logging_and_cache_storage, ...)  │ │
│  │                                                                         │ │
│  │ 5. 返回归一化后的 ModelResponseStream                                  │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
└───────────────────────────┬──────────────────────────────────────────────┘
                            │
                            ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  用户代码接收 ModelResponseStream                                          │
│  {                                                                        │
│      "id": "chatcmpl-xxx",                                               │
│      "object": "chat.completion.chunk",                                  │
│      "created": 1714567890,                                              │
│      "model": "gpt-4",                                                    │
│      "choices": [{                                                        │
│          "index": 0,                                                      │
│          "delta": {"content": "Hello"},                                   │
│          "finish_reason": null                                            │
│      }]                                                                   │
│  }                                                                        │
└──────────────────────────────────────────────────────────────────────────┘
```

### 4.2 关键数据结构转换过程

#### 阶段 1：Provider 原始响应 → ModelResponseIterator

以 Anthropic 为例：

```python
# 文件: litellm/llms/anthropic/chat/handler.py:115
async def make_call(client, api_base, headers, data, model, messages, logging_obj, ...):
    response = await client.post(api_base, headers=headers, data=data, stream=True, ...)
    
    # 包装为 Anthropic 特定的迭代器
    completion_stream = ModelResponseIterator(
        streaming_response=response.aiter_lines(),  # 原始 SSE 行迭代器
        sync_stream=False,
        json_mode=json_mode,
        speed=speed,
    )
    
    return completion_stream, response.headers
```

#### 阶段 2：ModelResponseIterator → GenericStreamingChunk / ModelResponseStream

`ModelResponseIterator` 负责将原生 SSE 事件解析并转换：

```python
# 文件: litellm/llms/anthropic/chat/handler.py:ModelResponseIterator
def __next__(self) -> ModelResponseStream:
    # 解析 SSE 行，识别事件类型
    for line in self.streaming_response:
        if line.startswith("data: "):
            event = json.loads(line[6:])
            event_type = event.get("type")
            
            if event_type == "content_block_delta":
                # 转换为 OpenAI 格式的 chunk
                text, tool_use, thinking_blocks, provider_fields = \
                    self._content_block_delta_helper(event)
                
                # 构建 ModelResponseStream
                return ModelResponseStream(
                    choices=[StreamingChoices(
                        delta=Delta(content=text),
                        finish_reason=None
                    )]
                )
            
            elif event_type == "message_delta":
                # 处理结束事件
                self.received_finish_reason = event["delta"].get("stop_reason")
                # ...
```

#### 阶段 3：CustomStreamWrapper 统一处理

无论 provider 如何，最终都通过 `CustomStreamWrapper` 的 `chunk_creator` 进行统一：

```python
# 文件: litellm/litellm_core_utils/streaming_handler.py:1829
def __next__(self) -> ModelResponseStream:
    while True:
        # 获取原始 chunk（可能是 provider 特定格式或 ModelResponseStream）
        chunk = next(self.completion_stream)
        
        # 统一格式转换
        response = self.chunk_creator(chunk=chunk)
        
        if response is None:
            continue
        
        # 记录到 chunks 列表（用于 stream_options 和最终 usage 计算）
        self.chunks.append(response)
        
        # 处理 usage（如果 stream_options.include_usage=True，在最后一个 chunk 返回）
        if hasattr(response, "usage") and not self.send_stream_usage:
            # 暂存 usage，不立即返回
            obj_dict = response.model_dump()
            del obj_dict["usage"]
            response = self.model_response_creator(chunk=obj_dict, ...)
        
        return response
```

### 4.3 流式响应的结束处理

当流式响应结束时，`CustomStreamWrapper` 会进行特殊处理：

```python
# 文件: litellm/litellm_core_utils/streaming_handler.py:1922
except StopIteration:
    if self.sent_last_chunk is True:
        # 已经发送过结束 chunk，现在需要构建完整响应（用于日志和缓存）
        complete_streaming_response = litellm.stream_chunk_builder(
            chunks=self.chunks,
            messages=self.messages,
            logging_obj=self.logging_obj,
        )
        
        if self.send_stream_usage is True and self.sent_stream_usage is False:
            # 如果要求 stream_options.include_usage=True，发送包含 usage 的最终 chunk
            self.sent_stream_usage = True
            response = self.model_response_creator()
            if complete_streaming_response is not None:
                setattr(response, "usage", getattr(complete_streaming_response, "usage"))
            return response
        
        raise  # 重新抛出 StopIteration
    else:
        # 第一次遇到 StopIteration，发送结束 chunk
        self.sent_last_chunk = True
        processed_chunk = self.finish_reason_handler()
        
        # 添加 usage 到 _hidden_params（供 proxy 使用）
        if self.stream_options is None:
            usage = calculate_total_usage(chunks=self.chunks)
            processed_chunk._hidden_params["usage"] = usage
        
        return processed_chunk
```

---

## 5. Streaming 回调与非流式路径的关键分叉点

### 5.1 主入口分叉点

**文件位置**: `litellm/main.py` - `completion()` 函数

```python
# 文件: litellm/main.py:1060
def completion(model: str, messages: List = [], ..., stream: Optional[bool] = None, ...):
    # ... 参数预处理 ...
    
    # 根据 custom_llm_provider 分发到不同的 provider 处理
    if custom_llm_provider == "azure":
        # Azure 分支
        response = azure_chat_completions.completion(
            model=model,
            messages=messages,
            # ...
            acompletion=acompletion,  # 异步标识
            # ...
        )
        if optional_params.get("stream", False):
            # 流式响应的特殊日志处理
            logging.post_call(...)
            
    elif custom_llm_provider == "anthropic":
        # Anthropic 分支 - 看下面的详细分析
        
    # ... 更多 provider 分支
```

### 5.2 Provider 层分叉点 (Anthropic 示例)

**文件位置**: `litellm/llms/anthropic/chat/handler.py:321`

```python
def completion(self, model: str, messages: list, ..., acompletion=None, ...):
    # ... 准备工作 ...
    
    stream = optional_params.pop("stream", None)  # 提取 stream 参数
    
    if acompletion is True:
        # 异步调用分支
        if stream is True:
            # 【关键分叉点 1】异步流式
            print_verbose("makes async anthropic streaming POST request")
            data["stream"] = stream
            return self.acompletion_stream_function(
                model=model,
                messages=messages,
                data=data,
                # ...
            )
        else:
            # 【关键分叉点 2】异步非流式
            return self.acompletion_function(
                model=model,
                messages=messages,
                data=data,
                # ...
            )
    else:
        # 同步调用分支
        if stream is True:
            # 【关键分叉点 3】同步流式
            data["stream"] = stream
            completion_stream, headers = make_sync_call(...)
            return CustomStreamWrapper(
                completion_stream=completion_stream,
                model=model,
                custom_llm_provider="anthropic",
                logging_obj=logging_obj,
                _response_headers=process_anthropic_headers(headers),
            )
        else:
            # 【关键分叉点 4】同步非流式
            response = client.post(api_base, headers=headers, data=json.dumps(data), ...)
            return config.transform_response(
                model=model,
                raw_response=response,
                # ...
            )
```

### 5.3 分叉点决策树

```
completion() / acompletion() 入口
         │
         ▼
    ┌─────────┐
    │acompletion│
    │  = True? │
    └────┬────┘
         │
    ┌────┴────┐
    │         │
    ▼         ▼
  异步路径   同步路径
    │         │
    ▼         ▼
┌─────────┐ ┌─────────┐
│stream   │ │stream   │
│= True?  │ │= True?  │
└────┬────┘ └────┬────┘
     │           │
 ┌───┴───┐   ┌───┴───┐
 │       │   │       │
 ▼       ▼   ▼       ▼
异步流式 异步非流 同步流式 同步非流
         │           │
         ▼           ▼
   返回 ModelResponse  返回 CustomStreamWrapper
   (完整响应对象)      (可迭代对象)
```

### 5.4 两种路径的返回类型对比

| 路径 | 返回类型 | 处理方式 |
|------|----------|----------|
| **流式 (stream=True)** | `CustomStreamWrapper` | 实现 `__iter__` / `__aiter__` 协议，用户通过 `for chunk in response:` 迭代 |
| **非流式 (stream=False)** | `ModelResponse` | 完整的响应对象，包含 `choices[0].message.content` 等完整字段 |

**代码中的类型定义**：

```python
# 文件: litellm/main.py:430
async def acompletion(...) -> Union[ModelResponse, CustomStreamWrapper]:
    # ...

# 文件: litellm/main.py:1112
def completion(...) -> Union[ModelResponse, CustomStreamWrapper]:
    # ...
```

### 5.5 回调处理的差异

#### 流式路径的回调

**文件位置**: `litellm/litellm_core_utils/streaming_handler.py:1783`

```python
def run_success_logging_and_cache_storage(self, processed_chunk, cache_hit: bool):
    """
    流式响应的回调：每个 chunk 都会触发
    """
    if litellm.disable_streaming_logging is True:
        return
    
    # 异步日志（在线程池中运行，避免阻塞流式输出）
    if self.logging_loop is not None:
        future = asyncio.run_coroutine_threadsafe(
            self.logging_obj.async_success_handler(
                processed_chunk, None, None, cache_hit
            ),
            loop=self.logging_loop,
        )
        future.result()
    else:
        asyncio.run(
            self.logging_obj.async_success_handler(
                processed_chunk, None, None, cache_hit
            )
        )
    
    # 同步日志
    self.logging_obj.success_handler(processed_chunk, None, None, cache_hit)
```

**调用位置**：在 `__next__` / `__anext__` 中，每返回一个 chunk 前调用：

```python
# 文件: litellm/litellm_core_utils/streaming_handler.py:1866
if not litellm.disable_streaming_logging:
    executor.submit(
        self.run_success_logging_and_cache_storage,
        response,
        cache_hit,
    )  # 在线程池中异步执行，不阻塞流式输出
```

#### 非流式路径的回调

非流式路径的回调在 `completion()` 函数的末尾统一调用：

```python
# 文件: litellm/main.py（非流式分支）
# 通常在 provider 的 transform_response 之后
logging_obj.success_handler(
    model_response,  # 完整响应
    original_response,  # 原始响应
    start_time,  # 开始时间
    cache_hit,  # 是否缓存命中
)
```

### 5.6 关键分叉点总结表

| 分叉点位置 | 决策条件 | 流式路径 | 非流式路径 |
|------------|----------|----------|------------|
| `litellm/main.py:completion()` | `optional_params.get("stream", False)` | 传递 `stream=True` 给 provider | 传递 `stream=False` 或不传递 |
| `litellm/llms/*/chat/handler.py:completion()` | `stream is True` | 调用 `*_stream_function()` | 调用 `*_function()` |
| `litellm/llms/anthropic/chat/handler.py:203` | `acompletion is True and stream is True` | 异步流式请求，返回 `CustomStreamWrapper` | - |
| `litellm/llms/anthropic/chat/handler.py:448` | `acompletion is False and stream is True` | 同步流式请求，返回 `CustomStreamWrapper` | - |
| `litellm/litellm_core_utils/streaming_handler.py:1829` | `__next__` / `__anext__` 迭代 | 逐个返回 `ModelResponseStream` | - |
| 日志回调 | 流式 vs 非流式 | 每 chunk 调用一次，在线程池执行 | 响应完成后调用一次 |

---

## 6. Streaming Options 特殊处理

### 6.1 stream_options.include_usage

当用户设置 `stream_options={"include_usage": True}` 时，LiteLLM 需要在流式响应的最后一个 chunk 中包含 `usage` 字段。

**文件位置**: `litellm/litellm_core_utils/streaming_handler.py:225`

```python
def check_send_stream_usage(self, stream_options: Optional[dict]):
    return (
        stream_options is not None
        and stream_options.get("include_usage", False) is True
    )
```

**处理逻辑**：

```python
# 在 __next__ / __anext__ 中
if hasattr(response, "usage"):
    # 如果当前 chunk 有 usage，先暂存，不立即返回
    self.chunks.append(response.model_copy())  # 保存副本
    
    # 从返回的 chunk 中删除 usage
    obj_dict = response.model_dump()
    if "usage" in obj_dict:
        del obj_dict["usage"]
    response = self.model_response_creator(
        chunk=obj_dict, 
        hidden_params=response._hidden_params
    )

# 在流结束时（StopIteration 处理）
if self.sent_stream_usage is False and self.send_stream_usage is True:
    self.sent_stream_usage = True
    # 构建包含 usage 的最终 chunk
    response = self.model_response_creator()
    complete_streaming_response = litellm.stream_chunk_builder(
        chunks=self.chunks, ...
    )
    if complete_streaming_response is not None:
        setattr(response, "usage", getattr(complete_streaming_response, "usage"))
    return response  # 返回包含 usage 的最终 chunk
```

### 6.2 stream_chunk_builder 的作用

**文件位置**: `litellm/litellm_core_utils/streaming_chunk_builder_utils.py:35` - `ChunkProcessor` 类

当流式响应结束后，`stream_chunk_builder` 用于将所有收集到的 chunk 合并为一个完整的 `ModelResponse`：

```python
class ChunkProcessor:
    def __init__(self, chunks: List, messages: Optional[list] = None):
        self.chunks = self._sort_chunks(chunks)
        self.messages = messages
        self.first_chunk = chunks[0]
    
    def build_base_response(self, chunks: List[Dict[str, Any]]) -> ModelResponse:
        # 从第一个 chunk 获取基础信息
        id = ChunkProcessor._get_chunk_id(chunks)
        object = chunk["object"]
        created = chunk["created"]
        model = chunk["model"]
        
        # 构建完整响应
        response = ModelResponse(
            id=id,
            object=object.replace(".chunk", ""),  # "chat.completion.chunk" → "chat.completion"
            created=created,
            model=model,
            choices=[{
                "index": 0,
                "message": {"role": role, "content": ""},
                "finish_reason": finish_reason,
            }],
            usage=...
        )
        return response
    
    def get_combined_content(self, chunks: List[Dict[str, Any]]) -> str:
        # 合并所有 chunk 的 content
        content_list: List[str] = []
        for chunk in chunks:
            choices = chunk["choices"]
            for choice in choices:
                delta = choice.get("delta", {})
                content = delta.get("content", "")
                if content is not None:
                    content_list.append(content)
        return "".join(content_list)
    
    def get_combined_tool_content(self, tool_call_chunks: List[Dict[str, Any]]) -> List[ChatCompletionMessageToolCall]:
        # 合并所有 chunk 的 tool_calls
        # 处理 tool_calls 的增量拼接
        ...
```

---

## 7. 关键设计模式与架构考量

### 7.1 迭代器模式 (Iterator Pattern)

`CustomStreamWrapper` 实现了 Python 的迭代器协议，使得各种 provider 的流式响应都能以统一的方式被用户消费：

```python
class CustomStreamWrapper:
    def __iter__(self) -> Iterator["ModelResponseStream"]:
        return self
    
    def __aiter__(self) -> AsyncIterator["ModelResponseStream"]:
        return self
    
    def __next__(self) -> "ModelResponseStream":
        # 同步迭代逻辑
        ...
    
    async def __anext__(self) -> "ModelResponseStream":
        # 异步迭代逻辑
        ...
```

### 7.2 适配器模式 (Adapter Pattern)

各 provider 的 `ModelResponseIterator` 和 `handle_*_chunk` 方法本质上是适配器，将 provider 特定的接口适配为 LiteLLM 统一的接口：

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Anthropic    │────▶│ ModelResponse    │────▶│ CustomStream     │
│ SSE Events   │     │ Iterator         │     │ Wrapper          │
└──────────────┘     └──────────────────┘     └──────────────────┘
                                             
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ Bedrock      │────▶│ Bedrock Iterator │────▶│ CustomStream     │
│ EventStream  │     │ (隐式)           │     │ Wrapper          │
└──────────────┘     └──────────────────┘     └──────────────────┘
                                             
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│ OpenAI SDK   │────▶│ handle_openai_   │────▶│ CustomStream     │
│ Stream       │     │ chat_completion_ │     │ Wrapper          │
│              │     │ chunk            │     │                  │
└──────────────┘     └──────────────────┘     └──────────────────┘
```

### 7.3 模板方法模式 (Template Method Pattern)

`CustomStreamWrapper.__next__` / `__anext__` 定义了流式处理的算法骨架，而具体的 provider 处理通过 `chunk_creator` 中的多态分支实现：

```python
def __next__(self):
    # 模板方法：固定的算法骨架
    while True:
        # 步骤 1：获取下一个原始 chunk
        chunk = next(self.completion_stream)
        
        # 步骤 2：转换格式（由具体子类/分支实现）
        response = self.chunk_creator(chunk=chunk)
        
        # 步骤 3：处理 stream_options
        if hasattr(response, "usage") and not self.send_stream_usage:
            # ... 暂存 usage
        
        # 步骤 4：日志和回调
        executor.submit(self.run_success_logging_and_cache_storage, ...)
        
        # 步骤 5：返回结果
        return response
```

### 7.4 同步/异步双支持

LiteLLM 的流式处理同时支持同步和异步迭代，这通过两套独立但逻辑相似的方法实现：

| 特性 | 同步实现 | 异步实现 |
|------|----------|----------|
| 迭代协议 | `__iter__`, `__next__` | `__aiter__`, `__anext__` |
| 从底层获取数据 | `next(self.completion_stream)` | `async for chunk in self.completion_stream` 或 `asyncio.to_thread()` |
| 日志执行 | `executor.submit()` (线程池) | 直接 `await` 或 `executor.submit()` |
| 错误处理 | `StopIteration` | `StopAsyncIteration` |

---

## 8. 总结与关键洞察

### 8.1 流式归一化的核心策略

1. **两层转换架构**：
   - 第一层：Provider 特定迭代器（如 `ModelResponseIterator`）将原生格式转换为中间格式
   - 第二层：`CustomStreamWrapper.chunk_creator()` 将中间格式统一转换为 OpenAI 的 `ModelResponseStream`

2. **中间格式抽象**：
   - `GenericStreamingChunk` 作为 provider 与 `CustomStreamWrapper` 之间的契约
   - 新 provider 只需实现到 `GenericStreamingChunk` 的转换，无需关心 OpenAI 格式细节

3. **延迟处理与增量构建**：
   - 流式响应不等待完整响应，而是逐块处理
   - `self.chunks` 列表用于暂存所有 chunk，供 `stream_options` 和最终日志使用
   - `stream_chunk_builder` 仅在需要时（如流结束、日志记录）才合并完整响应

### 8.2 关键代码位置速查

| 功能 | 文件位置 | 关键方法/类 |
|------|----------|-------------|
| 核心流式包装器 | `litellm/litellm_core_utils/streaming_handler.py` | `CustomStreamWrapper` |
| 同步迭代入口 | `litellm/litellm_core_utils/streaming_handler.py:1829` | `CustomStreamWrapper.__next__()` |
| 异步迭代入口 | `litellm/litellm_core_utils/streaming_handler.py:2021` | `CustomStreamWrapper.__anext__()` |
| Provider 分发 | `litellm/litellm_core_utils/streaming_handler.py:1121` | `CustomStreamWrapper.chunk_creator()` |
| Anthropic 迭代器 | `litellm/llms/anthropic/chat/handler.py:522` | `ModelResponseIterator` |
| Chunk 合并工具 | `litellm/litellm_core_utils/streaming_chunk_builder_utils.py:35` | `ChunkProcessor` |
| 主入口分叉 | `litellm/main.py:1060` | `completion()` 函数 |

### 8.3 扩展新 Provider 的流式支持

为新 provider 添加流式支持需要实现：

1. **Provider 特定迭代器**（可选，推荐用于复杂格式）：
   - 继承或实现类似 `ModelResponseIterator` 的类
   - 将原生流式事件解析为 `GenericStreamingChunk` 或直接返回 `ModelResponseStream`

2. **在 `CustomStreamWrapper.chunk_creator` 添加分支**：
   ```python
   elif self.custom_llm_provider == "new_provider":
       response_obj = self.handle_new_provider_chunk(chunk)
       completion_obj["content"] = response_obj["text"]
       if response_obj["is_finished"]:
           self.received_finish_reason = response_obj["finish_reason"]
   ```

3. **实现 `handle_*_chunk` 方法**：
   - 将 provider 原生 chunk 转换为包含 `text`, `is_finished`, `finish_reason` 等字段的字典

### 8.4 与非流式路径的关系

- **统一入口**：`completion()` / `acompletion()` 是唯一入口，通过 `stream` 参数决定路径
- **返回类型多态**：返回类型是 `Union[ModelResponse, CustomStreamWrapper]`
- **日志回调差异**：
  - 非流式：一次完整回调，包含完整响应
  - 流式：多次增量回调，每个 chunk 一次；流结束时可能有最终合并回调
- **Usage 处理**：
  - 非流式：响应直接包含 `usage`
  - 流式：
    - 默认：`usage` 存储在 `_hidden_params` 中，不在 chunk 中显示
    - `stream_options.include_usage=True`：在最后一个 chunk 中包含 `usage`

---

## 附录：数据结构定义

### A.1 ModelResponseStream (归一化后格式)

```python
class ModelResponseStream(BaseModel):
    id: Optional[str] = Field(default=None)
    object: Optional[str] = Field(default="chat.completion.chunk")
    created: Optional[int] = Field(default_factory=lambda: int(time.time()))
    model: Optional[str] = Field(default=None)
    system_fingerprint: Optional[str] = Field(default=None)
    choices: List[StreamingChoices] = Field(default_factory=list)
    usage: Optional[Usage] = Field(default=None)
    
    # 隐藏参数（用于内部传递元数据）
    _hidden_params: Dict[str, Any] = PrivateAttr(default_factory=dict)
```

### A.2 StreamingChoices

```python
class StreamingChoices(BaseModel):
    delta: Delta = Field(default_factory=Delta)
    finish_reason: Optional[str] = Field(default=None)
    index: int = Field(default=0)
    logprobs: Optional[Any] = Field(default=None)
```

### A.3 Delta

```python
class Delta(BaseModel):
    content: Optional[str] = Field(default=None)
    reasoning_content: Optional[str] = Field(default=None)
    role: Optional[str] = Field(default=None)
    tool_calls: Optional[List[ChatCompletionMessageToolCall]] = Field(default=None)
    function_call: Optional[FunctionCall] = Field(default=None)
    audio: Optional[ChatCompletionAudioDelta] = Field(default=None)
    images: Optional[List[ImageContentBlock]] = Field(default=None)
    annotations: Optional[List[Any]] = Field(default=None)
    thinking_blocks: Optional[List[Any]] = Field(default=None)
    provider_specific_fields: Optional[Dict[str, Any]] = Field(default=None)
```

---

*报告生成完毕*
