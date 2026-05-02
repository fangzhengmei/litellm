# LiteLLM 冷却、回退与重试机制分析报告

## 概述

LiteLLM 在 Router 层实现了三层可靠性保障机制：
1. **重试 (Retry)**: 在同一 deployment 内进行多次尝试
2. **冷却 (Cooldown)**: 将故障 deployment 暂时移出可用池
3. **回退 (Fallback)**: 切换到其他 model/group 进行尝试

本文档详细分析这三个机制的状态流转逻辑、触发条件检测、优先级协作顺序，以及跨 provider 时的状态传递。

---

## 一、重试机制 (Retry)

### 1.1 核心实现位置

| 模块 | 文件位置 | 关键函数/类 |
|------|----------|-------------|
| 重试决策 | `litellm/utils.py` | `_should_retry()`, `_calculate_retry_after()` |
| 重试执行 | `litellm/router.py` | `async_function_with_retries()` |
| 重试策略 | `litellm/router_utils/get_retry_from_policy.py` | `get_num_retries_from_retry_policy()` |

### 1.2 触发条件检测

重试机制通过 `_should_retry()` 函数判断是否应该重试：

```python
# litellm/utils.py:6715-6741
def _should_retry(status_code: int):
    """
    Retries on 408, 409, 429 and 500 errors.
    """
    # 重试请求超时
    if status_code == 408:
        return True
    
    # 重试锁超时
    if status_code == 409:
        return True
    
    # 重试限流错误
    if status_code == 429:
        return True
    
    # 重试服务端内部错误 (5xx)
    if status_code >= 500:
        return True
    
    return False
```

**不触发重试的状态码**:
- `400 Bad Request`: 客户端请求错误
- `401 Unauthorized` *: 认证错误（特殊处理，见下文）
- `403 Forbidden` *: 权限错误（特殊处理，见下文）
- `404 Not Found`: 资源不存在
- 其他 4xx 客户端错误

### 1.3 特殊异常处理

`should_retry_this_error()` 函数 (`router.py:5942-6012`) 处理特殊情况：

| 异常类型 | 处理逻辑 |
|----------|----------|
| `ContextWindowExceededError` | 若配置了 `context_window_fallbacks` 则允许重试 |
| `ContentPolicyViolationError` | 若配置了 `content_policy_fallbacks` 则允许重试 |
| `NotFoundError` | 直接抛出，不重试 |
| `RateLimitError` | 若有健康 deployment 或 fallbacks 则允许重试 |
| `AuthenticationError (401)` | 若有多个 deployment 则允许重试（切换到其他 deployment） |
| `403 Forbidden` | 若有多个 deployment 则允许重试 |

### 1.4 重试延迟策略

使用指数退避算法 (`_calculate_retry_after()`):

```python
# 核心参数 (constants.py)
INITIAL_RETRY_DELAY = 0.5      # 初始延迟 0.5s
MAX_RETRY_DELAY = 8.0          # 最大延迟 8s
JITTER = 0.75                   # 抖动因子
DEFAULT_MAX_RETRIES = 2         # 默认重试次数
```

**延迟计算逻辑**:
1. 优先使用响应头 `Retry-After` 字段（若存在且 <= 60s）
2. 否则使用指数退避: `sleep = INITIAL_RETRY_DELAY * 2^num_retries`
3. 添加随机抖动避免雪崩
4. 限制在 `[min_timeout, MAX_RETRY_DELAY]` 范围内

**即时重试优化** (`_time_to_sleep_before_retry()`):
- 若同一 model_group 内有其他健康 deployment → **立即重试 (延迟=0)**
- 若有 fallback 配置 → **立即重试 (延迟=0)**
- 单 deployment 场景 → 正常使用指数退避

### 1.5 重试策略覆盖

重试策略优先级:
1. **请求级别** (`kwargs["num_retries"]`)
2. **Deployment 级别** (异常上的 `exception.num_retries`)
3. **Retry Policy** (基于异常类型的细粒度控制)
4. **Router 全局配置** (`self.num_retries`)

```python
# router.py:5739-5775
# 1. 从异常获取 deployment 级别配置
deployment_num_retries = getattr(e, "num_retries", None)

# 2. 从 retry policy 获取
_retry_policy_retries = _get_num_retries_from_retry_policy(
    exception=original_exception,
    model_group=_model_group_for_retry_policy,
    model_group_retry_policy=model_group_retry_policy,
    retry_policy=self.retry_policy,
)
```

---

## 二、冷却机制 (Cooldown)

### 2.1 核心实现位置

| 模块 | 文件位置 | 关键函数/类 |
|------|----------|-------------|
| 缓存管理 | `litellm/router_utils/cooldown_cache.py` | `CooldownCache` 类 |
| 冷却逻辑 | `litellm/router_utils/cooldown_handlers.py` | `_is_cooldown_required()`, `_should_cooldown_deployment()`, `_set_cooldown_deployments()` |
| 健康检查 | `litellm/router.py` | `_async_get_healthy_deployments()` |

### 2.2 触发条件检测

冷却机制分为两层检测：

#### 第一层: `_is_cooldown_required()` - 是否需要冷却

