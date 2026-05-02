# LiteLLM Proxy Server 中间件链与 Guardrail 注册机制分析

> 分析日期：2026-05-02
> 代码版本：基于当前仓库版本

---

## 目录

1. [概述](#1-概述)
2. [中间件链架构](#2-中间件链架构)
3. [Guardrail 注册机制](#3-guardrail-注册机制)
4. [Guardrail 执行流程](#4-guardrail-执行流程)
5. [多个守护规则的执行顺序](#5-多个守护规则的执行顺序)
6. [关键代码文件索引](#6-关键代码文件索引)
7. [附录：配置示例](#7-附录配置示例)

---

## 1. 概述

LiteLLM Proxy Server 采用**多层级请求处理架构**，结合了：

- **ASGI 中间件层**：处理跨域、认证、请求计数等通用能力
- **Callback Manager 机制**：统一管理 guardrails 和其他回调
- **Policy Engine**：支持基于策略的 guardrail 组合与继承
- **Pipeline Executor**：支持有序的多步骤 guardrail 执行

这种架构允许：
- 静态配置和动态发现的 guardrails 并存
- 基于请求上下文（团队、API Key、模型）的条件执行
- 可组合的执行顺序（Pipeline 模式 vs 独立执行模式）

---

## 2. 中间件链架构

### 2.1 ASGI 中间件注册顺序

在 `proxy_server.py` 中，中间件按以下顺序注册：

```python
# 注册顺序（代码位置：proxy_server.py:1537-1547）
app.add_middleware(CORSMiddleware, ...)           # 第1层：跨域处理
app.add_middleware(PrometheusAuthMiddleware)      # 第2层：Prometheus 认证
app.add_middleware(InFlightRequestsMiddleware)    # 第3层：飞行请求计数
```

### 2.2 各中间件职责

| 中间件 | 文件位置 | 职责说明 |
|--------|----------|----------|
| **CORSMiddleware** | FastAPI 内置 | 处理 CORS 跨域请求，配置 `allow_origins`、`allow_headers`、`allow_methods`、`expose_headers` |
| **PrometheusAuthMiddleware** | `proxy/middleware/prometheus_auth_middleware.py` | 保护 `/metrics` 端点，验证请求是否携带正确的 `PROMETHEUS_AUTH_TOKEN` |
| **InFlightRequestsMiddleware** | `proxy/middleware/in_flight_requests_middleware.py` | 统计当前正在处理的 HTTP 请求数量，暴露为 Prometheus gauge `litellm_in_flight_requests`，供 `/health/backlog` 端点使用 |

### 2.3 中间件执行流程

```
请求进入
    ↓
[CORSMiddleware]
  - 检查 Origin
  - 添加 CORS 响应头
  - 处理 OPTIONS 预检请求
    ↓
[PrometheusAuthMiddleware]
  - 仅对 /metrics 路径生效
  - 验证 Authorization header 或 query 参数中的 token
    ↓
[InFlightRequestsMiddleware]
  - 计数器 +1
  - Prometheus gauge inc()
    ↓
  路由处理（业务逻辑）
    ↓
[InFlightRequestsMiddleware finally]
  - 计数器 -1
  - Prometheus gauge dec()
    ↓
响应返回
```

### 2.4 InFlightRequestsMiddleware 实现细节

```python
# 核心实现（in_flight_requests_middleware.py:29-51）
class InFlightRequestsMiddleware:
    _in_flight: int = 0                              # 类级别的计数器
    _gauge: Optional[Any] = None                     # Prometheus Gauge
    _gauge_init_attempted: bool = False              # 延迟初始化标记

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        InFlightRequestsMiddleware._in_flight += 1
        gauge = InFlightRequestsMiddleware._get_gauge()
        if gauge is not None:
            gauge.inc()
        try:
            await self.app(scope, receive, send)
        finally:
            InFlightRequestsMiddleware._in_flight -= 1
            if gauge is not None:
                gauge.dec()
```

**关键点**：
- 计数器是**类级别**的，同一 worker 进程内共享
- 支持 Prometheus 多进程模式（`multiprocess_mode="livesum"`）
- 仅对 `scope["type"] == "http"` 的请求计数

---

## 3. Guardrail 注册机制

Guardrails 采用**双层注册机制**：

1. **初始化器注册**：定义如何创建 guardrail 实例
2. **回调注册**：将实例加入执行队列

### 3.1 Guardrail 类型系统

```python
# 定义位置：types/guardrails.py
class GuardrailEventHooks(StrEnum):
    pre_call = "pre_call"           # 请求前执行
    post_call = "post_call"         # 响应后执行
    during_call = "during_call"     # 流式响应中执行
    pre_mcp_call = "pre_mcp_call"   # MCP 工具调用前
    during_mcp_call = "during_mcp_call"  # MCP 工具调用中
```

### 3.2 初始化器注册表

#### 3.2.1 静态注册

```python
# guardrail_registry.py:41-50
guardrail_initializer_registry = {
    SupportedGuardrailIntegrations.BEDROCK.value: initialize_bedrock,
    SupportedGuardrailIntegrations.LAKERA.value: initialize_lakera,
    SupportedGuardrailIntegrations.LAKERA_V2.value: initialize_lakera_v2,
    SupportedGuardrailIntegrations.PRESIDIO.value: initialize_presidio,
    SupportedGuardrailIntegrations.HIDE_SECRETS.value: initialize_hide_secrets,
    SupportedGuardrailIntegrations.TOOL_PERMISSION.value: initialize_tool_permission,
    SupportedGuardrailIntegrations.GRAYSWAN.value: initialize_grayswan,
    SupportedGuardrailIntegrations.LLM_AS_A_JUDGE.value: initialize_llm_as_a_judge,
}
```

#### 3.2.2 动态发现

```python
# guardrail_registry.py:57-130
def get_guardrail_initializer_from_hooks():
    """
    自动扫描 guardrail_hooks 目录下的子目录
    发现包含 guardrail_initializer_registry 或 initialize_guardrail 的模块
    """
    discovered_initializers = {}
    hooks_dir = os.path.join(current_dir, "guardrail_hooks")
    
    for item in os.listdir(hooks_dir):
        item_path = os.path.join(hooks_dir, item)
        if not os.path.isdir(item_path) or item.startswith("__"):
            continue
        
        init_file = os.path.join(item_path, "__init__.py")
        if not os.path.exists(init_file):
            continue
        
        module_path = f"litellm.proxy.guardrails.guardrail_hooks.{item}"
        module = importlib.import_module(module_path)
        
        # 优先查找 guardrail_initializer_registry 字典
        if hasattr(module, "guardrail_initializer_registry"):
            discovered_initializers.update(module.guardrail_initializer_registry)
        # 回退到 initialize_guardrail 函数
        elif hasattr(module, "initialize_guardrail"):
            discovered_initializers[item] = module.initialize_guardrail
    
    return discovered_initializers
```

#### 3.2.3 已注册的 Guardrails

| Guardrail 类型 | 目录位置 | 支持的事件钩子 |
|----------------|----------|----------------|
| **aim** | `guardrail_hooks/aim/` | pre_call, post_call |
| **akto** | `guardrail_hooks/akto/` | pre_call, post_call |
| **aporia_ai** | `guardrail_hooks/aporia_ai/` | post_call, during_call |
| **azure** | `guardrail_hooks/azure/` | pre_call, during_call (prompt_shield, text_moderation) |
| **bedrock_guardrails** | `guardrail_hooks/bedrock_guardrails.py` | pre_call, post_call, during_call, pre_mcp_call, during_mcp_call |
| **block_code_execution** | `guardrail_hooks/block_code_execution/` | 可配置 |
| **crowdstrike_aidr** | `guardrail_hooks/crowdstrike_aidr/` | pre_call, post_call |
| **custom_code** | `guardrail_hooks/custom_code/` | 可配置 |
| **dynamoai** | `guardrail_hooks/dynamoai/` | 可配置 |
| **enkryptai** | `guardrail_hooks/enkryptai/` | 可配置 |
| **generic_guardrail_api** | `guardrail_hooks/generic_guardrail_api/` | 通用 API 接口 |
| **grayswan** | `guardrail_hooks/grayswan/` | 可配置 |
| **guardrails_ai** | `guardrail_hooks/guardrails_ai/` | 可配置 |
| **hiddenlayer** | `guardrail_hooks/hiddenlayer/` | 可配置 |
| **ibm_guardrails** | `guardrail_hooks/ibm_guardrails/` | 可配置 |
| **javelin** | `guardrail_hooks/javelin/` | 可配置 |
| **lakera_ai** | `guardrail_hooks/lakera_ai.py` | pre_call |
| **lasso** | `guardrail_hooks/lasso/` | 可配置 |
| **litellm_content_filter** | `guardrail_hooks/litellm_content_filter/` | 可配置 |
| **llm_as_a_judge** | `guardrail_hooks/llm_as_a_judge/` | 可配置 |
| **noma** | `guardrail_hooks/noma/` | 可配置 |
| **openai** | `guardrail_hooks/openai/` | moderations |
| **pangea** | `guardrail_hooks/pangea/` | 可配置 |
| **panw_prisma_airs** | `guardrail_hooks/panw_prisma_airs/` | 可配置 |
| **pillar** | `guardrail_hooks/pillar/` | 可配置 |
| **prompt_security** | `guardrail_hooks/prompt_security/` | 可配置 |
| **promptguard** | `guardrail_hooks/promptguard/` | 可配置 |
| **qualifire** | `guardrail_hooks/qualifire/` | 可配置 |
| **semantic_guard** | `guardrail_hooks/semantic_guard/` | 可配置 |
| **tool_policy** | `guardrail_hooks/tool_policy/` | 可配置 |
| **unified_guardrail** | `guardrail_hooks/unified_guardrail/` | 统一入口 |
| **xecguard** | `guardrail_hooks/xecguard/` | 可配置 |
| **zscaler_ai_guard** | `guardrail_hooks/zscaler_ai_guard/` | 可配置 |

### 3.3 初始化流程

```
配置解析 (config.yaml 中的 guardrails 节)
    ↓
init_guardrails_v2() 调用
    ↓
InMemoryGuardrailHandler.initialize_guardrail()
    ├── 1. 生成 guardrail_id (UUID)
    ├── 2. 解析 LitellmParams
    ├── 3. 查找初始化器 (guardrail_initializer_registry)
    ├── 4. 调用初始化器创建实例
    │       └── litellm.logging_callback_manager.add_litellm_callback()
    ├── 5. 存储到 IN_MEMORY_GUARDRAILS
    └── 6. 映射 guardrail_id -> CustomGuardrail
    ↓
_populate_router_guardrail_list()  (用于负载均衡)
```

### 3.4 回调管理器机制

Guardrails 通过 `logging_callback_manager` 统一管理：

```python
# 典型的 guardrail 初始化器示例 (aim/__init__.py)
def initialize_guardrail(litellm_params: LitellmParams, guardrail: Guardrail):
    _aim_callback = AIMGuardrail(
        api_key=litellm_params.api_key,
        guardrail_name=guardrail.get("guardrail_name", ""),
        event_hook=litellm_params.mode,      # pre_call / post_call
        default_on=litellm_params.default_on,
    )
    # 注册到回调管理器
    litellm.logging_callback_manager.add_litellm_callback(_aim_callback)
    return _aim_callback
```

**Callback Manager 关键属性**：

| 属性 | 类型 | 用途 |
|------|------|------|
| `litellm.callbacks` | `List[Any]` | 全局回调列表，按注册顺序执行 |
| `litellm.logging_callback_manager` | `CallbackManager` | 回调管理器，提供 `add_litellm_callback()`、`remove_callback_from_list_by_object()` 等方法 |

### 3.5 自定义 Guardrail

支持从 Python 文件或模块路径加载自定义 guardrail：

```python
# guardrail_registry.py:498-552
def initialize_custom_guardrail(self, guardrail, guardrail_type, litellm_params, config_file_path):
    """
    示例: guardrail_type = "my_module.MyCustomGuardrail"
    或: guardrail_type = "/path/to/file.py.MyCustomGuardrail"
    """
    # 1. 动态加载类
    _guardrail_class = get_instance_fn(guardrail_type, config_file_path)
    
    # 2. 实例化
    _guardrail_callback = _guardrail_class(
        guardrail_name=guardrail["guardrail_name"],
        event_hook=mode,  # 从 litellm_params.mode 获取
        default_on=default_on,
        **extra_params,
    )
    
    # 3. 注册
    litellm.logging_callback_manager.add_litellm_callback(_guardrail_callback)
```

---

## 4. Guardrail 执行流程

### 4.1 请求处理中的关键嵌入点

Guardrails 在请求处理的多个阶段嵌入：

```
HTTP 请求到达
    ↓
[ASGI 中间件链]
    ↓
路由匹配 (如 /chat/completions)
    ↓
认证检查 (user_api_key_auth)
    ↓
┌─────────────────────────────────────────────────────────┐
│  common_request_processing.py: ProxyBaseLLMRequestProcessing  │
├─────────────────────────────────────────────────────────┤
│  1. add_litellm_data_to_request()                        │
│     - 准备请求数据                                         │
├─────────────────────────────────────────────────────────┤
│  2. proxy_logging_obj.pre_call_hook()  ←─── Guardrails │
│     ├── 执行 Pipeline (policy engine)                    │
│     └── 执行独立 guardrails                              │
├─────────────────────────────────────────────────────────┤
│  3. route_request()                                      │
│     - 调用 LLM 提供商 API                                 │
│     └── during_call_hook() (流式响应)                   │
├─────────────────────────────────────────────────────────┤
│  4. post_call_success_hook()  ←─── Guardrails          │
│     - 响应后检查                                          │
└─────────────────────────────────────────────────────────┘
    ↓
响应返回
```

### 4.2 Pre-Call Hook 执行逻辑

```python
# utils.py:1339-1466 (精简版)
async def pre_call_hook(self, user_api_key_dict, data, call_type):
    
    # ========== 阶段 1: 执行 Pipeline ==========
    data = await self._maybe_execute_pipelines(
        data=data,
        user_api_key_dict=user_api_key_dict,
        call_type=call_type,
        event_hook="pre_call",
    )
    
    # 获取被 pipeline 管理的 guardrails，避免重复执行
    pipeline_managed = metadata.get("_pipeline_managed_guardrails", set())
    
    # ========== 阶段 2: 执行独立 Guardrails ==========
    for callback in litellm.callbacks:
        if isinstance(callback, CustomGuardrail):
            
            # 跳过已被 pipeline 管理的 guardrails
            if callback.guardrail_name in pipeline_managed:
                continue
            
            # 检查是否应该执行 (event_type, default_on, enabled_roles 等)
            if not callback.should_run_guardrail(data=data, event_type=GuardrailEventHooks.pre_call):
                continue
            
            # 执行 guardrail
            result = await self._process_guardrail_callback(
                callback=callback,
                data=data,
                user_api_key_dict=user_api_key_dict,
                call_type=call_type,
                event_type=GuardrailEventHooks.pre_call,
            )
            
            if result is not None:
                data = result  # guardrail 可能修改请求数据
```

### 4.3 should_run_guardrail 检查逻辑

Guardrail 是否执行由以下条件决定：

```python
# 检查维度 (CustomGuardrail 基类逻辑)
def should_run_guardrail(self, data: dict, event_type: GuardrailEventHooks) -> bool:
    
    # 1. 事件类型匹配
    if self.event_hook is not None:
        if not self._event_hook_is_event_type(event_type):
            return False
    
    # 2. default_on 检查
    if self.default_on is False:
        # 需要显式在 metadata.guardrails 中指定
        guardrails_in_metadata = data.get("metadata", {}).get("guardrails", [])
        if self.guardrail_name not in guardrails_in_metadata:
            return False
    
    # 3. enabled_roles 检查 (如果配置了)
    if self.enabled_roles:
        user_role = data.get("user_api_key_dict", {}).get("user_role")
        if user_role not in self.enabled_roles:
            return False
    
    return True
```

### 4.4 During-Call Hook (流式响应)

对于流式响应，guardrails 在每个 chunk 处理时并行执行：

```python
# utils.py:1502-1580
async def during_call_hook(self, data, user_api_key_dict, call_type):
    guardrail_tasks = []
    
    for callback in litellm.callbacks:
        if isinstance(callback, CustomGuardrail):
            # 检查应该运行
            if not callback.should_run_guardrail(data, event_type=GuardrailEventHooks.during_call):
                continue
            
            # 创建任务
            guardrail_task = self._run_guardrail_task_with_enrichment(
                callback,
                callback.async_moderation_hook(
                    data=data,
                    user_api_key_dict=user_api_key_dict,
                    call_type=call_type,
                ),
            )
            guardrail_tasks.append(guardrail_task)
    
    # 并行执行所有 guardrails
    if guardrail_tasks:
        await asyncio.gather(*guardrail_tasks)
```

**关键点**：during_call 是**并行执行**的，不保证顺序。

### 4.5 Post-Call Hook

响应成功后的执行逻辑：

```python
# common_request_processing.py:1506-1620 (post_call 处理)
# 非流式请求: 直接调用 async_post_call_success_hook
# 流式请求: 组装完成后通过 _run_deferred_stream_guardrails 执行

@staticmethod
async def _run_deferred_stream_guardrails(
    captured_data, captured_user_api_key_dict, 
    captured_logging_obj, assembled_response, cache_hit
):
    # 检查模型级别的 guardrails
    guardrail_data = _check_and_merge_model_level_guardrails(
        data=captured_data, llm_router=_global_llm_router
    )
    
    for cb in litellm.callbacks:
        if not isinstance(cb, CustomGuardrail):
            continue
        
        # 检查应该运行
        if not cb.should_run_guardrail(
            data=guardrail_data,
            event_type=GuardrailEventHooks.post_call,
        ):
            continue
        
        # 跳过已通过 apply_guardrail 执行的 (unified_guardrail)
        if "apply_guardrail" in type(cb).__dict__:
            continue
        
        # 执行
        guardrail_result = await cb.async_post_call_success_hook(
            user_api_key_dict=captured_user_api_key_dict,
            data=guardrail_data,
            response=_response,
        )
```

---

## 5. 多个守护规则的执行顺序

LiteLLM 提供**两种执行模式**，支持灵活的顺序控制：

| 模式 | 执行方式 | 顺序控制 | 适用场景 |
|------|----------|----------|----------|
| **独立执行** | 按 `litellm.callbacks` 列表顺序 | 注册顺序 + `should_run_guardrail` 过滤 | 简单场景，各 guardrail 独立 |
| **Pipeline 模式** | 按 pipeline.steps 定义的顺序 | 显式定义步骤顺序 + 条件动作 | 复杂场景，需要有向执行流 |

### 5.1 独立执行模式顺序

**执行顺序**：

1. **Pipeline 优先**：如果有 policy engine 解析出的 pipelines，先执行
2. **按注册顺序**：`litellm.callbacks` 列表中的顺序就是执行顺序
3. **过滤机制**：`should_run_guardrail()` 返回 False 的会被跳过

```
litellm.callbacks = [cb1, cb2, cb3, cb4, cb5]
                           ↓
                    顺序执行
                           ↓
                    should_run_guardrail 检查
                           ↓
                    [cb1 ✓ 执行] → [cb2 ✗ 跳过] → [cb3 ✓ 执行] → ...
```

### 5.2 Pipeline 模式详解

#### 5.2.1 Pipeline 配置结构

```python
# pipeline_executor.py:33-128
class PipelineExecutor:
    
    @staticmethod
    async def execute_steps(
        steps: List[PipelineStep],
        mode: str,        # pre_call / post_call
        data: dict,
        user_api_key_dict: Any,
        call_type: str,
        policy_name: str,
    ) -> PipelineExecutionResult:
        
        step_results = []
        working_data = data.copy()
        
        for i, step in enumerate(steps):
            # ========== 执行单个步骤 ==========
            outcome, modified_data, error_detail = await PipelineExecutor._run_step(
                step=step,
                mode=mode,
                data=working_data,
                user_api_key_dict=user_api_key_dict,
                call_type=call_type,
            )
            
            # ========== 确定动作 ==========
            action = _pipeline_action_for_outcome(step, outcome)
            # action 可选值: "allow", "block", "next", "modify_response"
            
            # ========== 处理结果 ==========
            # 1. pass_data: 将修改后的数据传递给下一步
            if step.pass_data and modified_data is not None:
                working_data = {**working_data, **modified_data}
            
            # 2. 终止动作
            if action == "allow":
                return PipelineExecutionResult(
                    terminal_action="allow",
                    step_results=step_results,
                    modified_data=working_data,
                )
            
            if action == "block":
                return PipelineExecutionResult(
                    terminal_action="block",
                    step_results=step_results,
                    error_message=error_detail,
                )
            
            if action == "modify_response":
                return PipelineExecutionResult(
                    terminal_action="modify_response",
                    step_results=step_results,
                    modify_response_message=step.modify_response_message,
                )
            
            # action == "next" → 继续下一步
```

#### 5.2.2 步骤动作配置

```python
# 动作映射逻辑 (pipeline_executor.py:209-223)
def _pipeline_action_for_outcome(step: PipelineStep, outcome: str) -> str:
    """
    outcome → action 映射:
    - "pass" → step.on_pass
    - "fail" → step.on_fail
    - "error" → step.on_error (如果配置了) 或 step.on_fail
    
    每个 action 可选值:
    - "allow": 立即通过，跳过后续步骤
    - "block": 立即阻止，返回错误
    - "next": 继续下一步
    - "modify_response": 修改响应 (特殊动作)
    """
    if outcome == "pass":
        return step.on_pass
    if outcome == "fail":
        return step.on_fail
    if step.on_error is not None:
        return step.on_error
    return step.on_fail
```

#### 5.2.3 Pipeline 执行流程图

```
Pipeline 开始
    ↓
[Step 1: guardrail_a]
    ├── outcome = "pass"
    │       ├── on_pass = "next" → 继续 Step 2
    │       ├── on_pass = "allow" → 立即通过，终止
    │       └── on_pass = "block" → 立即阻止，返回错误
    │
    ├── outcome = "fail"
    │       ├── on_fail = "next" → 继续 Step 2 (忽略失败)
    │       ├── on_fail = "allow" → 失败但通过 (特殊场景)
    │       └── on_fail = "block" → 阻止，返回错误
    │
    └── outcome = "error"
            └── 同 fail (使用 on_error 或 on_fail)
    ↓
[Step 2: guardrail_b]
    ...
    ↓
[Step N: guardrail_n]
    ↓
所有步骤完成 → 默认 "allow"
```

#### 5.2.4 完整 Pipeline 配置示例

```yaml
# config.yaml 示例
policies:
  - policy_name: "content_safety_policy"
    guardrails:
      add:
        - "prompt_injection_detect"
        - "toxic_content_filter"
        - "pii_masking"
    pipeline:
      mode: "pre_call"
      steps:
        # Step 1: 检查 prompt injection
        - guardrail: "prompt_injection_detect"
          on_pass: "next"
          on_fail: "block"
          on_error: "next"   # 错误时继续
          pass_data: false
        
        # Step 2: 过滤有毒内容
        - guardrail: "toxic_content_filter"
          on_pass: "next"
          on_fail: "block"
          pass_data: false
        
        # Step 3: 掩码 PII (数据传递到下一步)
        - guardrail: "pii_masking"
          on_pass: "allow"   # 通过后允许请求
          on_fail: "block"
          pass_data: true    # 将掩码后的数据传递给实际请求
```

### 5.3 Policy Engine 与 Guardrail 解析

Policy Engine 负责**根据请求上下文确定哪些 guardrails 和 pipelines 应该执行**。

#### 5.3.1 Policy 匹配流程

```
请求上下文 (PolicyMatchContext):
  - team_alias: "team-enterprise"
  - key_alias: "prod-key"
  - model: "gpt-4"
  - tags: ["production", "sensitive"]
    ↓
[PolicyMatcher] 查找匹配的 policies
    ├── 检查 policy_attachments (绑定到团队/key/模型)
    └── 通配符匹配: "*", "gpt-*", "team-*"
    ↓
[PolicyResolver] 解析
    ├── 1. 解析继承链 (inherit)
    │       ├── parent_policy → child_policy
    │       └── 按顺序应用 add/remove
    ├── 2. 评估条件 (condition)
    │       └── 如: model == "gpt-4" and tag == "sensitive"
    ├── 3. 收集 guardrails (union)
    └── 4. 收集 pipelines
    ↓
注入请求 metadata:
  - metadata["_guardrail_pipelines"] → 给 _maybe_execute_pipelines 用
  - metadata["_pipeline_managed_guardrails"] → 避免重复执行
```

#### 5.3.2 继承链解析

```python
# policy_resolver.py:31-69
@staticmethod
def resolve_inheritance_chain(
    policy_name: str,
    policies: Dict[str, Policy],
    visited: Optional[Set[str]] = None,
) -> List[str]:
    """
    示例:
    base_policy (无继承)
        ↓ inherit
    security_policy (继承 base_policy)
        ↓ inherit
    enterprise_policy (继承 security_policy)
    
    结果: ["base_policy", "security_policy", "enterprise_policy"]
    """
    if policy_name in visited:  # 循环检测
        return []
    
    policy = policies.get(policy_name)
    if policy is None:
        return []
    
    visited.add(policy_name)
    
    if policy.inherit:
        parent_chain = PolicyResolver.resolve_inheritance_chain(
            policy_name=policy.inherit, policies=policies, visited=visited
        )
        return parent_chain + [policy_name]
    
    return [policy_name]
```

#### 5.3.3 Add/Remove 语义

```python
# policy_resolver.py:100-126
# 按继承链顺序应用 add/remove

guardrails: Set[str] = set()

for chain_policy_name in inheritance_chain:  # 从 root 到 leaf
    policy = policies.get(chain_policy_name)
    
    # 条件检查
    if context is not None and policy.condition is not None:
        if not ConditionEvaluator.evaluate(policy.condition, context):
            continue  # 条件不匹配，跳过此 policy
    
    # Add
    for guardrail in policy.guardrails.get_add():
        guardrails.add(guardrail)
    
    # Remove
    for guardrail in policy.guardrails.get_remove():
        guardrails.discard(guardrail)
```

**继承链应用示例**：

```yaml
policies:
  - policy_name: "base"
    guardrails:
      add: ["g1", "g2", "g3"]
  
  - policy_name: "security"
    inherit: "base"
    guardrails:
      add: ["g4"]
      remove: ["g2"]
  
  - policy_name: "enterprise"
    inherit: "security"
    guardrails:
      add: ["g5", "g6"]
      remove: ["g3"]

# 解析 enterprise 得到的 guardrails:
# base:     {g1, g2, g3}
# security: {g1, g3, g4}  (add g4, remove g2)
# enterprise: {g1, g4, g5, g6}  (add g5,g6, remove g3)
# 最终: ["g1", "g4", "g5", "g6"]
```

### 5.4 执行顺序优先级总结

```
┌─────────────────────────────────────────────────────────────────────┐
│                      执行顺序优先级 (从高到低)                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  1. ASGI Middleware 层                                              │
│     ├── CORSMiddleware (最先)                                      │
│     ├── PrometheusAuthMiddleware                                    │
│     └── InFlightRequestsMiddleware                                  │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  2. Pre-Call Hook (请求路由后)                                      │
│     ├── Phase A: Policy Engine Pipelines                            │
│     │       └── 按 pipeline.steps 定义的顺序执行                    │
│     │           ├── Step 1 → Step 2 → ... → Step N               │
│     │           └── 每个步骤可配置: allow/block/next/modify       │
│     │                                                               │
│     └── Phase B: 独立 Guardrails                                    │
│             └── 按 litellm.callbacks 注册顺序执行                   │
│                 ├── 过滤: should_run_guardrail()                   │
│                 └── 跳过: pipeline_managed_guardrails              │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  3. During-Call Hook (流式响应时，并行执行)                          │
│     └── asyncio.gather(*guardrail_tasks)                          │
│         注意: 并行执行，无固定顺序                                    │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  4. Post-Call Hook (响应成功后)                                     │
│     ├── 非流式: 按 litellm.callbacks 顺序                           │
│     └── 流式: 组装完成后执行                                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. 关键代码文件索引

### 6.1 核心文件

| 文件路径 | 职责说明 |
|----------|----------|
| `litellm/proxy/proxy_server.py` | Proxy 服务器入口，中间件注册 |
| `litellm/proxy/utils.py` | `ProxyLogging` 类，pre_call/post_call/during_call hooks |
| `litellm/proxy/common_request_processing.py` | 请求处理通用流程 |
| `litellm/proxy/guardrails/guardrail_registry.py` | Guardrail 注册表，初始化器管理 |
| `litellm/proxy/guardrails/init_guardrails.py` | Guardrail 初始化入口 |

### 6.2 Policy Engine 文件

| 文件路径 | 职责说明 |
|----------|----------|
| `litellm/proxy/policy_engine/architecture.md` | 架构说明文档 |
| `litellm/proxy/policy_engine/policy_registry.py` | Policy 内存存储 |
| `litellm/proxy/policy_engine/policy_matcher.py` | 策略匹配 (team/key/model/tags) |
| `litellm/proxy/policy_engine/policy_resolver.py` | 继承链解析 + add/remove 应用 |
| `litellm/proxy/policy_engine/pipeline_executor.py` | Pipeline 步骤执行器 |
| `litellm/proxy/policy_engine/condition_evaluator.py` | 条件表达式求值 |
| `litellm/proxy/policy_engine/attachment_registry.py` | Policy 绑定关系管理 |

### 6.3 Guardrail Hooks 目录

```
litellm/proxy/guardrails/guardrail_hooks/
├── aim/                    # AIM Security
├── akto/                   # Akto API Security
├── aporia_ai/              # Aporia AI
├── azure/                  # Azure Content Safety
│   ├── base.py
│   ├── prompt_shield.py    # Prompt injection
│   └── text_moderation.py  # Content moderation
├── bedrock_guardrails.py   # AWS Bedrock Guardrails
├── block_code_execution/   # 代码执行阻止
├── crowdstrike_aidr/       # CrowdStrike AI Defense
├── custom_code/            # 自定义代码 guardrail
├── custom_guardrail.py     # 自定义 guardrail 基类
├── dynamoai/               # DynamoAI
├── enkryptai/              # EnKrypt AI
├── generic_guardrail_api/  # 通用 API 接口
├── grayswan/               # GraySwan
├── guardrails_ai/          # Guardrails AI
├── hiddenlayer/            # HiddenLayer
├── ibm_guardrails/         # IBM Guardrails
├── javelin/                # Javelin
├── lakera_ai.py            # Lakera AI (v1)
├── lakera_ai_v2.py         # Lakera AI (v2)
├── lasso/                  # Lasso Security
├── litellm_content_filter/ # LiteLLM 内置内容过滤器
│   ├── categories/         # 规则定义 YAML
│   ├── policy_templates/   # 策略模板
│   └── content_filter.py
├── llm_as_a_judge/         # LLM 作为裁判
├── mcp_end_user_permission/# MCP 终端用户权限
├── mcp_security/           # MCP 安全
├── model_armor/            # Model Armor
├── noma/                   # Noma Security
├── openai/                 # OpenAI Moderations
├── pangea/                 # Pangea
├── panw_prisma_airs/       # Palo Alto Networks Prisma
├── pillar/                 # Pillar
├── presidio.py             # Microsoft Presidio (PII)
├── prompt_security/        # Prompt Security
├── promptguard/            # PromptGuard
├── qualifire/              # Qualifire
├── semantic_guard/         # 语义 Guard
├── tool_permission.py      # 工具权限
├── tool_policy/            # 工具策略
├── unified_guardrail/      # 统一 Guardrail 入口
├── xecguard/               # XecGuard
└── zscaler_ai_guard/       # Zscaler AI Guard
```

---

## 7. 附录：配置示例

### 7.1 基础 Guardrail 配置

```yaml
# config.yaml
guardrails:
  - guardrail_name: "lakera_prompt_injection"
    litellm_params:
      guardrail: "lakera_ai"
      mode: "pre_call"           # pre_call / post_call / during_call
      api_key: "os.environ/LAKERA_API_KEY"
      default_on: true            # 默认对所有请求生效
      categories:
        - "prompt_injection"
        - "jailbreak"

  - guardrail_name: "pii_masking"
    litellm_params:
      guardrail: "presidio"
      mode: "pre_call"
      default_on: true
      entities:
        - "EMAIL_ADDRESS"
        - "PERSON"
        - "PHONE_NUMBER"
      masking_strategy: "replace"  # replace / redact / hash / encrypt
```

### 7.2 带 Policy 的配置

```yaml
# config.yaml
policies:
  - policy_name: "base_security"
    guardrails:
      add:
        - "lakera_prompt_injection"
        - "pii_masking"

  - policy_name: "enterprise_security"
    inherit: "base_security"
    guardrails:
      add:
        - "toxic_content_filter"
        - "data_exfiltration_detect"
      remove:
        - "pii_masking"   # 企业环境可能不需要 PII masking
    condition:
      model: "gpt-4"       # 仅对 gpt-4 生效

# 绑定 policy 到团队/模型
policy_attachments:
  - policy_name: "base_security"
    teams: ["*"]            # 所有团队
    
  - policy_name: "enterprise_security"
    teams: ["team-enterprise", "team-security"]
    models: ["gpt-4", "claude-3-opus"]
```

### 7.3 Pipeline 配置

```yaml
# config.yaml
policies:
  - policy_name: "strict_content_policy"
    pipeline:
      mode: "pre_call"
      steps:
        - guardrail: "prompt_injection"
          on_pass: "next"
          on_fail: "block"
          on_error: "next"
          pass_data: false
        
        - guardrail: "malicious_code_detect"
          on_pass: "next"
          on_fail: "block"
          pass_data: false
        
        - guardrail: "input_sanitizer"
          on_pass: "next"
          on_fail: "next"
          pass_data: true   # 传递清理后的数据
        
        - guardrail: "role_based_access"
          on_pass: "allow"
          on_fail: "block"
          pass_data: false

# 绑定
policy_attachments:
  - policy_name: "strict_content_policy"
    teams: ["team-production"]
    models: ["*"]
```

### 7.4 自定义 Guardrail

```python
# my_custom_guardrail.py
from litellm.integrations.custom_guardrail import CustomGuardrail
from fastapi import HTTPException

class MyCustomGuardrail(CustomGuardrail):
    
    async def async_pre_call_hook(
        self,
        user_api_key_dict,
        cache,
        data,
        call_type,
    ):
        """
        在请求发送到 LLM 之前执行
        """
        messages = data.get("messages", [])
        
        # 自定义检查逻辑
        for msg in messages:
            content = msg.get("content", "")
            if self._contains_sensitive_pattern(content):
                raise HTTPException(
                    status_code=400,
                    detail={
                        "error": {
                            "message": "Request blocked by custom guardrail",
                            "type": "custom_guardrail_error",
                            "guardrail": self.guardrail_name,
                        }
                    }
                )
        
        # 可以修改请求数据后返回
        # data["messages"] = modified_messages
        return data
    
    def _contains_sensitive_pattern(self, content: str) -> bool:
        # 自定义模式匹配
        sensitive_keywords = ["internal_api_key", "db_password"]
        return any(kw in content for kw in sensitive_keywords)
```

使用配置：

```yaml
guardrails:
  - guardrail_name: "my_custom_guard"
    litellm_params:
      guardrail: "my_custom_guardrail.MyCustomGuardrail"  # 模块路径.类名
      mode: "pre_call"
      default_on: true
```

---

## 总结

### 关键设计原则

1. **分层架构**：ASGI 中间件 → Callback Manager → Policy Engine → Pipeline Executor
2. **双重注册**：静态注册表 + 动态目录扫描
3. **灵活执行**：独立执行（按注册顺序）+ Pipeline（显式步骤顺序）
4. **上下文感知**：基于 team_alias、key_alias、model、tags 的条件匹配
5. **组合能力**：Policy 继承 + add/remove 语义 + 条件过滤

### 执行顺序决策树

```
请求进入
    │
    ├── [ASGI Middleware] 固定顺序
    │       CORSMiddleware → PrometheusAuth → InFlightRequests
    │
    └── [业务层]
            │
            ├── pre_call_hook
            │       │
            │       ├── Phase 1: Pipelines (按 steps 顺序)
            │       │       Step 1 → Step 2 → ... (可终止: allow/block)
            │       │
            │       └── Phase 2: 独立 Guardrails
            │               按 litellm.callbacks 顺序
            │               过滤: should_run_guardrail
            │
            ├── during_call_hook (流式)
            │       asyncio.gather → 并行，无顺序
            │
            └── post_call_hook
                    按 litellm.callbacks 顺序
```

---

*分析完成。本报告基于 LiteLLM 代码库的实际实现，涵盖了中间件架构、guardrail 注册机制、执行流程和顺序控制的核心设计。*
