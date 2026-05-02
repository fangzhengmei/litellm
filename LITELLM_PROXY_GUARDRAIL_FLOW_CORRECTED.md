# LiteLLM Proxy Guardrail 执行流：修正版深度分析

> 分析日期：2026-05-02
> 代码版本：基于实际代码路径追踪
> 修正内容：显式触发入口、Pipeline 跳过机制、MCP 场景分支

---

## 目录

1. [核心纠偏总结](#1-核心纠偏总结)
2. [显式触发入口：三级优先级](#2-显式触发入口三级优先级)
3. [Pipeline 执行顺序：Pipeline 优先，独立 Guardrails 后行](#3-pipeline-执行顺序pipeline-优先独立-guardrails-后行)
4. [MCP 场景触发分支](#4-mcp-场景触发分支)
5. [完整执行流程图解](#5-完整执行流程图解)
6. [关键代码位置索引](#6-关键代码位置索引)

---

## 1. 核心纠偏总结

| 问题 | 之前理解 | 实际代码行为 |
|------|----------|--------------|
| **显式触发入口** | 只提到 metadata | 三级优先级：根级 `data["guardrails"]` > `metadata["guardrails"]` > `litellm_metadata["guardrails"]` |
| **Pipeline 执行顺序** | 未明确说明 Pipeline 管理的会被跳过 | **Phase A: Pipeline 优先执行** → **Phase B: 独立 Guardrails 执行**，但跳过 `_pipeline_managed_guardrails` 中的 |
| **MCP 场景** | 未详细分析 | 当 `call_type=call_mcp_tool` 时，`pre_call` 转换为 `pre_mcp_call`，`during_call` 转换为 `during_mcp_call` |

---

## 2. 显式触发入口：三级优先级

### 2.1 代码路径追踪

```python
# litellm/integrations/custom_guardrail.py:289-308

def get_guardrail_from_metadata(
    self, data: dict
) -> Union[List[str], List[Dict[str, DynamicGuardrailParams]]]:
    """
    Returns the guardrail(s) to be run from the metadata or root
    
    优先级:
    1. data["guardrails"] (根级别，优先级最高)
    2. data["metadata"]["guardrails"]
    3. data["litellm_metadata"]["guardrails"]
    """

    # ========== 优先级 1: 根级 "guardrails" ==========
    # 调用方可以在请求体根级别直接指定 guardrails
    if "guardrails" in data:
        return data["guardrails"]
    
    # ========== 优先级 2 & 3: metadata 中的 "guardrails" ==========
    # 检查两个可能的 metadata 位置：
    # - "metadata": 普通端点使用
    # - "litellm_metadata": thread/assistant 端点使用
    # 
    # 为什么检查两者？
    # For regular endpoints move_guardrails_to_metadata writes to "metadata";
    # for thread/assistant endpoints it writes to "litellm_metadata".
    # We check the one that actually contains the "guardrails" key so that
    # a non-empty litellm_metadata without guardrails does not shadow
    # the merged list stored in metadata (which would cause team guardrails
    # to be silently skipped while default_on=True policy guardrails still fire).
    for meta_key in ("metadata", "litellm_metadata"):
        meta = data.get(meta_key) or {}
        if isinstance(meta, dict) and "guardrails" in meta:
            return meta.get("guardrails") or []
    
    return []
```

### 2.2 三级优先级图解

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    get_guardrail_from_metadata 三级优先级                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  请求数据结构示例:                                                             │
│                                                                              │
│  {                                                                           │
│    "model": "gpt-4",                                                         │
│    "messages": [...],                                                        │
│                                                                              │
│    ┌────────────────────────────────────────────────────────────────────┐    │
│    │  优先级 1 (最高): 根级 "guardrails"                                  │    │
│    ├────────────────────────────────────────────────────────────────────┤    │
│    │  "guardrails": ["my_guardrail_a", "my_guardrail_b"]              │    │
│    │                                                                     │    │
│    │  如果存在，直接返回，不再检查 metadata                              │    │
│    └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│    "metadata": {                                                             │
│      ┌────────────────────────────────────────────────────────────────────┐ │
│      │  优先级 2: "metadata" 中的 "guardrails"                             │ │
│      ├────────────────────────────────────────────────────────────────────┤ │
│      │  "guardrails": ["policy_guardrail"]                                │ │
│      │                                                                     │ │
│      │  只有根级没有时才检查这里                                           │ │
│      └────────────────────────────────────────────────────────────────────┘ │
│    },                                                                         │
│                                                                              │
│    "litellm_metadata": {                                                      │
│      ┌────────────────────────────────────────────────────────────────────┐ │
│      │  优先级 3: "litellm_metadata" 中的 "guardrails"                      │ │
│      ├────────────────────────────────────────────────────────────────────┤ │
│      │  "guardrails": ["assistant_guardrail"]                             │ │
│      │                                                                     │ │
│      │  thread/assistant 端点使用                                           │ │
│      └────────────────────────────────────────────────────────────────────┘ │
│    }                                                                          │
│  }                                                                           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 实际请求示例

#### 示例 1: 根级指定 (优先级最高)

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}],
    "guardrails": ["my_guardrail_a", "my_guardrail_b"]
  }'
```

**结果**：`get_guardrail_from_metadata` 直接返回 `["my_guardrail_a", "my_guardrail_b"]`

---

#### 示例 2: metadata 中指定

```bash
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}],
    "metadata": {
      "guardrails": ["policy_guardrail"]
    }
  }'