```python
# cooldown_handlers.py:40-96
def _is_cooldown_required(
    litellm_router_instance, model_id, exception_status, exception_str=None
):
    # 排除 API 连接错误 (不冷却)
    ignored_strings = ["APIConnectionError"]
    if exception_str and any(ig in exception_str for ig in ignored_strings):
        return False
    
    # 4xx 客户端错误处理
    if 400 <= exception_status < 500:
        if exception_status == 429:    # 限流 → 冷却
            return True
        elif exception_status == 401:  # 认证错误 → 冷却
            return True
        elif exception_status == 408:  # 超时 → 冷却
            return True
        elif exception_status == 404:  # 未找到 → 冷却
            return True
        else:                           # 其他 4xx → 不冷却
            return False
    
    # 5xx 服务端错误 → 冷却
    return True  # exception_status >= 500
```

**需要冷却的状态码**:
- `429 Too Many Requests` - 限流
- `401 Unauthorized` - 认证失败
- `408 Request Timeout` - 请求超时
- `404 Not Found` - 资源不存在
- `5xx` - 所有服务端错误

#### 第二层: `_should_cooldown_deployment()` - 是否应该冷却

即使异常满足冷却条件，也需要检查以下条件：

```python
# cooldown_handlers.py:166-257
def _should_cooldown_deployment(
    litellm_router_instance, deployment, exception_status, original_exception
):
    # v2 逻辑（当前默认）:
    # 1. 若为 429 限流错误且不是单 deployment model group → 冷却
    # 2. 若错误率 > 阈值 (默认 50%) 且有足够请求样本 → 冷却
    # 3. 若 litellm._should_retry() 返回 False (不可重试错误) → 冷却
    
    # 错误率计算
    percent_fails = num_fails_this_minute / (num_successes + num_fails)
    
    # 条件 2: 单 deployment 全失败且有足够流量
    if (percent_fails == 1.0 
        and total_requests_this_minute >= SINGLE_DEPLOYMENT_TRAFFIC_FAILURE_THRESHOLD):
        return True
    
    # 条件 3: 错误率超过阈值 (默认 50%)
    if (percent_fails > DEFAULT_FAILURE_THRESHOLD_PERCENT 
        and total_requests_this_minute >= DEFAULT_FAILURE_THRESHOLD_MINIMUM_REQUESTS
        and not is_single_deployment_model_group):
        return True
```

**冷却阈值常量** (`constants.py`):
| 常量 | 默认值 | 说明 |
|------|--------|------|
| `DEFAULT_FAILURE_THRESHOLD_PERCENT` | 0.5 | 错误率超过 50% 触发冷却 |
| `DEFAULT_FAILURE_THRESHOLD_MINIMUM_REQUESTS` | 5 | 至少需要 5 个请求样本 |
| `SINGLE_DEPLOYMENT_TRAFFIC_FAILURE_THRESHOLD` | 1000 | 单 deployment 场景的流量阈值 |
| `DEFAULT_COOLDOWN_TIME_SECONDS` | 5 | 默认冷却时间 5 秒 |
| `DEFAULT_ALLOWED_FAILS` | 3 | v1 逻辑的允许失败次数 |

### 2.3 冷却前置检查

在执行冷却前，`_should_run_cooldown_logic()` 会先检查禁用条件：

| 条件 | 结果 |
|------|------|
| `router.disable_cooldowns = True` | 不冷却 |
| `deployment is None` | 不冷却 |
| `_is_cooldown_required() = False` | 不冷却 |
| `time_to_cooldown ≈ 0` | 不冷却 |
| deployment 是 `provider_default_deployment_ids` | 不冷却 |
| 无法找到 model_group | 不冷却 |

### 2.4 冷却状态存储

冷却状态存储在 `CooldownCache` 中，支持 Redis 或内存缓存：

```python
# cooldown_cache.py:24-29
class CooldownCacheValue(TypedDict):
    exception_received: str      # 异常信息（脱敏）
    status_code: str             # HTTP 状态码
    timestamp: float             # 冷却开始时间
    cooldown_time: float         # 冷却持续时间
```

**缓存键格式**: `deployment:{model_id}:cooldown`

**缓存过期策略**:
- 使用 TTL (Time-To-Live) 等于 cooldown_time
- 冷却时间优先级:
  1. 动态配置 (`time_to_cooldown` 参数)
  2. 默认配置 (`CooldownCache.default_cooldown_time`)

### 2.5 冷却状态影响路由选择

在获取可用 deployment 时，冷却中的 deployment 会被过滤：

```python
# router.py:6536-6564
async def _async_get_healthy_deployments(self, model, parent_otel_span):
    # 1. 获取所有 deployments
    _, _all_deployments = self._common_checks_available_deployment(model=model)
    
    # 2. 获取冷却中的 deployments
    unhealthy_deployments = await _async_get_cooldown_deployments(
        litellm_router_instance=self, 
        parent_otel_span=parent_otel_span
    )
    
    # 3. 过滤掉冷却中的 deployment
    unhealthy_set = set(unhealthy_deployments)
    healthy_deployments = [
        d for d in _all_deployments 
        if d["model_info"]["id"] not in unhealthy_set
    ]
    
    return healthy_deployments, _all_deployments
```

**关键点**: 冷却状态是 **per-deployment** 的，不是 per-model-group。

---

## 三、回退机制 (Fallback)

### 3.1 核心实现位置

