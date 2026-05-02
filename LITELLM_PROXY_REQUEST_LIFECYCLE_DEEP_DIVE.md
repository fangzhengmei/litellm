# LiteLLM Proxy 请求生命周期深度分析

> 分析日期：2026-05-02
> 代码版本：基于当前仓库版本
> 重点：中间件真实执行顺序、Guardrail 触发条件分支、各阶段先后关系

---

## 目录

1. [ASGI 中间件：洋葱模型执行顺序](#1-asgi-中间件洋葱模型执行顺序)
2. [Guardrail 触发条件：完整决策流程](#2-guardrail-触发条件完整决策流程)
3. [请求生命周期时间线：精确的先后关系](#3-请求生命周期时间线精确的先后关系)
4. [流式 vs 非流式：关键差异](#4-流式-vs-非流式关键差异)
5. [关键代码位置索引](#5-关键代码位置索引)
6. [附录：配置示例](#6-附录配置示例)

---

## 1. ASGI 中间件：洋葱模型执行顺序

### 1.1 注册顺序

在 `proxy_server.py` 中，中间件按以下顺序注册：

```python
# proxy_server.py:1537-1547

# 第1个注册
app.add_middleware(
    CORSMiddleware,
    allow_origins=origins,
    allow_credentials=allow_cors_credentials,
    allow_methods=["*"],
    allow_headers=["*"],
    expose_headers=LITELLM_UI_ALLOW_HEADERS,
)

# 第2个注册
app.add_middleware(PrometheusAuthMiddleware)

# 第3个注册
app.add_middleware(InFlightRequestsMiddleware)
```

### 1.2 真实执行顺序：洋葱模型

**FastAPI/Starlette 的中间件遵循"洋葱模型"：**

- **最先注册**的中间件 **最后处理请求**，**最先处理响应**
- **最后注册**的中间件 **最先处理请求**，**最后处理响应**

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           注册顺序 vs 执行顺序                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  注册顺序:                                                                │
│  ┌──────────┐    ┌──────────────────────┐    ┌──────────────────────┐ │
│  │   CORS   │ →  │ PrometheusAuth       │ →  │ InFlightRequests     │ │
│  │ (第1个)  │    │ Middleware (第2个)   │    │ Middleware (第3个)   │ │
│  └──────────┘    └──────────────────────┘    └──────────────────────┘ │
│                                                                         │
│  真实执行顺序 (请求进入时):                                               │
│  ┌──────────────────────┐    ┌──────────────────────┐    ┌──────────┐ │
│  │ InFlightRequests     │ →  │ PrometheusAuth       │ →  │   CORS   │ │
│  │ Middleware (最先)    │    │ Middleware           │    │ (最后)   │ │
│  └──────────────────────┘    └──────────────────────┘    └──────────┘ │
│                                                                         │
│  真实执行顺序 (响应返回时):                                               │
│  ┌──────────┐    ┌──────────────────────┐    ┌──────────────────────┐ │
│  │   CORS   │ →  │ PrometheusAuth       │ →  │ InFlightRequests     │ │
│  │ (最先)   │    │ Middleware           │    │ Middleware (最后)     │ │
│  └──────────┘    └──────────────────────┘    └──────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.3 完整执行流程图

```
HTTP 请求到达
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  InFlightRequestsMiddleware (最后注册 → 最先处理请求)         │
│  ├── _in_flight += 1                                        │
│  └── Prometheus gauge.inc()                                 │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  PrometheusAuthMiddleware                                   │
│  └── 仅对 /metrics 路径生效                                  │
│      └── 检查 Authorization header 或 query 参数中的 token   │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  CORSMiddleware (最先注册 → 最后处理请求)                     │
│  ├── 检查 Origin                                             │
│  ├── 处理 OPTIONS 预检请求                                    │
│  └── 添加 CORS 响应头                                         │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  路由处理 (FastAPI Router)                                   │
│  ├── 认证检查 (user_api_key_auth)                           │
│  └── 业务逻辑 (chat_completion 等)                          │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  CORSMiddleware (最先注册 → 最先处理响应)                     │
│  └── 确保 CORS 响应头正确设置                                 │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  PrometheusAuthMiddleware                                   │
│  └── 无响应处理逻辑                                           │
└────────────────────────────────────────────────────────────┘
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  InFlightRequestsMiddleware (最后注册 → 最后处理响应)         │
│  ├── _in_flight -= 1  (在 finally 块中)                     │
│  └── Prometheus gauge.dec()                                 │
└────────────────────────────────────────────────────────────┘
    │
    ▼
响应返回客户端
```

### 1.4 InFlightRequestsMiddleware 实现细节

```python
# proxy/middleware/in_flight_requests_middleware.py:29-51

class InFlightRequestsMiddleware:
    """
    统计当前正在处理的 HTTP 请求数量。
    注意：这是类级别的计数器，同一 worker 进程内共享。
    """
    
    _in_flight: int = 0
    _gauge: Optional[Any] = None
    _gauge_init_attempted: bool = False

    def __init__(self, app: ASGIApp) -> None:
        self.app = app

    async def __call__(self, scope: Scope, receive: Receive, send: Send) -> None:
        # 仅对 HTTP 请求计数
        if scope["type"] != "http":
            await self.app(scope, receive, send)
            return

        # 请求进入时：计数 +1
        InFlightRequestsMiddleware._in_flight += 1
        gauge = InFlightRequestsMiddleware._get_gauge()
        if gauge is not None:
            gauge.inc()
        
        try:
            # 调用下一个中间件/路由
            await self.app(scope, receive, send)
        finally:
            # 响应返回时（无论成功失败）：计数 -1
            InFlightRequestsMiddleware._in_flight -= 1
            if gauge is not None:
                gauge.dec()
```

---

## 2. Guardrail 触发条件：完整决策流程

### 2.1 should_run_guardrail 核心逻辑

Guardrail 是否执行由 `should_run_guardrail()` 方法决定，这是一个**多层级的决策流程**。

```python
# litellm/integrations/custom_guardrail.py:406-476

def should_run_guardrail(
    self,
    data,
    event_type: GuardrailEventHooks,
) -> bool:
    """
    返回 True 表示该 guardrail 应该在当前事件上执行。
    这是一个多层级的决策流程，涉及多种配置方式。
    """
    
    # ========== 步骤 1: 获取元数据 ==========
    
    # 1.1 显式请求的 guardrails (调用方可以指定)
    requested_guardrails = self.get_guardrail_from_metadata(data)
    
    # 1.2 全局关闭开关 (仅管理员可配置)
    disable_global_guardrail = self.get_disable_global_guardrail(data)
    
    # 1.3 选择退出的全局 guardrails (仅管理员可配置)
    opted_out_global_guardrails = (
        self.get_opted_out_global_guardrails_from_metadata(data)
    )
    
    verbose_logger.debug(
        "should_run_guardrail: guardrail=%s, event_type=%s, "
        "default_on=%s, requested_guardrails=%s, "
        "disable_global_guardrail=%s, opted_out=%s",
        self.guardrail_name, event_type,
        self.default_on, requested_guardrails,
        disable_global_guardrail, opted_out_global_guardrails,
    )
    
    # ========== 步骤 2: default_on = True 的情况 ==========
    
    if self.default_on is True:
        
        # 分支 2a: 检查是否被 opt-out
        # 如果管理员在 key/team metadata 中配置了 opted_out_global_guardrails
        # 包含当前 guardrail 名称，则不执行
        if self.guardrail_name in opted_out_global_guardrails:
            return False
        
        # 分支 2b: 检查全局关闭
        # 如果管理员设置了 disable_global_guardrails=True，
        # 且 guardrail 是 default_on=True 的全局 guardrail，则不执行
        if disable_global_guardrail is not True:
            
            # 分支 2c: 检查 event_hook 是否匹配
            if self._event_hook_is_event_type(event_type):
                
                # 分支 2d: 企业版 Mode 类型处理
                # Mode 允许基于请求标签动态决定执行时机
                if isinstance(self.event_hook, Mode):
                    try:
                        from litellm_enterprise.integrations.custom_guardrail import (
                            EnterpriseCustomGuardrailHelper,
                        )
                    except ImportError:
                        raise ImportError(
                            "Setting tag-based guardrails is only available in "
                            "litellm-enterprise."
                        )
                    
                    result = EnterpriseCustomGuardrailHelper._should_run_if_mode_by_tag(
                        data, self.event_hook, event_type
                    )
                    if result is not None:
                        return result
                
                # ✅ default_on=True，且所有条件满足 → 执行
                return True
            
            # ❌ event_hook 不匹配 → 不执行
            return False
    
    # ========== 步骤 3: default_on = False 或未设置 ==========
    # 这种情况下，guardrail 必须被显式请求才会执行
    
    # 分支 3a: 检查是否配置了 event_hook
    # 如果配置了 event_hook，且不在显式请求列表中 → 不执行
    if (
        self.event_hook
        and not self._guardrail_is_in_requested_guardrails(requested_guardrails)
        and event_type.value != "logging_only"
    ):
        return False
    
    # 分支 3b: 检查 event_hook 是否匹配
    if not self._event_hook_is_event_type(event_type):
        return False
    
    # 分支 3c: 企业版 Mode 类型处理
    if isinstance(self.event_hook, Mode):
        try:
            from litellm_enterprise.integrations.custom_guardrail import (
                EnterpriseCustomGuardrailHelper,
            )
        except ImportError:
            raise ImportError(
                "Setting tag-based guardrails is only available in "
                "litellm-enterprise."
            )
        result = EnterpriseCustomGuardrailHelper._should_run_if_mode_by_tag(
            data, self.event_hook, event_type
        )
        if result is not None:
            return result
    
    # ✅ 所有检查通过 → 执行
    return True
```

### 2.2 决策流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                      should_run_guardrail 决策流程                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  输入:                                                                        │
│  ├── self.default_on: bool           (配置文件中设置)                        │
│  ├── self.event_hook: GuardrailEventHooks | Mode  (配置文件中设置)            │
│  ├── event_type: GuardrailEventHooks  (当前事件: pre_call/during_call等)     │
│  └── data: dict                        (请求数据，包含元数据)                   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │  从 data 中提取元数据                                                    │ │
│  │  ├── requested_guardrails: 显式请求的 guardrails                         │ │
│  │  │   来源: data["guardrails"] 或 data["metadata"]["guardrails"]          │ │
│  │  │   权限: 调用方可设置                                                   │ │
│  │  │                                                                        │ │
│  │  ├── disable_global_guardrails: 全局关闭开关                             │ │
│  │  │   来源: user_api_key_metadata.disable_global_guardrails              │ │
│  │  │   权限: 仅管理员可设置                                                 │ │
│  │  │                                                                        │ │
│  │  └── opted_out_global_guardrails: 选择退出列表                            │ │
│  │      来源: user_api_key_metadata.opted_out_global_guardrails            │ │
│  │      权限: 仅管理员可设置                                                 │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│                              │                                               │
│                              ▼                                               │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                     default_on = True ?                                  │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                              │                                               │
│              ┌───────────────┴───────────────┐                               │
│              │                               │                               │
│              ▼                               ▼                               │
│      ┌─────────────┐                 ┌─────────────┐                       │
│      │    True     │                 │   False     │                       │
│      └─────────────┘                 └─────────────┘                       │
│              │                               │                               │
│              ▼                               ▼                               │
│  ┌──────────────────────┐         ┌──────────────────────┐                 │
│  │  全局 Guardrail 分支  │         │  显式请求 Guardrail  │                 │
│  └──────────────────────┘         └──────────────────────┘                 │
│              │                               │                               │
│              ▼                               ▼                               │
│  ┌──────────────────────────┐     ┌──────────────────────────┐             │
│  │ 2a. 是否在 opt-out 列表?  │     │ 3a. 是否有 event_hook    │             │
│  │  opted_out_global_        │     │  且不在请求列表中?         │             │
│  │  guardrails 中?           │     │                          │             │
│  └──────────────────────────┘     └──────────────────────────┘             │
│              │                               │                               │
│      ┌───────┴───────┐               ┌───────┴───────┐                       │
│      │               │               │               │                       │
│      ▼               ▼               ▼               ▼                       │
│   是返回 False    否继续执行      是返回 False    否继续执行                 │
│              │                               │                               │
│              ▼                               ▼                               │
│  ┌──────────────────────────┐     ┌──────────────────────────┐             │
│  │ 2b. disable_global_      │     │ 3b. event_hook 是否匹配  │             │
│  │     guardrails = True?   │     │     当前 event_type?      │             │
│  └──────────────────────────┘     └──────────────────────────┘             │
│              │                               │                               │
│              ▼                               ▼                               │
│         是返回 False                    ┌───────┴───────┐                   │
│              │                         │               │                   │
│              ▼                         ▼               ▼                   │
│  ┌──────────────────────────┐      是返回 True    否返回 False            │
│  │ 2c. event_hook 是否匹配   │                                           │
│  │     当前 event_type?      │                                           │
│  └──────────────────────────┘                                           │
│              │                                                             │
│      ┌───────┴───────┐                                                     │
│      │               │                                                     │
│      ▼               ▼                                                     │
│   是继续执行        否返回 False                                            │
│              │                                                             │
│              ▼                                                             │
│  ┌──────────────────────────┐                                             │
│  │ 2d. 企业版 Mode 类型处理   │                                             │
│  │     (基于标签动态决定)      │                                             │
│  └──────────────────────────┘                                             │
│              │                                                             │
│              ▼                                                             │
│         返回 True ✅                                                        │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 2.3 三种配置方式的优先级

| 配置方式 | 来源 | 权限 | 优先级 | 说明 |
|----------|------|------|--------|------|
| **显式请求** | `data["guardrails"]` 或 `data["metadata"]["guardrails"]` | 调用方 | **高** | `default_on=False` 时必须显式指定 |
| **opt-out** | `user_api_key_metadata.opted_out_global_guardrails` | 管理员 | **中** | 仅覆盖 `default_on=True` 的 guardrails |
| **全局关闭** | `user_api_key_metadata.disable_global_guardrails` | 管理员 | **低** | 仅影响 `default_on=True` 且未 opt-out 的 |
| **default_on** | Guardrail 初始化配置 | 配置文件 | **基础** | 全局生效，可被 opt-out 和全局关闭覆盖 |

### 2.4 元数据来源详解

#### 2.4.1 显式请求的 Guardrails

```python
# litellm/integrations/custom_guardrail.py:289-308

def get_guardrail_from_metadata(
    self, data: dict
) -> Union[List[str], List[Dict[str, DynamicGuardrailParams]]]:
    """
    从请求数据中提取显式请求的 guardrails。
    
    优先级:
    1. data["guardrails"] (请求体根级别)
    2. data["metadata"]["guardrails"]
    3. data["litellm_metadata"]["guardrails"]
    
    调用方可以通过以下方式指定:
    - {"guardrails": ["my_guardrail"]}
    - {"metadata": {"guardrails": ["my_guardrail"]}}
    """

    # 优先级 1: 请求体根级别
    if "guardrails" in data:
        return data["guardrails"]
    
    # 优先级 2 & 3: metadata 中的 guardrails
    # 检查 "metadata" 和 "litellm_metadata" 两个位置
    # 避免一个为空但另一个有值时被覆盖
    for meta_key in ("metadata", "litellm_metadata"):
        meta = data.get(meta_key) or {}
        if isinstance(meta, dict) and "guardrails" in meta:
            return meta.get("guardrails") or []
    
    return []
```

#### 2.4.2 管理员配置的元数据

```python
# litellm/integrations/custom_guardrail.py:227-247

@staticmethod
def _get_admin_metadata(data: dict) -> dict:
    """
    从管理员配置的元数据中提取设置。
    
    重要: 只读取管理员注入的元数据，不读取请求体中的用户元数据。
    这是为了防止用户通过设置请求体中的 metadata 来禁用 guardrails。
    
    来源:
    - user_api_key_metadata (API Key 级别的元数据)
    - user_api_key_team_metadata (Team 级别的元数据)
    
    优先级: Key 级别设置覆盖 Team 级别设置
    """
    team_meta: dict = {}
    key_meta: dict = {}
    
    for key in ("metadata", "litellm_metadata"):
        meta = data.get(key)
        if not isinstance(meta, dict):
            continue
        
        # Team 级别元数据
        team_meta = meta.get("user_api_key_team_metadata") or team_meta
        
        # Key 级别元数据 (优先级更高)
        key_meta = meta.get("user_api_key_metadata") or key_meta
    
    # 合并: Key 级别覆盖 Team 级别
    return {**team_meta, **key_meta}
```

```python
# litellm/integrations/custom_guardrail.py:249-256

def get_disable_global_guardrail(self, data: dict) -> Optional[bool]:
    """
    检查是否应该禁用所有全局 guardrails (default_on=True)。
    
    来源: user_api_key_metadata.disable_global_guardrails
    权限: 仅管理员可配置
    """
    return self._get_admin_metadata(data).get("disable_global_guardrails", False)
```

```python
# litellm/integrations/custom_guardrail.py:258-265

def get_opted_out_global_guardrails_from_metadata(self, data: dict) -> List[str]:
    """
    获取当前请求选择退出的全局 guardrails 列表。
    
    来源: user_api_key_metadata.opted_out_global_guardrails
    权限: 仅管理员可配置
    """
    value = self._get_admin_metadata(data).get("opted_out_global_guardrails")
    return value if isinstance(value, list) else []
```

### 2.5 实际配置场景示例

#### 场景 1: 全局 Guardrail，所有请求都执行

```yaml
# config.yaml
guardrails:
  - guardrail_name: "lakera_prompt_injection"
    litellm_params:
      guardrail: "lakera_ai"
      mode: "pre_call"
      api_key: "os.environ/LAKERA_API_KEY"
      default_on: true  # 全局启用
```

**执行条件**：
- ✅ `default_on=True`
- ❌ 不在 `opted_out_global_guardrails` 中
- ❌ `disable_global_guardrails` 为 False
- ✅ `event_hook` 匹配

**结果**：所有请求都会执行

---

#### 场景 2: 某个 Team 选择退出某个 Guardrail

```yaml
# Team 元数据配置 (通过 UI 或 API 设置)
user_api_key_team_metadata:
  opted_out_global_guardrails:
    - "lakera_prompt_injection"  # 选择退出
```

**执行条件**：
- ✅ `default_on=True`
- ✅ 在 `opted_out_global_guardrails` 中 → **返回 False**

**结果**：该 Team 的请求不会执行此 guardrail

---

#### 场景 3: 全局关闭所有 Guardrails

```yaml
# Key 元数据配置
user_api_key_metadata:
  disable_global_guardrails: true
```

**执行条件**：
- ✅ `default_on=True`
- ❌ 不在 `opted_out_global_guardrails` 中
- ✅ `disable_global_guardrails=True` → **返回 False**

**结果**：该 Key 的请求不会执行任何 `default_on=True` 的 guardrails

---

#### 场景 4: 显式请求 Guardrail

```yaml
# config.yaml
guardrails:
  - guardrail_name: "custom_content_check"
    litellm_params:
      guardrail: "custom_code"
      mode: "pre_call"
      default_on: false  # 不全局启用
```

**请求时显式指定**：
```json
{
  "model": "gpt-4",
  "messages": [...],
  "guardrails": ["custom_content_check"]
}
```

**执行条件**：
- ❌ `default_on=True` (跳过 default_on 分支)
- ✅ `event_hook` 配置了
- ✅ 在 `requested_guardrails` 中
- ✅ `event_hook` 匹配 → **返回 True**

**结果**：仅当请求中显式指定时才执行

---

## 3. 请求生命周期时间线：精确的先后关系

### 3.1 非流式请求完整时间线

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    非流式请求 (stream: false) 完整时间线                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  时间轴                                                                      │
│  ─────────────────────────────────────────────────────────────────────────▶ │
│                                                                             │
│  [T1] HTTP 请求到达                                                          │
│       │                                                                     │
│       ▼                                                                     │
│  [T2] ASGI 中间件链                                                          │
│       ├── InFlightRequestsMiddleware (_in_flight += 1)                      │
│       ├── PrometheusAuthMiddleware (仅 /metrics)                             │
│       └── CORSMiddleware                                                     │
│       │                                                                     │
│       ▼                                                                     │
│  [T3] 路由匹配 + 认证检查                                                     │
│       ├── 路由匹配: /v1/chat/completions                                    │
│       └── user_api_key_auth: 验证 API Key 权限                               │
│       │                                                                     │
│       ▼                                                                     │
│  [T4] common_processing_pre_call_logic()                                    │
│       ├── add_litellm_data_to_request()                                     │
│       ├── litellm.utils.function_setup()  (创建 logging_obj)                │
│       │                                                                     │
│       └── [T4a] pre_call_hook()  ←─── Guardrail 执行点 1                   │
│           │                                                                 │
│           ├── Phase A: Policy Engine Pipelines                              │
│           │   └── 按 steps 定义的顺序执行                                   │
│           │       ├── 每个 step 可配置 allow/block/next                    │
│           │       └── 阻塞请求: raise HTTPException                         │
│           │                                                                 │
│           └── Phase B: 独立 Guardrails                                      │
│               └── 按 litellm.callbacks 注册顺序执行                         │
│                   ├── should_run_guardrail() 检查                           │
│                   └── async_pre_call_hook() 执行                            │
│                       └── 可修改请求数据或阻塞请求                           │
│       │                                                                     │
│       ▼                                                                     │
│  [T5] base_process_llm_request()                                             │
│       │                                                                     │
│       ├── [T5a] asyncio.create_task(during_call_hook())  ←── 提前启动       │
│       │   │                                                                 │
│       │   └── 🔴 并行执行: during_call_hook 在后台运行                      │
│       │                                                                     │
│       ├── [T5b] route_request()  → 实际 LLM API 调用                        │
│       │   │                                                                 │
│       │   └── 🔴 同时: during_call_hook 仍在后台执行                       │
│       │                                                                     │
│       ├── [T5c] asyncio.gather(*tasks)  ←── 等待两者完成                    │
│       │   │                                                                 │
│       │   ├── responses[0] = during_call_hook 结果                          │
│       │   └── responses[1] = LLM 响应                                       │
│       │                                                                     │
│       └── [T5d] post_call_success_hook()  ←─── Guardrail 执行点 2          │
│           │                                                                 │
│           ├── should_run_guardrail() 检查 (event_type=post_call)           │
│           ├── async_post_call_success_hook() 执行                           │
│           │   ├── 可检查/修改响应                                            │
│           │   └── 可阻塞响应 (raise HTTPException)                          │
│           │                                                                 │
│           └── ⚠️  关键点: 响应尚未返回给客户端                                │
│               可以拦截、修改或拒绝                                            │
│       │                                                                     │
│       ▼                                                                     │
│  [T6] 响应返回客户端                                                          │
│       │                                                                     │
│       ▼                                                                     │
│  [T7] ASGI 中间件响应阶段                                                     │
│       ├── CORSMiddleware                                                    │
│       ├── PrometheusAuthMiddleware                                          │
│       └── InFlightRequestsMiddleware (_in_flight -= 1, finally 块)         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 during_call_hook 的并行执行机制

```python
# litellm/proxy/common_request_processing.py:1096-1129

# 关键代码分析
tasks = []

# 步骤 1: 提前启动 during_call_hook
# 注意: 使用 asyncio.create_task() 创建后台任务
# 这意味着 during_call_hook 会与后续的 route_request() 并行执行
tasks.append(
    asyncio.create_task(
        proxy_logging_obj.during_call_hook(
            data=self.data,
            user_api_key_dict=user_api_key_dict,
            call_type=route_type,
        )
    )
)

# 步骤 2: 执行实际 LLM 调用
# 注意: 这里使用 await，所以当前协程会阻塞等待
# 但 during_call_hook 的 task 仍在后台运行
llm_call = await route_request(
    data=self.data,
    route_type=route_type,
    llm_router=llm_router,
    user_model=user_model,
    user_api_key_dict=user_api_key_dict,
)
tasks.append(llm_call)

# 步骤 3: 等待所有任务完成
# 此时 LLM 调用已完成，等待 during_call_hook 也完成
llm_responses = asyncio.gather(*tasks)
responses = await llm_responses

# responses[0] = during_call_hook 结果
# responses[1] = LLM 响应
```

**并行执行图解**：

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    during_call_hook 与 LLM 调用的并行关系                       │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  时间轴                                                                       │
│  ─────────────────────────────────────────────────────────────────────────▶  │
│                                                                              │
│  [T0] create_task(during_call_hook)                                          │
│       │                                                                      │
│       ├──▶ during_call_hook 开始执行 (后台)                                  │
│       │      │                                                              │
│       │      ▼                                                              │
│       │    [guardrail 执行中]                                                │
│       │      │                                                              │
│       │      │                                                              │
│       ▼      │                                                              │
│  [T1] await route_request()                                                  │
│       │      │                                                              │
│       │      ├──▶ 两者并行执行                                              │
│       │      │                                                              │
│       ▼      │                                                              │
│  [T2] LLM 调用完成                                                           │
│       │      │                                                              │
│       │      ▼                                                              │
│       │    during_call_hook 可能还在执行                                    │
│       │      │                                                              │
│       ▼      │                                                              │
│  [T3] await asyncio.gather(*tasks)                                          │
│       │      │                                                              │
│       │      ▼                                                              │
│       │    等待 during_call_hook 完成                                       │
│       │      │                                                              │
│       ▼      │                                                              │
│  [T4] 两者都完成                                                             │
│       │                                                                      │
│       ▼                                                                      │
│  [T5] post_call_success_hook()                                               │
│                                                                              │
│  关键点:                                                                      │
│  ├── during_call_hook 与 LLM 调用真正并行                                   │
│  ├── 这样可以减少总延迟 (重叠执行)                                            │
│  └── 但 during_call_hook 无法阻止 LLM 调用已发出                              │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 3.3 三个阶段的 Guardrail 对比

| 阶段 | 触发时机 | 能否阻塞请求 | 能否修改数据 | 执行方式 |
|------|----------|--------------|--------------|----------|
| **pre_call** | 路由请求之前 | ✅ 可以 | ✅ 可以修改请求 | 顺序执行 |
| **during_call** | 与 LLM 调用并行 | ❌ 无法阻止已发出的请求 | ✅ 可以修改 (流式) | 并行执行 |
| **post_call** | LLM 响应返回后 | ✅ 可以拦截响应 (非流式) | ✅ 可以修改响应 | 顺序执行 |

---

## 4. 流式 vs 非流式：关键差异

### 4.1 核心差异图解

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    非流式请求 vs 流式请求的 Guardrail 差异                      │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        非流式请求 (stream: false)                         │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  pre_call_hook → during_call_hook (并行) → LLM 完成 → post_call → 返回    │
│                                                        │                     │
│                                                        ▼                     │
│                                             ┌─────────────────┐             │
│                                             │ 响应尚未发送    │             │
│                                             │ post_call 可以: │             │
│                                             │  - 拦截响应     │             │
│                                             │  - 修改响应     │             │
│                                             │  - 记录日志     │             │
│                                             └─────────────────┘             │
│                                                                              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                         流式请求 (stream: true)                           │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  pre_call_hook → during_call_hook (并行) → LLM 返回 chunks                 │
│       │                                         │                           │
│       │                                         ▼                           │
│       │                              立即发送 chunk 给客户端                  │
│       │                                         │                           │
│       │                                         ▼                           │
│       │                              流结束 (所有 chunk 已发送)              │
│       │                                         │                           │
│       │                                         ▼                           │
│       │                              _on_deferred_stream_complete 回调      │
│       │                                         │                           │
│       │                                         ▼                           │
│       │                              _run_deferred_stream_guardrails()      │
│       │                                         │                           │
│       ▼                                         ▼                           │
│  ┌─────────────────┐                    ┌─────────────────┐                 │
│  │ pre_call 可以:  │                    │ post_call 仅:   │                 │
│  │  - 拦截请求     │                    │  - 记录日志     │                 │
│  │  - 修改请求     │                    │  - 审计          │                 │
│  │  - 记录日志     │                    │  ❌ 无法拦截    │                 │
│  └─────────────────┘                    │  ❌ 无法修改    │                 │
│                                          └─────────────────┘                 │
│                                                                              │
│  ⚠️  关键警示: 流式请求中，post_call guardrails 在流结束后才执行              │
│     此时所有内容已发送给客户端，无法阻止或修改已发送的内容                      │
│                                                                              │
└──────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 流式请求延迟执行机制

```python
# litellm/proxy/common_request_processing.py:1212-1233

# 检查是否有 post_call guardrails 且是流式响应
if _post_call_guardrails_active and isinstance(
    response, CustomStreamWrapper
):
    # 捕获当前上下文的引用
    _captured_data = self.data
    _captured_user_api_key_dict = user_api_key_dict
    _captured_logging_obj = logging_obj

    # 定义延迟执行的回调函数
    async def _on_deferred_stream_complete(
        assembled_response, cache_hit
    ):
        # 流结束后才执行
        await ProxyBaseLLMRequestProcessing._run_deferred_stream_guardrails(
            captured_data=_captured_data,
            captured_user_api_key_dict=_captured_user_api_key_dict,
            captured_logging_obj=_captured_logging_obj,
            assembled_response=assembled_response,
            cache_hit=cache_hit,
        )

    # 将回调注册到 logging_obj
    # CustomStreamWrapper 会在流结束时调用此回调
    logging_obj._on_deferred_stream_complete = _on_deferred_stream_complete
```

### 4.3 延迟执行的 Guardrail 逻辑

```python
# litellm/proxy/common_request_processing.py:1544-1651

@staticmethod
async def _run_deferred_stream_guardrails(
    captured_data: dict,
    captured_user_api_key_dict: "UserAPIKeyAuth",
    captured_logging_obj: Any,
    assembled_response: Any,
    cache_hit: Any,
) -> None:
    """
    在流式响应结束后执行 post_call guardrails。
    
    ⚠️  重要提示:
    - 此函数在所有 chunk 已发送给客户端后才执行
    - 仅用于审计和日志记录
    - 无法拦截或修改已发送的内容
    """
    
    from litellm.litellm_core_utils.thread_pool_executor import executor

    _response = assembled_response
    try:
        from litellm.proxy.proxy_server import llm_router as _global_llm_router
        from litellm.proxy.utils import _check_and_merge_model_level_guardrails

        # 合并模型级别的 guardrails
        guardrail_data = _check_and_merge_model_level_guardrails(
            data=captured_data, llm_router=_global_llm_router
        )
        
        # 遍历所有 guardrails
        for cb in litellm.callbacks:
            if not isinstance(cb, CustomGuardrail):
                continue
            
            # 检查是否应该执行 (event_type=post_call)
            if not cb.should_run_guardrail(
                data=guardrail_data,
                event_type=GuardrailEventHooks.post_call,
            ):
                continue
            
            try:
                # 跳过有 apply_guardrail 方法的 guardrails
                # 这些已经通过 unified_guardrail 的流式迭代器执行过了
                if "apply_guardrail" in type(cb).__dict__:
                    # 避免重复调用 API (如 OpenAI Moderation 会产生双重费用)
                    continue
                else:
                    # 执行 post_call hook
                    guardrail_result = await cb.async_post_call_success_hook(
                        user_api_key_dict=captured_user_api_key_dict,
                        data=guardrail_data,
                        response=_response,
                    )
                
                # 即使 guardrail 修改了响应，也无法阻止已发送的内容
                if guardrail_result is not None:
                    _response = guardrail_result
                    
            except Exception as e:
                # 记录错误，但无法影响已发送的响应
                verbose_proxy_logger.exception(
                    "Error running post-call guardrail %s on streaming response: %s",
                    getattr(cb, "guardrail_name", type(cb).__name__),
                    e,
                )
                
    except Exception as e:
        verbose_proxy_logger.exception(
            "Error in deferred streaming guardrail initialization: %s",
            e,
        )
        
    finally:
        # 触发日志记录 (审计用)
        try:
            asyncio.create_task(
                captured_logging_obj.async_success_handler(
                    _response,
                    cache_hit=cache_hit,
                    start_time=None,
                    end_time=None,
                )
            )
        except Exception as e:
            verbose_proxy_logger.exception(
                "Error in deferred streaming async logging: %s",
                e,
            )

        try:
            executor.submit(
                captured_logging_obj.success_handler,
                _response,
                cache_hit=cache_hit,
                start_time=None,
                end_time=None,
            )
        except Exception as e:
            verbose_proxy_logger.exception(
                "Error in deferred streaming sync logging: %s",
                e,
            )
```

### 4.4 流式请求安全建议

| 安全需求 | 实现方式 | 说明 |
|----------|----------|------|
| **阻止敏感请求** | `pre_call` guardrail | 在请求发出前拦截，流式和非流式都有效 |
| **实时内容过滤** | `during_call` + `apply_guardrail` | 使用 unified_guardrail 的流式拦截机制 |
| **审计日志** | `post_call` guardrail | 流结束后记录，无法阻止已发送内容 |
| **响应修改** | 不推荐在流式中使用 | 内容已发送，修改无意义 |

**⚠️  重要安全警示**：

如果您需要在流式请求中进行内容安全检查：

1. **仅依赖 pre_call**：pre_call 可以在请求发出前拦截
2. **使用 unified_guardrail**：它提供流式内容过滤（通过 `apply_guardrail`）
3. **不要依赖 post_call**：流式中的 post_call 仅用于审计，无法阻止已发送内容

---

## 5. 关键代码位置索引

### 5.1 中间件相关

| 功能 | 文件 | 行号 |
|------|------|------|
| 中间件注册 | `litellm/proxy/proxy_server.py` | 1537-1547 |
| InFlightRequestsMiddleware | `litellm/proxy/middleware/in_flight_requests_middleware.py` | 29-51 |
| PrometheusAuthMiddleware | `litellm/proxy/middleware/prometheus_auth_middleware.py` | - |

### 5.2 Guardrail 触发条件

| 功能 | 文件 | 行号 |
|------|------|------|
| should_run_guardrail 核心逻辑 | `litellm/integrations/custom_guardrail.py` | 406-476 |
| get_guardrail_from_metadata | `litellm/integrations/custom_guardrail.py` | 289-308 |
| get_disable_global_guardrail | `litellm/integrations/custom_guardrail.py` | 249-256 |
| get_opted_out_global_guardrails | `litellm/integrations/custom_guardrail.py` | 258-265 |
| _get_admin_metadata | `litellm/integrations/custom_guardrail.py` | 227-247 |
| _event_hook_is_event_type | `litellm/integrations/custom_guardrail.py` | 478-505 |

### 5.3 请求生命周期

| 功能 | 文件 | 行号 |
|------|------|------|
| common_processing_pre_call_logic | `litellm/proxy/common_request_processing.py` | 746-908 |
| pre_call_hook 调用 | `litellm/proxy/common_request_processing.py` | 884-886 |
| during_call_hook 并行启动 | `litellm/proxy/common_request_processing.py` | 1096-1107 |
| asyncio.gather 等待 | `litellm/proxy/common_request_processing.py` | 1124-1129 |
| 非流式 post_call | `litellm/proxy/common_request_processing.py` | 1296-1300 |
| 流式延迟回调注册 | `litellm/proxy/common_request_processing.py` | 1212-1233 |
| _run_deferred_stream_guardrails | `litellm/proxy/common_request_processing.py` | 1544-1651 |
| _has_post_call_guardrails | `litellm/proxy/common_request_processing.py` | 1490-1504 |

### 5.4 Policy Engine 相关

| 功能 | 文件 | 行号 |
|------|------|------|
| Pipeline 执行器 | `litellm/proxy/policy_engine/pipeline_executor.py` | - |
| Policy 解析器 | `litellm/proxy/policy_engine/policy_resolver.py` | - |
| Policy 匹配器 | `litellm/proxy/policy_engine/policy_matcher.py` | - |

---

## 6. 附录：配置示例

### 6.1 完整 Guardrail 配置示例

```yaml
# config.yaml

# ========== Guardrails 定义 ==========
guardrails:
  # 1. 全局 Guardrail - 所有请求默认执行
  - guardrail_name: "lakera_prompt_injection"
    litellm_params:
      guardrail: "lakera_ai"
      mode: "pre_call"
      api_key: "os.environ/LAKERA_API_KEY"
      default_on: true  # 全局启用
      categories:
        - "prompt_injection"
        - "jailbreak"

  # 2. 按需 Guardrail - 必须显式请求才执行
  - guardrail_name: "content_safety_check"
    litellm_params:
      guardrail: "azure"
      mode: ["pre_call", "post_call"]  # 支持多个阶段
      api_key: "os.environ/AZURE_API_KEY"
      api_base: "os.environ/AZURE_API_BASE"
      default_on: false  # 不全局启用，必须显式请求

  # 3. 流式专用 Guardrail
  - guardrail_name: "realtime_moderation"
    litellm_params:
      guardrail: "openai"
      mode: "during_call"  # 仅在流式中执行
      api_key: "os.environ/OPENAI_API_KEY"
      default_on: true

# ========== Policy 配置 (企业版) ==========
policies:
  - policy_name: "strict_security_policy"
    guardrails:
      add:
        - "lakera_prompt_injection"
        - "content_safety_check"
    pipeline:
      mode: "pre_call"
      steps:
        - guardrail: "lakera_prompt_injection"
          on_pass: "next"
          on_fail: "block"
          pass_data: false
        
        - guardrail: "content_safety_check"
          on_pass: "allow"
          on_fail: "block"
          pass_data: true
```

### 6.2 显式请求 Guardrail 的请求示例

```bash
# 示例 1: 请求体根级别指定
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}],
    "guardrails": ["content_safety_check"]
  }'

# 示例 2: metadata 中指定
curl -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-xxx" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "Hello"}],
    "metadata": {
      "guardrails": ["content_safety_check"]
    }
  }'
```

### 6.3 管理员配置元数据示例

```python
# 通过 API 设置 Team 或 Key 的元数据
# 这会影响 default_on=True 的 guardrails

# 示例 1: 某个 Team 选择退出特定 guardrail
{
  "team_id": "team-sensitive-data",
  "metadata": {
    "opted_out_global_guardrails": ["lakera_prompt_injection"]
  }
}

# 示例 2: 某个 Key 禁用所有全局 guardrails
{
  "key_id": "sk-test-key",
  "metadata": {
    "disable_global_guardrails": true
  }
}

# 示例 3: 组合使用
{
  "key_id": "sk-special-key",
  "metadata": {
    "disable_global_guardrails": true,
    "opted_out_global_guardrails": ["realtime_moderation"]
  }
}
```

---

## 总结

### 关键发现

1. **ASGI 中间件执行顺序**：
   - 注册顺序: CORSMiddleware → PrometheusAuth → InFlightRequests
   - 真实执行顺序（请求进入时）: InFlightRequests → PrometheusAuth → CORSMiddleware
   - 真实执行顺序（响应返回时）: CORSMiddleware → PrometheusAuth → InFlightRequests
   - 这是 FastAPI/Starlette 的"洋葱模型"

2. **Guardrail 触发条件**：
   - `default_on=True`: 全局生效，可被 `opted_out_global_guardrails` 和 `disable_global_guardrails` 覆盖
   - `default_on=False`: 必须显式在 `metadata.guardrails` 中指定才执行
   - 管理员配置的元数据优先级高于用户请求体中的元数据

3. **三个阶段的先后关系**：
   - `pre_call_hook`: 在路由请求之前执行，可以拦截和修改请求
   - `during_call_hook`: 通过 `asyncio.create_task()` 提前启动，与 LLM 调用真正并行
   - `post_call_hook`: LLM 响应返回后执行

4. **流式 vs 非流式的关键差异**：
   - 非流式: `post_call` 在响应返回客户端之前执行，可以拦截和修改
   - 流式: `post_call` 通过 `_on_deferred_stream_complete` 延迟执行，在流结束后才执行
   - ⚠️ 流式中的 `post_call` guardrails 仅用于审计，无法阻止已发送的内容

### 安全建议

| 场景 | 推荐做法 |
|------|----------|
| 阻止恶意请求 | 使用 `pre_call` guardrail |
| 流式内容过滤 | 使用 `unified_guardrail` + `apply_guardrail` |
| 审计日志 | 使用 `post_call` guardrail |
| 敏感数据环境 | 避免依赖流式 `post_call` 做安全检查 |

---

*深度分析完成。本报告聚焦于中间件真实执行顺序、Guardrail 触发条件的完整决策流程、各阶段的精确时间线，以及流式与非流式请求的关键差异。*