```

**结果**：根级没有 `guardrails`，检查 `metadata`，返回 `["policy_guardrail"]`

---

### 2.4 should_run_guardrail 中的显式检查

```python
# litellm/integrations/custom_guardrail.py:406-476 (精简)

def should_run_guardrail(self, data, event_type: GuardrailEventHooks) -> bool:
    
    requested_guardrails = self.get_guardrail_from_metadata(data)
    
    # ========== default_on = True 的情况 ==========
    if self.default_on is True:
        # 检查 opt-out 和全局关闭
        if self.guardrail_name in opted_out_global_guardrails:
            return False
        if disable_global_guardrail is not True:
            # 检查 event_hook 匹配
            if self._event_hook_is_event_type(event_type):
                return True
        return False
    
    # ========== default_on = False 的情况 ==========
    # 必须显式请求才执行
    
    # 检查条件 1: 是否配置了 event_hook
    # 如果配置了 event_hook，但不在显式请求列表中 → 不执行
    if (
        self.event_hook
        and not self._guardrail_is_in_requested_guardrails(requested_guardrails)
        and event_type.value != "logging_only"
    ):
        return False  # ← 显式检查是否在请求列表中
    
    # 检查条件 2: event_hook 是否匹配
    if not self._event_hook_is_event_type(event_type):
        return False
    
    return True
```

### 2.5 _guardrail_is_in_requested_guardrail 实现

```python
# litellm/integrations/custom_guardrail.py:310-322

def _guardrail_is_in_requested_guardrails(
    self,
    requested_guardrails: Union[List[str], List[Dict[str, DynamicGuardrailParams]]],
) -> bool:
    """
    检查当前 guardrail 是否在显式请求的列表中。
    
    支持两种格式:
    1. 字符串列表: ["guardrail_name_1", "guardrail_name_2"]
    2. 对象列表: [{"guardrail_name": {"extra_param": "value"}}]
    """
    
    for _guardrail in requested_guardrails:
        # 格式 2: 对象列表
        if isinstance(_guardrail, dict):
            if self.guardrail_name in _guardrail:
                return True
        
        # 格式 1: 字符串列表
        elif isinstance(_guardrail, str):
            if self.guardrail_name == _guardrail:
                return True

    return False
```

---

## 3. Pipeline 执行顺序：Pipeline 优先，独立 Guardrails 后行

### 3.1 关键发现：Pipeline 管理的会被跳过

这是**之前理解的重要偏差**。实际代码行为是：

```
Phase A: Pipeline 执行 (按 steps 定义的顺序)
    ↓
将 _pipeline_managed_guardrails 写入 metadata
    ↓
Phase B: 独立 Guardrails 遍历 (按 litellm.callbacks 顺序)
    ↓
检查: guardrail_name in _pipeline_managed_guardrails ?
    │
    ├── 是 → continue (跳过)
    └── 否 → 执行
```

### 3.2 代码路径追踪

#### 步骤 1: 解析 Pipeline 时记录 managed_guardrails

```python
# litellm/proxy/litellm_pre_call_utils.py:2219-2232

# Add resolved guardrails to request metadata
if metadata_variable_name not in data:
    data[metadata_variable_name] = {}

# ========== 关键: 追踪 pipeline 管理的 guardrails ==========
pipeline_managed_guardrails: set = set()