| 模块 | 文件位置 | 关键函数/类 |
|------|----------|-------------|
| 简单回退 | `litellm/litellm_core_utils/fallback_utils.py` | `async_completion_with_fallbacks()` |
| Router 回退 | `litellm/router.py` | `async_function_with_fallbacks()`, `async_function_with_fallbacks_common_utils()` |
| 回退事件 | `litellm/router_utils/fallback_event_handlers.py` | `run_async_fallback()`, `get_fallback_model_group()` |

### 3.2 两种回退模式

#### 模式 1: 简单回退 (fallback_utils.py)

用于非 Router 场景的简单回退：

```python
# fallback_utils.py:14-76
async def async_completion_with_fallbacks(**kwargs):
    """
    按顺序尝试每个 fallback model
    """
    fallbacks = [original_model] + kwargs.pop("fallbacks", [])
    
    for fallback in fallbacks:
        try:
            # 设置当前 fallback 的 model 参数
            if isinstance(fallback, dict):
                model = fallback.pop("model", original_model)
                completion_kwargs.update(fallback)
            else:
                model = fallback
            
            # 调用 litellm.acompletion
            response = await litellm.acompletion(
                **completion_kwargs, model=model
            )
            if response is not None:
                return response
                
        except Exception as e:
            verbose_logger.exception(f"Fallback failed: {model}")
            continue
    
    # 所有 fallback 都失败
    raise Exception("All fallback attempts failed")
```

#### 模式 2: Router 回退 (router.py)

用于 Router 场景的复杂回退，支持多种 fallback 类型。

### 3.3 回退类型

Router 支持四种回退类型：

| 类型 | 配置参数 | 触发条件 |
|------|----------|----------|
| **普通回退** | `fallbacks` | 重试耗尽后的任意错误 |
| **上下文窗口回退** | `context_window_fallbacks` | `ContextWindowExceededError` |
| **内容策略回退** | `content_policy_fallbacks` | `ContentPolicyViolationError` |
| **优先级回退** | `order` 字段 | 同一 model_group 内按优先级顺序 |

### 3.4 回退配置格式

#### 标准格式
```python
# 格式: [{"source_model": ["target_model1", "target_model2"]}]
fallbacks = [
    {"gpt-3.5-turbo": ["claude-3-haiku", "llama-3-8b"]},
    {"gpt-4": ["gpt-4o", "claude-3-5-sonnet"]}
]
```

#### 通配符格式 (默认回退)
```python
# 所有 model 都使用这个回退列表
fallbacks = [
    {"*": ["gpt-3.5-turbo", "claude-3-haiku"]}
]
```

#### 非标准格式 (直接列表)
```python
# 直接指定 fallback 列表，跳过 model_group 匹配
fallbacks = ["claude-3-haiku", "gpt-3.5-turbo"]

# 或带参数的字典列表
fallbacks = [
    {"model": "claude-3-haiku", "max_tokens": 1024},
    {"model": "gpt-3.5-turbo"}
]
```

#### 优先级回退格式
```python
# 通过 order 字段指定优先级，数字越小优先级越高
model_list = [
    {
        "model_name": "azure-gpt-35-turbo",
        "litellm_params": {"model": "azure/deployment-eastus", "order": 1},
        "model_info": {"id": "dep-1"}
    },
    {
        "model_name": "azure-gpt-35-turbo",
        "litellm_params": {"model": "azure/deployment-westus", "order": 2},
        "model_info": {"id": "dep-2"}
    }
]
# 先尝试 order=1，失败后回退到 order=2
```

### 3.5 回退执行流程

```
async_function_with_fallbacks()
├── 调用 async_function_with_retries()
│   └── 若重试耗尽抛出异常
├── 捕获异常
└── 调用 async_function_with_fallbacks_common_utils()
    ├── 检查 disable_fallbacks
    ├── 检查是否有基于 order 的优先级回退
    │   └── 若有，先尝试同 model_group 内的高 order deployment
    ├── 检查异常类型
    │   ├── ContextWindowExceededError → 使用 context_window_fallbacks
    │   ├── ContentPolicyViolationError → 使用 content_policy_fallbacks
    │   └── 其他异常 → 使用普通 fallbacks
    ├── 查找匹配的 fallback_model_group
    │   ├── 精确匹配 ({"gpt-4": [...]})
    │   └── 通配符匹配 ({"*": [...]})
    └── 调用 run_async_fallback()
        └── 递归调用 async_function_with_fallbacks() 处理新 model_group
```

### 3.6 回退深度限制

```python
# constants.py
ROUTER_MAX_FALLBACKS = 5  # 最大回退次数
```

通过 `fallback_depth` 字段追踪回退层级，防止无限循环。

---

## 四、三者协作顺序与优先级

