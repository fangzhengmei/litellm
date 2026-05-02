# LiteLLM Proxy 请求链路分析 - 修正报告

> 本文档修正并补充之前的分析报告，重点澄清主密钥处理、限流钩子装配顺序、状态码映射和跳过通用检查的场景。

---

## 目录

1. [主密钥与 PROXY_ADMIN 的权限检查](#1-主密钥与-proxy_admin-的权限检查)
2. [限流钩子的真实装配与执行顺序](#2-限流钩子的真实装配与执行顺序)
3. [状态码映射校正](#3-状态码映射校正)
4. [跳过通用检查的场景及其影响](#4-跳过通用检查的场景及其影响)
5. [关键代码位置索引](#5-关键代码位置索引)

---

## 1. 主密钥与 PROXY_ADMIN 的权限检查

### 1.1 关键发现：PROXY_ADMIN 仍然经过 common_checks

**重要修正**：之前的分析可能暗示 PROXY_ADMIN 主密钥会跳过所有权限检查，但实际并非如此。

从 `_run_centralized_common_checks` 的注释明确说明：

```python
# litellm/proxy/auth/user_api_key_auth.py:1620-1624
"""
- ``PROXY_ADMIN`` tokens still run through ``common_checks`` so
  team-blocked / team-budget / end-user-budget / tag-budget /
  vector-store / tool-allowlist enforcement applies to admin keys
  too. Admin status is honored where the underlying check exempts it
  (``_is_api_route_allowed``, ``organization_role_based_access_check``).
"""
```

### 1.2 主密钥检查矩阵

| 检查类型 | PROXY_ADMIN 是否受限制 | 豁免检查的场景 |
|---------|----------------------|----------------|
| **团队阻塞状态** | ✅ 受限制 | 无 |
| **团队预算** | ✅ 受限制 | 无 |
| **组织预算** | ✅ 受限制 | 无 |
| **标签预算** | ✅ 受限制 | 无 |
| **终端用户预算** | ✅ 受限制 | 无 |
| **模型访问权限** | ✅ 受限制 | 无 |
| **向量存储权限** | ✅ 受限制 | 无 |
| **工具白名单** | ✅ 受限制 | 无 |
| **路由权限 (管理路由)** | ❌ 豁免 | `_is_api_route_allowed` 检查豁免 |
| **组织角色权限** | ❌ 豁免 | `organization_role_based_access_check` 豁免 |

### 1.3 主密钥与 Virtual Key 的区别

```
┌─────────────────────────────────────────────────────────────────┐
│                     主密钥 (Master Key)                          │
├─────────────────────────────────────────────────────────────────┤
│  来源: LITELLM_MASTER_KEY 环境变量 或 config.yaml master_key    │
│  前缀: 无强制要求 (通常 sk-)                                     │
│  存储: 不存储在数据库 (除非 disable_adding_master_key_hash_to_db │
│        为 False)                                                 │
│  角色: PROXY_ADMIN (当匹配时)                                    │
│  检查: 经过 common_checks，但部分路由权限检查豁免                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    虚拟密钥 (Virtual Key)                         │
├─────────────────────────────────────────────────────────────────┤
│  来源: /key/generate 端点生成                                     │
│  前缀: sk- (强制)                                                │
│  存储: LiteLLM_VerificationToken 表 (SHA-256 哈希)              │
│  角色: INTERNAL_USER 或关联的团队/用户角色                        │
│  检查: 完整 common_checks 检查                                    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.4 主密钥匹配逻辑

```python
# litellm/proxy/spend_tracking/spend_tracking_utils.py:55-66
def _is_master_key(api_key: Optional[str], _master_key: Optional[str]) -> bool:
    if _master_key is None or api_key is None:
        return False
    
    # 首先进行原始密钥比较
    is_master_key = secrets.compare_digest(api_key, _master_key)
    if is_master_key:
        return True
    
    # 然后进行哈希比较 (主密钥会被哈希存储在某些场景)
    is_master_key = secrets.compare_digest(api_key, hash_token(_master_key))
    if is_master_key:
        return True
```

### 1.5 主密钥禁用数据库存储配置

```python
# litellm/proxy/spend_tracking/spend_tracking_utils.py:299-302
_is_master_key(api_key=api_key, _master_key=master_key)
and general_settings.get("disable_adding_master_key_hash_to_db") is True
# 此时使用别名: "litellm_proxy_master_key"
```

---

## 2. 限流钩子的真实装配与执行顺序

### 2.1 默认 PROXY_HOOKS 配置

**重要修正**：`DynamicRateLimitHandlerV3` 不在默认 `PROXY_HOOKS` 中。

```python
# litellm/proxy/hooks/__init__.py:22-30
PROXY_HOOKS = {
    "max_budget_limiter": _PROXY_MaxBudgetLimiter,           # 个人预算限制
    "parallel_request_limiter": _PROXY_MaxParallelRequestsHandler_v3,  # TPM/RPM/并行请求 (默认 v3)
    "cache_control_check": _PROXY_CacheControlCheck,         # 缓存控制
    "responses_id_security": ResponsesIDSecurity,             # 响应 ID 安全
    "litellm_skills": SkillsInjectionHook,                    # 技能注入
    "max_iterations_limiter": _PROXY_MaxIterationsHandler,   # 最大迭代限制
    "max_budget_per_session_limiter": _PROXY_MaxBudgetPerSessionHandler,  # 每会话预算
}
```

### 2.2 遗留模式切换

```python
# litellm/proxy/hooks/__init__.py:33-34
if os.getenv("LEGACY_MULTI_INSTANCE_RATE_LIMITING", "false").lower() == "true":
    PROXY_HOOKS["parallel_request_limiter"] = _PROXY_MaxParallelRequestsHandler
```

### 2.3 DynamicRateLimitHandlerV3 与 ParallelRequestLimiterV3 的关系

**核心发现**：`DynamicRateLimitHandlerV3` 是 `ParallelRequestLimiterV3` 的**包装器**，而非独立实现。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DynamicRateLimitHandlerV3                             │
│  (优先级感知 + 饱和度感知的增强版)                                        │
├─────────────────────────────────────────────────────────────────────────┤
│  内部组合: self.v3_limiter = _PROXY_MaxParallelRequestsHandler_v3(...) │
│                                                                         │
│  新增功能:                                                               │
│  - 模型饱和度检查 (saturation check)                                    │
│  - 优先级预留 (Priority Reservation)                                    │
│  - 三阶段检查 (只读 → 决策 → 原子增量)                                   │
└─────────────────────────────────────────────────────────────────────────┘
                              │
                              │ 委托调用
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                  _PROXY_MaxParallelRequestsHandler_v3                     │
│  (基础 TPM/RPM/并行请求限制器)                                             │
├─────────────────────────────────────────────────────────────────────────┤
│  核心功能:                                                               │
│  - 多维度速率限制 (Key/Team/Org/Project/Model)                          │
│  - Redis 原子计数器 (Lua 脚本)                                           │
│  - 滑动窗口算法                                                           │
│  - 支持 TPM/RPM/MaxParallelRequests                                      │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.4 DynamicRateLimitHandlerV3 三阶段检查流程

```python
# litellm/proxy/hooks/dynamic_rate_limiter_v3.py:409-425
async def _check_rate_limits(self, ...):
    """
    Phase 1: Read-only check of ALL limits (no increments)
    Phase 2: Decide which limits to enforce based on saturation
    Phase 3: Increment ALL counters atomically (model + priority)
    """
```

**详细流程**：

```
┌──────────────────────────────────────────────────────────────────────┐
│                      Phase 1: 只读检查                                  │
├──────────────────────────────────────────────────────────────────────┤
│  调用: self.v3_limiter.should_rate_limit(read_only=True)              │
│  行为: 读取所有计数器值，但不进行任何增量                                │
│  目的: 防止部分增量问题 (模型计数器增加但优先级检查失败)                 │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Phase 2: 饱和度决策                                │
├──────────────────────────────────────────────────────────────────────┤
│  模型级限制: ALWAYS 强制 (防止模型容量超配)                             │
│  优先级限制: 仅当 saturation >= threshold 时才强制                      │
│                                                                       │
│  示例: 100 RPM 模型，60% 优先级分配，80% 阈值                          │
│  - Saturation < 80%: 优先级可使用全部 100 RPM (仅模型限制)            │
│  - Saturation >= 80%: 优先级限制为 60 RPM (双重限制)                  │
└──────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌──────────────────────────────────────────────────────────────────────┐
│                      Phase 3: 原子增量                                  │
├──────────────────────────────────────────────────────────────────────┤
│  调用: self.v3_limiter.should_rate_limit() 或                         │
│        self.v3_limiter.async_increment_tokens_with_ttl_preservation   │
│  行为: 使用 Redis Lua 脚本原子递增所有相关计数器                        │
│  关键: 仅在请求确定允许时才执行                                         │
└──────────────────────────────────────────────────────────────────────┘
```

### 2.5 默认钩子执行顺序

钩子按 `PROXY_HOOKS` 字典顺序在 `pre_call_hook` 中遍历执行：

```python
# litellm/proxy/utils.py:484-495
def _add_proxy_hooks(self, llm_router: Optional[Router] = None):
    for hook in PROXY_HOOKS:  # 按字典顺序
        proxy_hook = get_proxy_hook(hook)
        # ... 初始化并添加到 litellm.callbacks
        litellm.logging_callback_manager.add_litellm_callback(proxy_hook_obj)
```

**执行顺序**（Python 3.7+ 字典插入顺序）：

| 顺序 | 钩子名称 | 类名 | 功能 |
|-----|---------|------|------|
| 1 | `max_budget_limiter` | `_PROXY_MaxBudgetLimiter` | 个人密钥预算检查 (非团队密钥) |
| 2 | `parallel_request_limiter` | `_PROXY_MaxParallelRequestsHandler_v3` | TPM/RPM/并行请求限制 |
| 3 | `cache_control_check` | `_PROXY_CacheControlCheck` | 缓存控制 |
| 4 | `responses_id_security` | `ResponsesIDSecurity` | 响应 ID 安全 |
| 5 | `litellm_skills` | `SkillsInjectionHook` | 技能注入 |
| 6 | `max_iterations_limiter` | `_PROXY_MaxIterationsHandler` | 最大迭代限制 |
| 7 | `max_budget_per_session_limiter` | `_PROXY_MaxBudgetPerSessionHandler` | 每会话预算限制 |

### 2.6 预算检查的双轨机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         预算检查执行顺序                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段 1: common_checks (同步)                                            │
│  ├── _team_max_budget_check()           → 团队预算                       │
│  ├── _team_multi_budget_check()          → 团队多窗口预算                │
│  ├── _organization_max_budget_check()    → 组织预算                      │
│  ├── _tag_max_budget_check()             → 标签预算                      │
│  └── _team_member_budget_check()         → 团队成员预算                  │
│                                                                         │
│  阶段 2: async_pre_call_hook (异步)                                      │
│  └── _PROXY_MaxBudgetLimiter             → 个人密钥预算 (非团队密钥)     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

**关键区分**：

```python
# litellm/proxy/hooks/max_budget_limiter.py:30-33
# Personal budget applies only to non-team requests
if user_api_key_dict.team_id is not None:
    return  # 团队密钥跳过此检查 (团队预算在 common_checks 中检查)
```

---

## 3. 状态码映射校正

### 3.1 状态码映射总表

**重要修正**：不同位置的预算超限返回不同的状态码。

| 异常场景 | 触发位置 | 异常类型 | 状态码 | 说明 |
|---------|---------|---------|--------|------|
| **团队预算超限** | `_team_max_budget_check` | `BudgetExceededError` → `ProxyException` | **400** | 通过 `auth_exception_handler` 转换 |
| **团队多窗口预算超限** | `_team_multi_budget_check` | `BudgetExceededError` → `ProxyException` | **400** | 同上 |
| **组织预算超限** | `_organization_max_budget_check` | `BudgetExceededError` → `ProxyException` | **400** | 同上 |
| **标签预算超限** | `_tag_max_budget_check` | `BudgetExceededError` → `ProxyException` | **400** | 同上 |
| **个人密钥预算超限** | `max_budget_limiter` | `HTTPException` | **429** | 直接抛出，不经过转换 |
| **速率限制超限 (TPM/RPM)** | `parallel_request_limiter_v3` | `HTTPException` | **429** | 直接抛出 |
| **并行请求超限** | `parallel_request_limiter_v3` | `HTTPException` | **429** | 直接抛出 |
| **模型容量超限** | `dynamic_rate_limiter_v3` | `HTTPException` | **429** | 直接抛出 |
| **密钥不存在** | `get_key_object` | `ProxyException` (token_not_found_in_db) | **401** | 默认 auth_error 状态码 |
| **密钥过期** | `user_api_key_auth` | `ProxyException` (expired_key) | **401** | 默认 auth_error 状态码 |
| **模型访问越权** | `can_team_access_model` | `ProxyException` (team_model_access_denied) | **401** | 默认 auth_error 状态码 |
| **路由越权** | `route_checks` | `HTTPException` | **403** | 直接抛出 |
| **团队阻塞** | `common_checks` | `Exception` | **403** | 转换为 HTTPException |

### 3.2 状态码转换流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    异常 → 状态码 转换流程                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  common_checks 中抛出:                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ BudgetExceededError                                              │   │
│  │   (团队/组织/标签/团队成员预算)                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                            │
│                              ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ auth_exception_handler._handle_authentication_error()            │   │
│  │   if isinstance(e, litellm.BudgetExceededError):                │   │
│  │       raise ProxyException(..., code=400)  ← 关键转换            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  pre_call_hook 中抛出:                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ HTTPException(status_code=429)                                   │   │
│  │   (个人预算 max_budget_limiter, 速率限制 parallel_request_limiter) │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                            │
│                              ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ auth_exception_handler._handle_authentication_error()            │   │
│  │   if isinstance(e, HTTPException):                                │   │
│  │       raise ProxyException(..., code=e.status_code)              │   │
│  │       ↳ 保留原始 429 状态码                                       │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 关键代码片段

**团队预算检查 → 400 状态码**：

```python
# litellm/proxy/auth/auth_checks.py:3424-3428
raise litellm.BudgetExceededError(
    current_cost=spend,
    max_budget=team_object.max_budget,
    message=f"Budget has been exceeded! Team={team_object.team_id} ...",
)
```

```python
# litellm/proxy/auth/auth_exception_handler.py:121-127
if isinstance(e, litellm.BudgetExceededError):
    raise ProxyException(
        message=e.message,
        type=ProxyErrorTypes.budget_exceeded,
        param=None,
        code=400,  # ← 关键：预算超限是 400，不是 429
    )
```

**个人预算检查 → 429 状态码**：

```python
# litellm/proxy/hooks/max_budget_limiter.py:49-51
if curr_spend >= max_budget:
    raise HTTPException(status_code=429, detail="Max budget limit reached.")
    # ↑ 直接抛出 HTTPException，状态码 429
```

**速率限制 → 429 状态码**：

```python
# litellm/proxy/hooks/dynamic_rate_limiter_v3.py:482-495
if descriptor_key == "model_saturation_check":
    raise HTTPException(
        status_code=429,  # ← 速率限制是 429
        detail={
            "error": f"Model capacity reached for {model}. ...",
        },
        headers={"retry-after": str(self.v3_limiter.window_size)},
    )
```

### 3.4 状态码设计意图

| 状态码 | 场景 | 语义 |
|-------|------|------|
| **400** | 预算超限 (团队/组织/标签) | 资源配额耗尽，属于"业务错误" |
| **401** | 认证失败 (密钥不存在/过期/格式错误) | 认证问题 |
| **403** | 授权失败 (路由越权/团队阻塞) | 权限问题 |
| **429** | 速率限制/并行请求/个人预算 | 请求过于频繁，可重试 |

---

## 4. 跳过通用检查的场景及其影响

### 4.1 _run_centralized_common_checks 的早期返回场景

`_run_centralized_common_checks` 函数中有多个 `return` 语句，会完全跳过所有 common_checks：

```python
# litellm/proxy/auth/user_api_key_auth.py:1641-1679
async def _run_centralized_common_checks(...):
    # 场景 1: Public Routes
    if route in LiteLLMRoutes.public_routes.value or ...:
        return
    
    # 场景 2: Pass-through Endpoints with auth: false
    if pass_through_endpoints is not None:
        for endpoint in pass_through_endpoints:
            if endpoint.get("path", "") == route and endpoint.get("auth") is not True:
                return
    
    # 场景 3: No-auth Dev Mode
    if master_key is None and not (enable_jwt_auth or enable_oauth2_auth or ...):
        return
    
    # 场景 4: Custom Auth without Common Checks
    if user_custom_auth is not None and not general_settings.get("custom_auth_run_common_checks", False):
        return
```

### 4.2 各场景详细分析

#### 场景 1: Public Routes (公共路由)

**触发条件**：
```python
route in LiteLLMRoutes.public_routes.value
or route_in_additonal_public_routes(current_route=route)
```

**典型路由**：
- `/health/readiness`
- `/health/live`
- `/metrics`
- 其他配置的额外公共路由

**影响**：
- ✅ 完全跳过 `common_checks`
- ❌ 无认证检查
- ❌ 无模型权限检查
- ❌ 无预算检查
- ❌ 无速率限制检查

**设计意图**：Kubernetes 健康检查、Prometheus 指标采集等无认证场景。

---

#### 场景 2: Pass-through Endpoints with `auth: false`

**触发条件**：
```python
# config.yaml 示例
general_settings:
  pass_through_endpoints:
    - path: /v1/completions
      target: http://localhost:8080
      auth: false  # ← 关键配置
```

**代码检查**：
```python
# litellm/proxy/auth/user_api_key_auth.py:1653-1661
pass_through_endpoints = general_settings.get("pass_through_endpoints", None)
if pass_through_endpoints is not None:
    for endpoint in pass_through_endpoints:
        if (
            isinstance(endpoint, dict)
            and endpoint.get("path", "") == route
            and endpoint.get("auth") is not True  # auth 不是 True
        ):
            return
```

**影响**：
- ✅ 完全跳过 `common_checks`
- ❌ 无认证检查
- ❌ 无模型权限检查
- ❌ 无预算检查
- ❌ 无速率限制检查

**风险**：配置 `auth: false` 的端点完全开放，需谨慎使用。

---

#### 场景 3: No-auth Dev Mode (无认证开发模式)

**触发条件**：
```python
master_key is None 
and not (
    general_settings.get("enable_jwt_auth", False)
    or general_settings.get("enable_oauth2_auth", False)
    or general_settings.get("enable_oauth2_proxy_auth", False)
)
```

**即**：
- `LITELLM_MASTER_KEY` 环境变量未设置
- `config.yaml` 中 `master_key` 未设置
- 且未启用 JWT/OAuth2/OAuth2 Proxy 认证

**影响**：
- ✅ 完全跳过 `common_checks`
- ❌ 无认证检查
- ❌ 无模型权限检查
- ❌ 无预算检查
- ❌ 无速率限制检查

**设计意图**：本地开发、快速原型验证，不适合生产环境。

---

#### 场景 4: Custom Auth without Common Checks

**触发条件**：
```python
user_custom_auth is not None 
and not general_settings.get("custom_auth_run_common_checks", False)
```

**配置示例**：
```yaml
# config.yaml
general_settings:
  user_custom_auth: "path/to/custom_auth.py"
  # 未设置 custom_auth_run_common_checks: true (默认 false)
```

**影响**：
- ✅ 完全跳过 `common_checks`
- ✅ 执行自定义认证逻辑
- ❌ 无 LiteLLM 内置模型权限检查
- ❌ 无 LiteLLM 内置预算检查
- ❌ 无 LiteLLM 内置速率限制检查

**设计意图**：让自定义认证完全控制请求生命周期。

---

### 4.3 部分跳过场景

上述场景是**完全跳过** `common_checks`，还有一些场景是**部分跳过**特定检查。

#### 场景 5: 零成本模型 (仅跳过预算检查)

**触发条件**：
```python
# litellm/proxy/auth/auth_checks.py:117-189
def _is_model_cost_zero(model, llm_router):
    # 条件:
    # 1. input_cost_per_token 显式设置为 0 (不是 None)
    # 2. output_cost_per_token 显式设置为 0 (不是 None)
    # 3. 且成本配置是显式的 (不是默认值)
```

**影响**：
- ❌ 仅跳过**预算检查** (`skip_budget_checks=True`)
- ✅ 仍然执行：
  - 模型权限检查
  - 路由权限检查
  - 速率限制检查
  - 其他所有检查

**关键代码**：
```python
# litellm/proxy/auth/user_api_key_auth.py:884-889
skip_budget_checks = _is_model_cost_zero(
    model=model, llm_router=llm_router
)
if skip_budget_checks:
    verbose_proxy_logger.debug(f"Model {model} has $0 cost - skipping budget checks")

# 后续调用 common_checks 时传入
await common_checks(..., skip_budget_checks=skip_budget_checks, ...)
```

---

#### 场景 6: 团队密钥的个人预算豁免

**触发条件**：
```python
# litellm/proxy/hooks/max_budget_limiter.py:30-33
if user_api_key_dict.team_id is not None:
    return  # 团队密钥跳过个人预算检查
```

**影响**：
- ❌ 仅跳过 `max_budget_limiter` 中的**个人预算检查**
- ✅ 团队预算检查在 `common_checks` 中仍然执行
- ✅ 所有其他检查仍然执行

**设计意图**：团队密钥的预算由团队/组织级别控制，而非个人密钥级别。

---

### 4.4 跳过检查场景汇总表

| 场景 | 触发条件 | 跳过范围 | 安全风险 | 推荐用途 |
|------|---------|---------|---------|---------|
| **Public Routes** | 路由在 `public_routes` 中 | 完全跳过 | 低 (仅健康检查等) | Kubernetes 探针 |
| **Pass-through auth: false** | 端点配置 `auth: false` | 完全跳过 | **高** (完全开放) | 内部服务直连 |
| **No-auth Dev Mode** | 无 master_key 且无 JWT | 完全跳过 | **极高** | 本地开发仅 |
| **Custom Auth No Checks** | `custom_auth_run_common_checks=false` | 完全跳过 | 中 (依赖自定义认证) | 高级自定义场景 |
| **零成本模型** | 模型成本显式配置为 0 | 仅预算检查 | 低 | 内部模型/免费模型 |
| **团队密钥个人豁免** | 团队密钥 (`team_id` 存在) | 仅个人预算检查 | 低 | 正常团队场景 |

---

## 5. 关键代码位置索引

### 5.1 主密钥与 PROXY_ADMIN

| 功能 | 文件 | 行号 |
|------|------|------|
| PROXY_ADMIN common_checks 说明 | `litellm/proxy/auth/user_api_key_auth.py` | 1620-1624 |
| 主密钥匹配逻辑 | `litellm/proxy/spend_tracking/spend_tracking_utils.py` | 55-66 |
| 主密钥禁用数据库存储 | `litellm/proxy/spend_tracking/spend_tracking_utils.py` | 299-331 |

### 5.2 限流钩子装配

| 功能 | 文件 | 行号 |
|------|------|------|
| 默认 PROXY_HOOKS 定义 | `litellm/proxy/hooks/__init__.py` | 22-30 |
| 遗留模式切换 | `litellm/proxy/hooks/__init__.py` | 33-34 |
| DynamicRateLimitHandlerV3 初始化 | `litellm/proxy/hooks/dynamic_rate_limiter_v3.py` | 67-75 |
| 三阶段检查说明 | `litellm/proxy/hooks/dynamic_rate_limiter_v3.py` | 409-425 |
| 钩子装配到 callbacks | `litellm/proxy/utils.py` | 484-495 |
| 个人预算团队豁免 | `litellm/proxy/hooks/max_budget_limiter.py` | 30-33 |

### 5.3 状态码映射

| 功能 | 文件 | 行号 |
|------|------|------|
| BudgetExceededError → 400 | `litellm/proxy/auth/auth_exception_handler.py` | 121-127 |
| HTTPException 状态码保留 | `litellm/proxy/auth/auth_exception_handler.py` | 128-134 |
| 个人预算 429 抛出 | `litellm/proxy/hooks/max_budget_limiter.py` | 49-51 |
| 速率限制 429 抛出 | `litellm/proxy/hooks/dynamic_rate_limiter_v3.py` | 482-495 |
| ProxyException 定义 | `litellm/proxy/_types.py` | 3478-3526 |
| ProxyErrorTypes 枚举 | `litellm/proxy/_types.py` | 3551-3654 |

### 5.4 跳过通用检查场景

| 功能 | 文件 | 行号 |
|------|------|------|
| _run_centralized_common_checks 入口 | `litellm/proxy/auth/user_api_key_auth.py` | 1604-1679 |
| Public Routes 检查 | `litellm/proxy/auth/user_api_key_auth.py` | 1641-1645 |
| Pass-through auth: false | `litellm/proxy/auth/user_api_key_auth.py` | 1653-1661 |
| No-auth Dev Mode | `litellm/proxy/auth/user_api_key_auth.py` | 1669-1674 |
| Custom Auth No Checks | `litellm/proxy/auth/user_api_key_auth.py` | 1676-1679 |
| _is_model_cost_zero 定义 | `litellm/proxy/auth/auth_checks.py` | 117-189 |
| skip_budget_checks 使用 | `litellm/proxy/auth/user_api_key_auth.py` | 884-889, 1846-1860 |

---

## 总结

本文档修正了之前分析中的几个关键点：

1. **PROXY_ADMIN 不跳过所有检查**：主密钥仍然经过 `common_checks`，仅特定路由权限检查豁免。

2. **状态码不一致**：
   - 团队/组织/标签预算超限 → **400** (通过 `auth_exception_handler` 转换)
   - 个人密钥预算超限 → **429** (直接 `HTTPException`)
   - 速率限制超限 → **429**

3. **限流钩子关系**：
   - `DynamicRateLimitHandlerV3` 是 `ParallelRequestLimiterV3` 的包装器
   - 默认 `PROXY_HOOKS` 中只有 `parallel_request_limiter` (v3)
   - 执行顺序按 `PROXY_HOOKS` 字典顺序

4. **跳过检查场景**：
   - 4 种完全跳过 `common_checks` 的场景
   - 2 种部分跳过特定检查的场景
   - 各场景有不同的安全风险和适用范围