if pipelines:
    # 从所有 pipeline steps 中提取 guardrail 名称
    pipeline_managed_guardrails = PolicyResolver.get_pipeline_managed_guardrails(
        pipelines
    )
    
    # 写入 metadata，供后续跳过检查使用
    data[metadata_variable_name]["_guardrail_pipelines"] = pipelines
    data[metadata_variable_name]["_pipeline_managed_guardrails"] = pipeline_managed_guardrails
    
    verbose_proxy_logger.debug(
        f"Policy engine: resolved {len(pipelines)} pipeline(s), "
        f"managed guardrails: {pipeline_managed_guardrails}"
    )
```

#### 步骤 2: PolicyResolver.get_pipeline_managed_guardrails 实现

```python
# litellm/proxy/policy_engine/policy_resolver.py:251-273

@staticmethod
def get_pipeline_managed_guardrails(
    pipelines: List[Tuple[str, GuardrailPipeline]],
) -> Set[str]:
    """
    从所有 pipeline steps 中提取 guardrail 名称集合。
    
    这些 guardrails 将在后续的独立遍历阶段被跳过，避免重复执行。
    """
    managed: Set[str] = set()
    
    for policy_name, pipeline in pipelines:
        if pipeline and pipeline.steps:
            for step in pipeline.steps:
                # 从每个 step 提取 guardrail 名称
                if step.guardrail:
                    managed.add(step.guardrail)
    
    return managed
```

#### 步骤 3: 独立遍历阶段跳过

```python
# litellm/proxy/utils.py:1381-1413

async def pre_call_hook(self, ...):
    
    # ========== Phase A: 先执行 Pipeline ==========
    # Execute guardrail pipelines before the normal callback loop
    data = await self._maybe_execute_pipelines(
        data=data,
        user_api_key_dict=user_api_key_dict,
        call_type=call_type,
        event_hook="pre_call",
    )

    # ========== 获取 pipeline 管理的 guardrails ==========
    # Get pipeline-managed guardrails to skip in normal loop
    metadata = data.get("metadata", data.get("litellm_metadata", {})) or {}
    pipeline_managed: set = metadata.get("_pipeline_managed_guardrails", set())

    # ========== Phase B: 遍历独立 Guardrails ==========
    for callback in litellm.callbacks:
        # ... 获取 _callback ...
        
        if (
            _callback is not None
            and isinstance(_callback, CustomGuardrail)
            and data is not None
        ):
            # ========== 关键: 跳过 pipeline 管理的 guardrails ==========
            # Skip guardrails managed by a pipeline
            if (
                _callback.guardrail_name
                and _callback.guardrail_name in pipeline_managed
            ):
                continue  # ← 跳过！不执行
            
            # ========== 执行独立 guardrail ==========
            result = await self._process_guardrail_callback(
                callback=_callback,
                data=data,
                user_api_key_dict=user_api_key_dict,
                call_type=call_type,
                event_type=GuardrailEventHooks.pre_call,
            )
```

### 3.3 执行顺序图解

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    Pipeline 优先执行，独立 Guardrails 后行                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  假设配置:                                                                     │
│  ────────────────────────────────────────────────────────────────────────── │
│  litellm.callbacks = [G1, G2, G3, G4, G5, G6]  (注册顺序)                  │
│                                                                              │
│  policies:                                                                    │
│    - policy_name: "my_policy"                                                │
│      pipeline:                                                                │
│        steps:                                                                 │
│          - guardrail: "G3"  ←── 包含在 pipeline 中                         │
│            on_pass: "next"                                                   │
│          - guardrail: "G5"  ←── 包含在 pipeline 中                         │
│            on_pass: "allow"                                                  │
│                                                                              │
│  所以:                                                                        │
│  pipeline_managed_guardrails = {"G3", "G5"}                                │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  实际执行顺序:                                                                 │
│  ────────────────────────────────────────────────────────────────────────── │
│                                                                              │
│  [Phase A: Pipeline 执行]                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  1. 执行 G3 (pipeline step 1)                                         │  │
│  │     → 根据 on_pass 决定继续或终止                                       │  │
│  │                                                                       │  │
│  │  2. 执行 G5 (pipeline step 2)                                         │  │
│  │     → on_pass="allow" → 终止 pipeline                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│                              ↓                                               │
│                                                                              │
│  [Phase B: 独立 Guardrails 遍历]                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  遍历 litellm.callbacks = [G1, G2, G3, G4, G5, G6]                  │  │
│  │                                                                       │  │
│  │  G1: 不在 pipeline_managed → 执行 ✅                                   │  │
│  │  G2: 不在 pipeline_managed → 执行 ✅                                   │  │
│  │  G3: 在 pipeline_managed → 跳过 ❌ (continue)                          │  │
│  │  G4: 不在 pipeline_managed → 执行 ✅                                   │  │
│  │  G5: 在 pipeline_managed → 跳过 ❌ (continue)                          │  │
│  │  G6: 不在 pipeline_managed → 执行 ✅                                   │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  最终执行顺序 (实际):                                                          │
│  G3 (Pipeline) → G5 (Pipeline) → G1 → G2 → G4 → G6                         │
│                                                                              │
│  ⚠️  关键点:                                                                  │
│  1. Pipeline 中的 steps 优先执行                                              │
│  2. Pipeline 中的 guardrails 在独立遍历阶段被跳过                             │
│  3. 这确保了:                                                                 │
│     - Pipeline 可以控制执行顺序和终止条件                                      │
│     - 避免重复执行                                                           │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.4 安全移除用户注入的 _pipeline_managed_guardrails

```python
# litellm/proxy/litellm_pre_call_utils.py:1144-1149