### 4.1 整体调用架构

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  async_function_with_fallbacks()                            │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  外层: 回退控制 (跨 model_group)                     │   │
│  │  - 管理 fallback_model_group 切换                   │   │
│  │  - 追踪 fallback_depth                               │   │
│  │  - 限制 max_fallbacks                                │   │
│  └─────────────────────────────────────────────────────┘   │
│                         │                                   │
│                         ▼                                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  async_function_with_retries()                       │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  中层: 重试控制 (同一 model_group 内)        │   │   │
│  │  │  - 管理 num_retries 计数                    │   │   │
│  │  │  - 指数退避延迟                            │   │   │
│  │  │  - 检查 healthy_deployments               │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  │                         │                           │   │
│  │                         ▼                           │   │
│  │  ┌─────────────────────────────────────────────┐   │   │
│  │  │  内层: 单请求执行                            │   │   │
│  │  │  - 调用 make_call() -> _acompletion()      │   │   │
│  │  │  - 选择具体 deployment                      │   │   │
│  │  │  - 失败时更新冷却状态                      │   │   │
│  │  └─────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 4.2 状态流转图

```
                    ┌─────────────────┐
                    │   初始请求      │
                    └────────┬────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │   选择可用 deployment        │
              │   (过滤冷却中的)            │
              └──────────────┬───────────────┘
                             │
                             ▼
              ┌──────────────────────────────┐
              │   调用 litellm.acompletion   │
              └──────────────┬───────────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
              成功 │                 │ 失败
                    │                 │
                    ▼                 ▼
            ┌───────────┐    ┌─────────────────────────┐
            │ 返回结果  │    │ 检查冷却条件            │
            │           │    │ _is_cooldown_required() │
            └───────────┘    └───────────┬─────────────┘
                                          │
                                    ┌──────┴──────┐
                                    │             │
                              需要冷却 │        不需要冷却
                                    │             │
                                    ▼             │
                            ┌───────────┐       │
                            │ 设置冷却  │       │
                            │ 状态      │       │
                            └─────┬─────┘       │
                                  │             │
                                  ▼             ▼
                            ┌─────────────────────────┐
                            │ 检查重试条件            │
                            │ should_retry_this_error()│
                            └───────────┬─────────────┘
                                        │
                                  ┌─────┴─────┐
                                  │           │
                            可重试 │      不可重试
                                  │           │
                                  ▼           ▼
                            ┌──────────┐  ┌──────────────────────┐
                            │ 还有重试 │  │ 检查 fallback 配置    │
                            │ 次数?    │  │                       │
                            └────┬─────┘  └──────────┬───────────┘
                                 │                    │
                           是 ───┴─── 否              │
                           │          │               │
                           ▼          ▼               ▼
                    ┌──────────┐ ┌──────────┐ ┌──────────────┐
                    │ 指数退避 │ │ 检查    │ │ 有 fallback?  │
                    │ 延迟重试 │ │ fallback│ └──────┬───────┘
                    └────┬─────┘ └────┬─────┘        │
                         │            │           是 ─┴─ 否
                         ▼            ▼           │       │
              ┌────────────────────┐ ┌──────────┐ │       ▼
              │ 重新选择健康的    │ │ 抛出异常 │ │  ┌────────────┐
              │ deployment 重试   │ │ 到上层  │ │  │切换到下一个│
              └────────────────────┘ └──────────┘ │  │model_group │
                                                    │  └──────┬─────┘
                                                    │         │
                                                    └─────────┘
                                                              │
                                                              ▼
                                                ┌───────────────────────────┐
                                                │ 递归调用 async_function_   │
                                                │ with_fallbacks() 处理新的 │
                                                │ model_group               │
                                                └───────────────────────────┘
```

### 4.3 详细协作流程

以一次请求的完整生命周期为例：

#### 阶段 1: 进入回退包装层

```python
# router.py:5594-5632
async def async_function_with_fallbacks(self, *args, **kwargs):
    """
    外层回退控制
    """
    try:
        # 先尝试带重试的调用
        response = await self.async_function_with_retries(*args, **kwargs)
        response = add_fallback_headers_to_response(response, attempted_fallbacks=0)
        return response
        
    except Exception as e:
        # 重试耗尽，尝试回退
        fallback_response = await self.async_function_with_fallbacks_common_utils(
            e=e,
            disable_fallbacks=disable_fallbacks,
            fallbacks=fallbacks,
            context_window_fallbacks=context_window_fallbacks,
            content_policy_fallbacks=content_policy_fallbacks,
            model_group=model_group,
            args=args,
            kwargs=kwargs,
        )
        return fallback_response
```

#### 阶段 2: 进入重试循环

```python
# router.py:5693-5888
async def async_function_with_retries(self, *args, **kwargs):
    """
    中层重试控制
    """
    try:
        # 第一次尝试
        response = await self.make_call(original_function, *args, **kwargs)
        return response
        
    except Exception as e:
        # 获取健康 deployment 列表（过滤冷却中的）
        _healthy_deployments, _all_deployments = await self._async_get_healthy_deployments(
            model=kwargs.get("model") or "",
            parent_otel_span=parent_otel_span,
        )
        
        # 检查是否应该重试
        self.should_retry_this_error(
            error=e,
            healthy_deployments=_healthy_deployments,
            all_deployments=_all_deployments,
            ...
        )
        
        # 计算重试延迟
        retry_after = self._time_to_sleep_before_retry(
            e=original_exception,
            remaining_retries=num_retries,
            num_retries=num_retries,
            healthy_deployments=_healthy_deployments,
            all_deployments=_all_deployments,
        )
        
        await asyncio.sleep(retry_after)
        
        # 重试循环
        for current_attempt in range(num_retries):
            try:
                response = await self.make_call(original_function, *args, **kwargs)
                return response
                
            except Exception as e:
                # 每次失败后重新检查健康状态
                _healthy_deployments, _ = await self._async_get_healthy_deployments(...)
                
                # 检查是否继续重试
                self.should_retry_this_error(...)
                
                # 计算下次延迟
                _timeout = self._time_to_sleep_before_retry(...)
                await asyncio.sleep(_timeout)
        
        # 重试耗尽
        raise original_exception
```

#### 阶段 3: 单次请求执行与冷却检测

```python
# 在 _acompletion 或其他地方的失败处理
# 例如 router.py:6618-6624

except Exception as e:
    # ... 日志记录 ...
    
    # 设置冷却状态
    _set_cooldown_deployments(
        litellm_router_instance=self,
        exception_status=e.status_code,
        original_exception=e,
        deployment=deployment["model_info"]["id"],
        time_to_cooldown=self.cooldown_time,
    )
    raise e
```

### 4.4 优先级总结

| 层级 | 机制 | 作用范围 | 优先级 |
|------|------|----------|--------|
| L1 | **重试 (Retry)** | 同一 model_group 内的多个 deployment 之间 | 最高 (先尝试) |
| L2 | **冷却 (Cooldown)** | 标记单个 deployment 不可用，影响路由选择 | 隐式 (被重试依赖) |
| L3 | **回退 (Fallback)** | 跨 model_group 切换 | 最低 (重试耗尽后) |

**关键设计决策**:
1. **重试优先于回退**: 同一 model_group 内的问题先通过重试解决
2. **冷却加速重试收敛**: 故障 deployment 被快速移出可用池，让重试更快找到健康实例
3. **即时重试优化**: 有健康实例时不等待，直接重试

### 4.5 决策树

```
请求失败
    │
    ├──► 检查 _is_cooldown_required()
    │       ├── 是 ──► 设置冷却状态 (CooldownCache)
    │       └── 否 ──► 不设置
    │
    └──► 检查 should_retry_this_error()
            │
            ├── 不可重试错误 (400, 404, 等)
            │       │
            │       ├── 有 fallback? ──► 回退到其他 model_group
            │       └── 无 fallback? ──► 抛出异常
            │
            └── 可重试错误 (408, 409, 429, 5xx, 401/403有多实例)
                    │
                    ├── 还有重试次数?
                    │       │
                    │       ├── 是 ──► 检查 healthy_deployments
                    │       │               │
                    │       │               ├── 有健康实例 ──► 延迟=0，立即重试
                    │       │               └── 无健康实例 ──► 指数退避延迟后重试
                    │       │
                    │       └── 否 ──► 重试耗尽
                    │                      │
                    │                      ├── 有 fallback? ──► 回退
                    │                      └── 无 fallback? ──► 抛出异常
                    │
                    └── 特别: 有 fallback 配置
                            │
                            └── 延迟=0，立即重试 (可能选择其他 deployment 或触发回退)
```

---

## 五、跨 Provider 状态传递

### 5.1 状态隔离设计

LiteLLM 的状态管理采用 **deployment-centric** 设计：

```
Model Group: "gpt-35-turbo"
├── Deployment A: openai/gpt-3.5-turbo
│   ├── id: "dep-001"
│   ├── provider: openai
│   └── 状态: 独立的 cooldown, success/failure 计数
│
├── Deployment B: azure/gpt-35-turbo-eastus
│   ├── id: "dep-002"
│   ├── provider: azure
│   └── 状态: 独立的 cooldown, success/failure 计数
│
└── Deployment C: azure/gpt-35-turbo-westus
    ├── id: "dep-003"
    ├── provider: azure
    └── 状态: 独立的 cooldown, success/failure 计数
```

**关键点**:
- 冷却状态以 `deployment["model_info"]["id"]` 为键
- 不同 provider 的 deployment 状态完全隔离
- 一个 provider 的故障不会影响另一个 provider 的状态

### 5.2 状态如何影响路由选择

当选择 deployment 时，`async_get_available_deployment()` 会：

1. **获取 model_group 下所有 deployment**
2. **过滤掉冷却中的 deployment** (通过 `_async_get_healthy_deployments`)
3. **应用路由策略** (simple-shuffle, least-busy, 等)

**效果**: 若 OpenAI 的 deployment 被冷却，路由会自动选择 Azure 的 deployment。

### 5.3 跨 Provider 失败传递

当一个 deployment 失败时，状态更新流程：

```
失败请求 (Deployment A: OpenAI)
         │
         ▼
┌────────────────────┐
│ 1. 调用 _set_cooldown_deployments() │
│    - 检查 _is_cooldown_required()  │
│    - 检查 _should_cooldown_deployment() │
└──────────┬─────────┘
           │
           ▼ (需要冷却)
┌────────────────────┐
│ 2. CooldownCache.add_deployment_to_cooldown() │
│    - 键: "deployment:dep-001:cooldown"        │
│    - 值: {exception, status_code, timestamp, cooldown_time} │
│    - TTL = cooldown_time                       │
└──────────┬─────────┘
           │
           ▼
┌────────────────────┐
│ 3. 下次请求路由时   │
│    - _async_get_healthy_deployments() 读取缓存 │
│    - dep-001 被标记为 unhealthy               │
│    - 自动选择 dep-002 (Azure) 或 dep-003      │
└────────────────────┘
```