for _meta_key in ("metadata", "litellm_metadata"):
    _user_meta = data.get(_meta_key)
    if isinstance(_user_meta, dict):
        # ========== 安全措施: 移除用户注入的 _pipeline_managed_guardrails ==========
        # 防止攻击者通过设置请求体中的 metadata 来影响 guardrail 执行
        _user_meta.pop("_pipeline_managed_guardrails", None)
        
        # 同时移除其他敏感字段
        for _k in [k for k in _user_meta if k.startswith("user_api_key_")]:
            _user_meta.pop(_k, None)
```

---

## 4. MCP 场景触发分支

### 4.1 GuardrailEventHooks 完整定义

```python
# litellm/types/guardrails.py:835-843

class GuardrailEventHooks(str, Enum):
    """
    Guardrail 事件钩子类型。
    
    用于指定 guardrail 在哪个阶段执行。
    """
    
    # ========== 标准 LLM 调用事件 ==========
    pre_call = "pre_call"           # LLM 调用前
    post_call = "post_call"         # LLM 响应后
    during_call = "during_call"     # 流式响应中 (并行执行)
    
    # 特殊事件
    logging_only = "logging_only"   # 仅用于日志记录，不执行实际检查
    
    # ========== MCP 工具调用事件 ==========
    pre_mcp_call = "pre_mcp_call"       # MCP 工具调用前
    during_mcp_call = "during_mcp_call"  # MCP 工具调用中
    
    # ========== Realtime 事件 ==========
    realtime_input_transcription = "realtime_input_transcription"
```

### 4.2 MCP 事件转换逻辑

当 `call_type == CallTypes.call_mcp_tool.value` 时，**事件类型会被转换**。

#### 转换点 1: pre_call → pre_mcp_call

```python
# litellm/proxy/utils.py:1051-1055

async def _process_guardrail_callback(
    self,
    callback: Any,
    data: dict,
    user_api_key_dict: Optional[UserAPIKeyAuth],
    call_type: CallTypesLiteral,
    event_type: GuardrailEventHooks,
):
    # ...
    
    # ========== MCP 事件转换 ==========
    # 如果是 MCP 工具调用，将 pre_call 转换为 pre_mcp_call
    if (
        event_type is GuardrailEventHooks.pre_call
        and call_type == CallTypes.call_mcp_tool.value
    ):
        event_type = GuardrailEventHooks.pre_mcp_call  # ← 转换！

    # 然后检查 should_run_guardrail
    if callback.should_run_guardrail(data=data, event_type=event_type) is not True:
        return None
```

#### 转换点 2: during_call → during_mcp_call

```python
# litellm/proxy/utils.py:1530-1535

async def during_call_hook(self, data: dict, ...):
    # ...
    
    for callback in litellm.callbacks:
        # ...
        
        event_type = GuardrailEventHooks.during_call
        
        # ========== MCP 事件转换 ==========
        # 如果是 MCP 工具调用，将 during_call 转换为 during_mcp_call
        if call_type == CallTypes.call_mcp_tool.value:
            event_type = GuardrailEventHooks.during_mcp_call  # ← 转换！

        # 然后检查 should_run_guardrail
        if (
            callback.should_run_guardrail(data=data, event_type=event_type)
            is not True
        ):
            continue