### 5.4 成功/失败计数的作用

冷却机制还依赖于 per-minute 的成功/失败计数：

```python
# cooldown_handlers.py:202-214
# 获取当前分钟的统计
num_successes_this_minute = get_deployment_successes_for_current_minute(
    litellm_router_instance=self, deployment_id=deployment
)
num_fails_this_minute = get_deployment_failures_for_current_minute(
    litellm_router_instance=self, deployment_id=deployment
)

# 计算错误率
percent_fails = num_fails_this_minute / (num_successes + num_fails)
```

**计数存储位置**:
- 使用 `router.failed_calls` (InMemoryCache)
- 使用 `router.success_calls` (通过 `increment_deployment_successes_for_current_minute`)
- TTL = 60 秒 (每分钟重置)

**跨 Provider 影响**: 这些计数同样是 per-deployment 的，一个 provider 的高错误率不会影响另一个 provider 的冷却判断。

### 5.5 Retry Policy 的跨 Provider 配置

Retry Policy 可以基于 model_group 配置，实现跨 provider 的统一策略：

```python
from litellm import Router
from litellm.types.router import RetryPolicy

# 定义不同异常的重试策略
retry_policy = RetryPolicy(
    TimeoutError=5,        # 超时重试 5 次
    RateLimitError=3,      # 限流重试 3 次
    AuthenticationError=0, # 认证错误不重试
)

# 按 model_group 配置不同策略
model_group_retry_policy = {
    "gpt-35-turbo": RetryPolicy(TimeoutError=10),  # 这个 model_group 更激进
    "claude-3-haiku": RetryPolicy(TimeoutError=2),  # 这个更保守
}

router = Router(
    model_list=[...],  # 包含 openai, azure, anthropic 等多 provider
    retry_policy=retry_policy,
    model_group_retry_policy=model_group_retry_policy,
)
```

**关键点**:
- Retry Policy 是 **per-model-group** 或 **per-exception-type** 的
- 同一 model_group 下的不同 provider 共享相同的 retry 策略
- 这是**唯一**跨 provider 共享的配置

### 5.6 Fallback 的跨 Provider 切换

Fallback 是实现跨 provider 故障转移的主要机制：

```python
model_list = [
    # OpenAI deployments
    {
        "model_name": "gpt-35-turbo",
        "litellm_params": {"model": "gpt-3.5-turbo", "api_key": "os.environ/OPENAI_API_KEY"},
        "model_info": {"id": "openai-gpt35"}
    },
    # Azure deployments (same model_group)
    {
        "model_name": "gpt-35-turbo",
        "litellm_params": {"model": "azure/gpt-35-turbo", "api_key": "os.environ/AZURE_API_KEY"},
        "model_info": {"id": "azure-gpt35"}
    },
    # Anthropic (different model_group for fallback)
    {
        "model_name": "claude-3-haiku",
        "litellm_params": {"model": "claude-3-haiku-20240307", "api_key": "os.environ/ANTHROPIC_API_KEY"},
        "model_info": {"id": "anthropic-haiku"}
    }
]

fallbacks = [
    # 同一 model_group 内的跨 provider 重试由 retry + cooldown 自动处理
    # 跨 model_group 的 fallback 需要显式配置
    {"gpt-35-turbo": ["claude-3-haiku"]}
]

router = Router(
    model_list=model_list,
    fallbacks=fallbacks,
    num_retries=2,
)
```

**跨 Provider 切换流程**:

```
请求 model="gpt-35-turbo"
         │
         ▼
┌─────────────────────────────┐
│ 1. 选择可用 deployment      │
│    - 候选: openai-gpt35, azure-gpt35 │
│    - 假设都健康              │
│    - 选择: openai-gpt35     │
└──────────┬──────────────────┘
           │
           ▼ (失败: 429 RateLimit)
┌─────────────────────────────┐
│ 2. 检查冷却条件             │
│    - 429 需要冷却           │
│    - 设置 openai-gpt35 冷却 │
└──────────┬──────────────────┘
           │
           ▼ (检查重试)
┌─────────────────────────────┐
│ 3. 检查 healthy_deployments │
│    - openai-gpt35: 冷却中   │
│    - azure-gpt35: 健康      │
│    - 有健康实例!             │
└──────────┬──────────────────┘
           │
           ▼ (重试延迟=0)
┌─────────────────────────────┐
│ 4. 立即重试，选择 azure-gpt35│
│    - 跨 provider (OpenAI → Azure) │
│    - 同一 model_group 内     │
└──────────┬──────────────────┘
           │
           ▼ (假设也失败，重试耗尽)
┌─────────────────────────────┐
│ 5. 检查 fallback             │
│    - gpt-35-turbo → claude-3-haiku │
│    - 跨 model_group          │
│    - 跨 provider (Azure → Anthropic) │
└──────────┬──────────────────┘
           │
           ▼
┌─────────────────────────────┐
│ 6. 递归调用，切换到          │
│    model="claude-3-haiku"   │
└─────────────────────────────┘
```

### 5.7 状态传递总结

| 状态类型 | 作用范围 | 跨 Provider 共享 | 数据存储 |
|----------|----------|------------------|----------|
| **Cooldown 状态** | Per-deployment | ❌ 隔离 | CooldownCache (Redis/InMemory) |
| **Success 计数** | Per-deployment | ❌ 隔离 | InMemoryCache (TTL=60s) |
| **Failure 计数** | Per-deployment | ❌ 隔离 | InMemoryCache (TTL=60s) |
| **Retry Policy** | Per-model-group / Per-exception | ✅ 共享 | Router 实例属性 |
| **Fallback 配置** | Per-model-group | ✅ 共享 | Router 实例属性 |

**设计哲学**:
1. **故障隔离优先**: 一个 provider 的故障状态不污染其他 provider
2. **路由层透明切换**: 通过冷却 + 重试机制，同一 model_group 内的跨 provider 切换对用户透明
3. **显式回退配置**: 跨 model_group 的切换需要显式配置，避免意外行为

---

## 六、关键常量与配置参考

### 6.1 核心常量 (constants.py)

```python
# 重试相关
DEFAULT_MAX_RETRIES = 2                    # 默认重试次数
INITIAL_RETRY_DELAY = 0.5                  # 初始退避延迟 (秒)
MAX_RETRY_DELAY = 8.0                      # 最大退避延迟 (秒)
JITTER = 0.75                               # 抖动因子

# 冷却相关
DEFAULT_COOLDOWN_TIME_SECONDS = 5          # 默认冷却时间 (秒)
DEFAULT_FAILURE_THRESHOLD_PERCENT = 0.5    # 错误率阈值 (50%)
DEFAULT_FAILURE_THRESHOLD_MINIMUM_REQUESTS = 5  # 最小请求样本
SINGLE_DEPLOYMENT_TRAFFIC_FAILURE_THRESHOLD = 1000  # 单 deployment 流量阈值
DEFAULT_ALLOWED_FAILS = 3                  # v1 逻辑的允许失败次数

# 回退相关
ROUTER_MAX_FALLBACKS = 5                   # 最大回退深度
```

### 6.2 Router 初始化参数

```python
Router(
    # 重试
    num_retries=2,                           # 全局重试次数
    retry_after=0,                           # 最小重试延迟
    retry_policy=None,                       # 细粒度重试策略
    model_group_retry_policy={},             # 按 model_group 的重试策略
    
    # 冷却
    allowed_fails=None,                      # v1 逻辑的允许失败次数
    allowed_fails_policy=None,               # 按异常类型的冷却策略
    cooldown_time=5,                          # 冷却时间
    disable_cooldowns=False,                  # 禁用冷却
    
    # 回退
    max_fallbacks=5,                          # 最大回退次数
    fallbacks=[],                             # 普通回退配置
    context_window_fallbacks=[],              # 上下文窗口回退
    content_policy_fallbacks=[],              # 内容策略回退
    default_fallbacks=None,                   # 默认回退
)
```

### 6.3 部署级别配置

```python
model_list = [
    {
        "model_name": "gpt-35-turbo",
        "litellm_params": {
            "model": "azure/gpt-35-turbo",
            "num_retries": 5,                  # deployment 级别的重试次数
            "order": 1,                        # 优先级 (用于同 group 内回退)
            "cooldown_time": 10,               # deployment 级别的冷却时间
        },
        "model_info": {
            "id": "azure-gpt35-eastus",        # 唯一标识，用于冷却状态键
        }
    }
]
```

---

## 七、典型场景分析

### 场景 1: 单实例限流 (429)

**配置**:
- 单 deployment: `openai/gpt-3.5-turbo`
- `num_retries=2`
- 无 fallback

**流程**:
1. 第一次请求 → 429 RateLimitError
2. 检查 `_should_retry(429)` → True
3. 检查 `_is_cooldown_required(429)` → True
4. 设置 deployment 冷却状态
5. 检查 `healthy_deployments` → 空 (单实例且已冷却)
6. 计算重试延迟: 指数退避 (0.5s → 1s → 2s)
7. 重试 2 次后耗尽
8. 无 fallback，抛出异常

**结果**: 请求失败，deployment 进入冷却 5 秒。

### 场景 2: 多实例跨 Provider 限流

**配置**:
- model_group: `gpt-35-turbo`
  - dep-001: `openai/gpt-3.5-turbo` (OpenAI)
  - dep-002: `azure/gpt-35-turbo` (Azure)
- `num_retries=2`

**流程**:
1. 选择 dep-001 → 429 RateLimitError
2. 设置 dep-001 冷却
3. 检查 `healthy_deployments` → [dep-002] 健康
4. 重试延迟 = 0 (有健康实例)
5. 重试选择 dep-002 → 成功
6. 返回结果

**结果**: 透明切换到 Azure，用户无感知。

### 场景 3: 认证错误 (401) + 回退

**配置**:
- model_group A: `gpt-4` (单实例，API Key 过期)
- model_group B: `claude-3-5-sonnet`
- `fallbacks=[{"gpt-4": ["claude-3-5-sonnet"]}]`
- `num_retries=2`

**流程**:
1. 选择 gpt-4 deployment → 401 AuthenticationError
2. 检查 `_should_retry(401)` → False (utils.py)
3. 检查 `should_retry_this_error()` → 单实例，不重试
4. 设置冷却状态
5. 重试耗尽 (立即)
6. 检查 fallback: `gpt-4` → `claude-3-5-sonnet`
7. 递归调用，切换到 claude-3-5-sonnet
8. 成功