```

### 4.3 MCP 事件转换图解

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    MCP 场景事件转换逻辑                                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  事件转换规则:                                                                 │
│  ────────────────────────────────────────────────────────────────────────── │
│                                                                              │
│  当 call_type == "call_mcp_tool" 时:                                         │
│                                                                              │
│  ┌─────────────────┐      ┌──────────────────┐                             │
│  │   pre_call      │  →   │   pre_mcp_call   │                             │
│  │  (事件输入)     │      │   (实际检查)     │                             │
│  └─────────────────┘      └──────────────────┘                             │
│                                                                              │
│  ┌─────────────────┐      ┌──────────────────┐                             │
│  │  during_call    │  →   │ during_mcp_call  │                             │
│  │  (事件输入)     │      │   (实际检查)     │                             │
│  └─────────────────┘      └──────────────────┘                             │
│                                                                              │
│  转换发生在 should_run_guardrail 检查之前                                    │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Guardrail 配置示例:                                                           │
│  ────────────────────────────────────────────────────────────────────────── │
│                                                                              │
│  # 场景 1: 仅在标准 LLM 调用时执行                                            │
│  guardrails:                                                                 │
│    - guardrail_name: "llm_content_check"                                     │
│      litellm_params:                                                         │
│        guardrail: "azure"                                                    │
│        mode: "pre_call"            ←── 仅匹配 pre_call                      │
│        default_on: true                                                      │
│                                                                              │
│  结果:                                                                        │
│  - 标准 LLM 调用: pre_call 匹配 → 执行 ✅                                    │
│  - MCP 调用: pre_mcp_call 不匹配 pre_call → 不执行 ❌                       │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────── │
│                                                                              │
│  # 场景 2: 仅在 MCP 工具调用时执行                                            │
│  guardrails:                                                                 │
│    - guardrail_name: "mcp_jwt_signer"                                       │
│      litellm_params:                                                         │
│        guardrail: "mcp_jwt_signer"                                          │
│        mode: "pre_mcp_call"        ←── 仅匹配 pre_mcp_call                  │
│        default_on: true                                                      │
│                                                                              │
│  结果:                                                                        │
│  - 标准 LLM 调用: pre_call 不匹配 pre_mcp_call → 不执行 ❌                  │
│  - MCP 调用: pre_mcp_call 匹配 → 执行 ✅                                     │
│                                                                              │
│  ────────────────────────────────────────────────────────────────────────── │
│                                                                              │
│  # 场景 3: 同时支持标准和 MCP 调用                                             │
│  guardrails:                                                                 │
│    - guardrail_name: "universal_guardrail"                                  │
│      litellm_params:                                                         │
│        guardrail: "bedrock_guardrails"                                      │
│        mode: ["pre_call", "pre_mcp_call"]  ←── 支持两种                     │
│        default_on: true                                                      │
│                                                                              │
│  结果:                                                                        │
│  - 标准 LLM 调用: pre_call 匹配 → 执行 ✅                                    │
│  - MCP 调用: pre_mcp_call 匹配 → 执行 ✅                                     │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.4 支持 MCP 事件的 Guardrails

从代码中看到，以下 guardrails 明确支持 MCP 事件：

| Guardrail | 支持的 MCP 事件 | 代码位置 |
|-----------|-----------------|----------|
| **bedrock_guardrails** | `pre_mcp_call`, `during_mcp_call` | `proxy/guardrails/guardrail_hooks/bedrock_guardrails.py:141-142` |
| **custom_code** | `pre_mcp_call`, `during_mcp_call` | `proxy/guardrails/guardrail_hooks/custom_code/custom_code_guardrail.py:127-128` |
| **oma** | `pre_mcp_call`, `during_mcp_call` | `proxy/guardrails/guardrail_hooks/noma/noma_v2.py:89-90` |
| **panw_prisma_airs** | `pre_mcp_call`, `during_mcp_call` | `proxy/guardrails/guardrail_hooks/panw_prisma_airs/panw_prisma_airs.py:102-103` |
| **mcp_jwt_signer** | 仅 `pre_mcp_call` (强制) | `proxy/guardrails/guardrail_hooks/mcp_jwt_signer/__init__.py:23` |

### 4.5 MCP JWT Signer 强制使用 pre_mcp_call

```python
# litellm/proxy/guardrails/guardrail_hooks/mcp_jwt_signer/__init__.py:20-27