**结果**: 自动回退到 Claude，gpt-4 deployment 进入冷却。

### 场景 4: 错误率阈值触发冷却

**配置**:
- model_group: `gpt-35-turbo` (2 个 deployments)
- `DEFAULT_FAILURE_THRESHOLD_PERCENT=0.5`
- 最近 1 分钟:
  - dep-001: 10 次请求，6 次失败 (60% 错误率)
  - dep-002: 10 次请求，1 次失败 (10% 错误率)

**流程**:
1. 新请求选择 dep-001 → 又一次失败
2. 检查 `_should_cooldown_deployment()`:
   - num_fails = 7, num_successes = 4
   - percent_fails = 7/11 ≈ 63.6% > 50%
   - total_requests = 11 >= 5 (最小样本)
   - 非单 deployment model_group
3. 设置 dep-001 冷却
4. 重试时 `healthy_deployments` 只有 dep-002
5. 后续请求自动选择 dep-002

**结果**: dep-001 因高错误率进入冷却，流量自动切到 dep-002。

---

## 八、代码位置索引

### 8.1 重试机制

| 功能 | 文件 | 行号 |
|------|------|------|
| 重试决策逻辑 | `litellm/utils.py` | 6715-6741 |
| 重试延迟计算 | `litellm/utils.py` | 6778-6801 |
| Router 重试执行 | `litellm/router.py` | 5693-5888 |
| 重试错误检查 | `litellm/router.py` | 5942-6012 |
| 重试前延迟计算 | `litellm/router.py` | 6060-6107 |
| 重试策略解析 | `litellm/router_utils/get_retry_from_policy.py` | - |

### 8.2 冷却机制

| 功能 | 文件 | 行号 |
|------|------|------|
| 冷却缓存类 | `litellm/router_utils/cooldown_cache.py` | 31-192 |
| 冷却必要性检查 | `litellm/router_utils/cooldown_handlers.py` | 40-96 |
| 冷却前置条件 | `litellm/router_utils/cooldown_handlers.py` | 98-163 |
| 冷却决策逻辑 | `litellm/router_utils/cooldown_handlers.py` | 166-258 |
| 设置冷却状态 | `litellm/router_utils/cooldown_handlers.py` | 260-320 |
| 获取冷却列表 | `litellm/router_utils/cooldown_handlers.py` | 323-395 |
| 健康部署检查 | `litellm/router.py` | 6536-6564 |

### 8.3 回退机制

| 功能 | 文件 | 行号 |
|------|------|------|
| 简单回退实现 | `litellm/litellm_core_utils/fallback_utils.py` | 14-76 |
| Router 回退入口 | `litellm/router.py` | 5594-5645 |
| Router 回退核心逻辑 | `litellm/router.py` | 5334-5591 |
| 回退事件处理 | `litellm/router_utils/fallback_event_handlers.py` | - |
| 获取回退 model_group | `litellm/router.py` | 6022-6058 |

### 8.4 核心常量

| 常量组 | 文件 | 行号 |
|--------|------|------|
| 重试/冷却/回退常量 | `litellm/constants.py` | 32-44, 363-366 |

---

## 九、总结

LiteLLM 的三层可靠性机制通过精心设计的协作顺序实现高可用：

### 核心原则

1. **由内向外，逐层扩散**:
   - 内层 (Retry): 快速尝试，最小延迟
   - 中层 (Cooldown): 状态标记，避免重复失败
   - 外层 (Fallback): 模式切换，最终保障

2. **状态隔离，透明切换**:
   - 冷却状态 per-deployment，故障隔离
   - 同一 model_group 内的跨 provider 切换对用户透明
   - 跨 model_group 切换需要显式配置

3. **智能决策，避免雪崩**:
   - 有健康实例时即时重试 (延迟=0)
   - 单实例时指数退避 + 抖动
   - 错误率阈值防止个别失败触发连锁反应

### 状态流转关键路径

```
请求
  │
  ▼
┌─────────────┐
│ 选择健康实例 │ ◄─────── 冷却状态影响选择
│ (过滤冷却)  │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  执行请求   │
└──────┬──────┘
       │
   ┌───┴───┐
   │       │
 成功    失败
   │       │
   │       ▼
   │  ┌─────────┐
   │  │检查冷却 │─────是────► 设置冷却状态
   │  │条件     │              (CooldownCache)
   │  └────┬────┘
   │       │
   │       ▼
   │  ┌─────────┐
   │  │检查重试 │─────可重试┐
   │  │条件     │           │
   │  └────┬────┘           │
   │       │                │
   │    不可重试            │
   │       │                │
   │       ▼                │
   │  ┌─────────┐           │
   │  │检查     │           │
   │  │fallback │◄──────────┘
   │  └────┬────┘
   │       │
   │   ┌───┴───┐
   │   │       │
   │  有      无
   │   │       │
   │   ▼       ▼
   │ 回退    抛出
   │         异常
   │
   ▼
 返回结果
```

这份分析报告涵盖了 LiteLLM Router 中重试、冷却、回退三个机制的完整实现逻辑、状态流转、协作顺序以及跨 provider 状态传递。理解这些机制对于配置高可用的 LLM 网关至关重要。