def initialize_guardrail(...):
    # ...
    
    mode = litellm_params.mode
    
    # ========== 强制检查: 必须使用 pre_mcp_call ==========
    if mode != "pre_mcp_call":
        raise ValueError(
            f"MCPJWTSigner guardrail '{guardrail_name}' has mode='{mode}' but must use "
            "mode='pre_mcp_call'. JWT injection only fires for MCP tool calls."
        )
```

---

## 5. 完整执行流程图解

### 5.1 Pre-Call 阶段完整流程

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    pre_call_hook 完整执行流程 (修正版)                         │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  入口: proxy_logging_obj.pre_call_hook()                                     │
│                                                                              │
│                              │                                               │
│                              ▼                                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Step 1: 执行 Policy Engine Pipelines                                 │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  data = await self._maybe_execute_pipelines(                         │  │
│  │      data=data,                                                       │  │
│  │      user_api_key_dict=user_api_key_dict,                            │  │
│  │      call_type=call_type,                                             │  │
│  │      event_hook="pre_call",  ←── 注意: 输入是 pre_call               │  │
│  │  )                                                                     │  │
│  │                                                                       │  │
│  │  内部:                                                                │  │
│  │  ├── 按 pipeline.steps 顺序执行                                       │  │
│  │  ├── 每个 step 可配置 allow/block/next                                │  │
│  │  └── block 会立即终止并返回错误                                        │  │
│  │                                                                       │  │
│  │  副作用:                                                              │  │
│  │  └── 从 steps 提取 guardrail 名称 → pipeline_managed_guardrails      │  │
│  │      写入 data["metadata"]["_pipeline_managed_guardrails"]           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│                              │                                               │
│                              ▼                                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Step 2: 获取 pipeline 管理的 guardrails (用于跳过)                    │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  metadata = data.get("metadata", data.get("litellm_metadata", {}))  │  │
│  │  pipeline_managed: set = metadata.get(                                │  │
│  │      "_pipeline_managed_guardrails", set()                            │  │
│  │  )                                                                     │  │
│  │                                                                       │  │
│  │  示例: pipeline_managed = {"G3", "G5"}                               │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│                              │                                               │
│                              ▼                                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Step 3: 遍历 litellm.callbacks (独立 Guardrails)                    │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  for callback in litellm.callbacks:                                   │  │
│  │      │                                                                │  │
│  │      ▼                                                                │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  Check 1: 是 CustomGuardrail 实例?                             │   │  │
│  │  │  if not isinstance(_callback, CustomGuardrail):               │   │  │
│  │  │      → 跳过 (是普通的 CustomLogger)                             │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │      │                                                                │  │
│  │      ▼ (是 CustomGuardrail)                                          │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  Check 2: 是否被 Pipeline 管理?                                 │   │  │
│  │  │  if _callback.guardrail_name in pipeline_managed:             │   │  │
│  │  │      → continue (跳过! 避免重复执行)                            │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │      │                                                                │  │
│  │      ▼ (不在 pipeline_managed 中)                                    │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  Step 3a: 事件类型转换 (MCP 场景)                               │   │  │
│  │  │                                                                 │   │  │
│  │  │  if event_type is GuardrailEventHooks.pre_call                │   │  │
│  │  │     and call_type == CallTypes.call_mcp_tool.value:           │   │  │
│  │  │      event_type = GuardrailEventHooks.pre_mcp_call  ← 转换!   │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │      │                                                                │  │
│  │      ▼                                                                │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  Step 3b: should_run_guardrail() 检查                          │   │  │
│  │  │                                                                 │   │  │
│  │  │  检查维度:                                                      │   │  │
│  │  │  1. default_on = True?                                         │   │  │
│  │  │     ├── 在 opted_out_global_guardrails 中? → 不执行            │   │  │
│  │  │     ├── disable_global_guardrails = True? → 不执行             │   │  │
│  │  │     └── event_hook 匹配? → 执行                                │   │  │
│  │  │                                                                 │   │  │
│  │  │  2. default_on = False?                                        │   │  │
│  │  │     ├── 显式请求入口检查:                                       │   │  │
│  │  │     │   ├── 优先级 1: data["guardrails"] (根级)               │   │  │
│  │  │     │   ├── 优先级 2: data["metadata"]["guardrails"]          │   │  │
│  │  │     │   └── 优先级 3: data["litellm_metadata"]["guardrails"]  │   │  │
│  │  │     └── event_hook 匹配? → 执行                                │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │      │                                                                │  │
│  │      ▼ (should_run_guardrail 返回 True)                             │  │
│  │  ┌──────────────────────────────────────────────────────────────┐   │  │
│  │  │  Step 3c: 执行 Guardrail                                        │   │  │
│  │  │                                                                 │   │  │
│  │  │  result = await self._process_guardrail_callback(             │   │  │
│  │  │      callback=_callback,                                       │   │  │
│  │  │      data=data,                                                │   │  │
│  │  │      user_api_key_dict=user_api_key_dict,                     │   │  │
│  │  │      call_type=call_type,                                      │   │  │
│  │  │      event_type=event_type,  ← 可能是 pre_call 或 pre_mcp_call│   │  │
│  │  │  )                                                              │   │  │
│  │  │                                                                 │   │  │
│  │  │  Guardrail 可以:                                                │   │  │
│  │  │  ├── 返回修改后的 data                                          │   │  │
│  │  │  └── 抛出 HTTPException 阻止请求                                │   │  │
│  │  └──────────────────────────────────────────────────────────────┘   │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│                              │                                               │
│                              ▼                                               │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  Step 4: 处理 Guardrail 元数据                                        │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │  self._process_guardrail_metadata(data)                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│                              │                                               │
│                              ▼                                               │
│                                                                              │
│  出口: return data (可能被 guardrails 修改)                                  │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 完整请求生命周期时间线 (修正版)

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    LiteLLM Proxy 请求生命周期 (修正版)                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  HTTP 请求到达                                                                 │
│      │                                                                       │
│      ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  ASGI 中间件层 (洋葱模型)                                               │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │  请求进入时的执行顺序:                                                 │  │
│  │  1. InFlightRequestsMiddleware (_in_flight += 1)                     │  │
│  │  2. PrometheusAuthMiddleware (仅 /metrics)                            │  │
│  │  3. CORSMiddleware                                                    │  │
│  │                                                                       │  │
│  │  响应返回时的执行顺序:                                                 │  │
│  │  1. CORSMiddleware                                                    │  │
│  │  2. PrometheusAuthMiddleware                                          │  │
│  │  3. InFlightRequestsMiddleware (_in_flight -= 1, finally 块)        │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│      │                                                                       │
│      ▼                                                                       │
│  路由匹配 + 认证检查 (user_api_key_auth)                                     │
│      │                                                                       │
│      ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  common_processing_pre_call_logic()                                    │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │  ├── add_litellm_data_to_request()                                    │  │
│  │  ├── litellm.utils.function_setup() (创建 logging_obj)                 │  │
│  │  └── pre_call_hook()  ←── 详细流程见上图                              │  │
│  │      ├── Phase A: Pipeline 优先执行                                    │  │
│  │      │   └── pipeline_managed_guardrails 被记录                        │  │
│  │      └── Phase B: 独立 Guardrails                                      │  │
│          └── pipeline_managed_guardrails 被跳过                           │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│      │                                                                       │
│      ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  during_call_hook 与 LLM 调用并行                                      │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  asyncio.create_task(during_call_hook())  ←── 提前启动 (后台任务)     │  │
│  │      │                                                                │  │
│  │      ├──▶ during_call_hook 内部:                                     │  │
│  │      │   ├── 遍历 litellm.callbacks                                   │  │
│  │      │   ├── 事件类型转换:                                            │  │
│  │      │   │   if call_type == "call_mcp_tool":                        │  │
│  │      │   │       event_type = GuardrailEventHooks.during_mcp_call     │  │
│  │      │   ├── should_run_guardrail() 检查                              │  │
│  │      │   └── asyncio.gather(*guardrail_tasks)  ←── 并行执行           │  │
│  │      │                                                                │  │
│  │      │                                                                │  │
│  │  await route_request()  ←── LLM 实际调用 (与 during_call 并行)        │  │
│  │      │                                                                │  │
│  │      ▼                                                                │  │
│  │  await asyncio.gather(*tasks)  ←── 等待两者都完成                     │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│      │                                                                       │
│      ▼                                                                       │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │  post_call_success_hook                                                │  │
│  ├──────────────────────────────────────────────────────────────────────┤  │
│  │                                                                       │  │
│  │  ⚠️  关键差异: 流式 vs 非流式                                           │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────┐    ┌─────────────────────────────────┐  │  │
│  │  │    非流式请求           │    │         流式请求                  │  │  │
│  │  ├─────────────────────────┤    ├─────────────────────────────────┤  │  │
│  │  │                         │    │                                 │  │  │
│  │  │  post_call_success_hook │    │  _on_deferred_stream_complete  │  │  │
│  │  │  立即执行               │    │  延迟到流结束后才执行           │  │  │
│  │  │                         │    │                                 │  │  │
│  │  │  ✅ 响应尚未发送给客户端  │    │  ❌ 所有 chunk 已发送给客户端  │  │  │
│  │  │  ✅ 可以拦截/修改响应    │    │  ❌ 无法阻止已发送内容         │  │  │
│  │  │                         │    │  ⚠️  仅用于审计和日志记录       │  │  │
│  │  └─────────────────────────┘    └─────────────────────────────────┘  │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│      │                                                                       │
│      ▼                                                                       │
│  响应返回客户端                                                               │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键代码位置索引

### 6.1 显式触发入口

| 功能 | 文件 | 行号 |
|------|------|------|
| `get_guardrail_from_metadata` (三级优先级) | `litellm/integrations/custom_guardrail.py` | 289-308 |
| `_guardrail_is_in_requested_guardrails` | `litellm/integrations/custom_guardrail.py` | 310-322 |

### 6.2 Pipeline 跳过机制

| 功能 | 文件 | 行号 |
|------|------|------|
| 记录 `_pipeline_managed_guardrails` | `litellm/proxy/litellm_pre_call_utils.py` | 2219-2232 |
| `get_pipeline_managed_guardrails` | `litellm/proxy/policy_engine/policy_resolver.py` | 251-273 |
| 独立遍历时跳过 | `litellm/proxy/utils.py` | 1390-1413 |
| 安全移除用户注入的字段 | `litellm/proxy/litellm_pre_call_utils.py` | 1144-1149 |

### 6.3 MCP 事件转换

| 功能 | 文件 | 行号 |
|------|------|------|
| `GuardrailEventHooks` 完整定义 | `litellm/types/guardrails.py` | 835-843 |
| `pre_call` → `pre_mcp_call` 转换 | `litellm/proxy/utils.py` | 1051-1055 |
| `during_call` → `during_mcp_call` 转换 | `litellm/proxy/utils.py` | 1530-1535 |
| MCP JWT Signer 强制检查 | `litellm/proxy/guardrails/guardrail_hooks/mcp_jwt_signer/__init__.py` | 23-27 |

### 6.4 中间件注册

| 功能 | 文件 | 行号 |
|------|------|------|
| 中间件注册顺序 | `litellm/proxy/proxy_server.py` | 1537-1547 |
| `InFlightRequestsMiddleware` | `litellm/proxy/middleware/in_flight_requests_middleware.py` | 29-51 |

---

## 总结

### 核心纠偏点回顾

| 纠偏点 | 实际代码行为 |
|--------|--------------|
| **显式触发入口** | 三级优先级：`data["guardrails"]` (根级) > `data["metadata"]["guardrails"]` > `data["litellm_metadata"]["guardrails"]` |
| **Pipeline 执行顺序** | **Phase A: Pipeline 优先执行** → 记录 `_pipeline_managed_guardrails` → **Phase B: 独立 Guardrails 执行**，但跳过 `_pipeline_managed_guardrails` 中的 |
| **MCP 事件转换** | 当 `call_type=call_mcp_tool` 时，`pre_call` → `pre_mcp_call`，`during_call` → `during_mcp_call`，转换发生在 `should_run_guardrail` 检查之前 |

### 执行顺序决策树 (修正版)

```
请求进入
    │
    ├── [ASGI 中间件] InFlightRequests → PrometheusAuth → CORS
    │
    ├── [pre_call_hook]
    │       ├── Phase A: Pipeline 优先 (按 steps 顺序)
    │       │       └── 记录 pipeline_managed_guardrails
    │       └── Phase B: 独立 Guardrails (按 litellm.callbacks 顺序)
    │               └── 跳过 pipeline_managed_guardrails 中的
    │
    ├── [during_call_hook] 与 LLM 调用并行
    │       ├── MCP 场景: during_call → during_mcp_call
    │       └── asyncio.gather 并行执行
    │
    └── [post_call]
            ├── 非流式: 立即执行，可拦截响应
            └── 流式: 延迟到流结束，仅用于审计
```

---

*修正版分析完成。本报告基于实际代码路径追踪，纠正了显式触发入口、Pipeline 跳过机制和 MCP 事件转换的理解偏差。*
