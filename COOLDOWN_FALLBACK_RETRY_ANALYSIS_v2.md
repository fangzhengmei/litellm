# LiteLLM 冷却、回退与重试机制分析报告（修订版）

## 核心修正说明

**本次修订的关键发现**：

```
优先级顺序（从高到低）：

1. should_retry_this_error()  ◄─── 最高优先级！
2. num_retries
3. fallback
```

**原分析错误**：之前认为 `num_retries` 优先级高于 `should_retry_this_error()`，实际上正好相反。

**最关键的分流逻辑**：
- 只有当 `should_retry_this_error()` **返回 True** 时，才会检查 `num_retries` 并进入重试循环
- 如果 `should_retry_this_error()` **抛出异常**，无论 `num_retries` 是多少，都会直接跳过重试

---

## 一、整体架构

### 1.1 三层包装结构

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  async_function_with_fallbacks()  ─────── 外层：回退控制   │
│  - 跨 model_group 切换                                          │
│  - 捕获内层所有异常                                             │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  async_function_with_retries()   ─────── 中层：重试控制     │
│  - 调用 should_retry_this_error() 【最高优先级】             │
│  - 仅当返回 True 时，才检查 num_retries                       │
│  - 指数退避延迟                                                │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  make_call() → _acompletion()  ─────── 内层：单次请求       │
│  - 选择具体 deployment                                         │
│  - 失败时更新冷却状态                                          │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 关键代码位置

| 功能 | 文件 | 行号 |
|------|------|------|
| 外层回退入口 | `router.py` | 5594-5645 |
| 中层重试控制 | `router.py` | 5693-5888 |
| **最高优先级：重试决策** | `router.py` | **5942-6012** |
| 重试延迟计算 | `router.py` | 6060-6107 |
| 冷却必要性检查 | `cooldown_handlers.py` | 40-96 |

---

## 二、最高优先级：should_retry_this_error()

这是**整个可靠性机制的核心分流点**，理解它是理解所有行为的关键。

### 2.1 完整逻辑流程

```python
# router.py:5942-6012
def should_retry_this_error(
    self,
    error,
    healthy_deployments,      # 当前健康的 deployment 列表
    all_deployments,          # 该 model_group 所有 deployment
    context_window_fallbacks, # 上下文窗口回退配置
    content_policy_fallbacks, # 内容策略回退配置
    regular_fallbacks,        # 普通回退配置
):
    """
    返回 True 表示允许重试
    抛出异常表示禁止重试，让外层处理
    """
    _num_healthy = len(healthy_deployments) if healthy_deployments else 0
    _num_all = len(all_deployments) if all_deployments else 0

    # ═══════════════════════════════════════════════════════════
    # 第一层：特殊错误类型 + 有对应 fallback → 直接抛异常
    # ═══════════════════════════════════════════════════════════
    
    # 1. ContextWindowExceededError + 有 context_window_fallbacks
    if isinstance(error, ContextWindowExceededError) and context_window_fallbacks:
        raise error  # 直接走 context_window_fallback
    
    # 2. ContentPolicyViolationError + 有 content_policy_fallbacks
    if isinstance(error, ContentPolicyViolationError) and content_policy_fallbacks:
        raise error  # 直接走 content_policy_fallback

    # ═══════════════════════════════════════════════════════════
    # 第二层：状态码检查（不可重试的状态码）
    # ═══════════════════════════════════════════════════════════
    
    status_code = getattr(error, "status_code", None)
    if status_code is not None and not litellm._should_retry(status_code):
        # _should_retry() 返回 False 的情况：400, 401, 403, 404, 等
        # 但 401/403 是例外，可能允许重试
        if status_code not in (401, 403):
            raise error  # 其他 4xx 直接抛异常

    # ═══════════════════════════════════════════════════════════
    # 第三层：NotFoundError 直接抛出
    # ═══════════════════════════════════════════════════════════
    
    if isinstance(error, litellm.NotFoundError):
        raise error

    # ═══════════════════════════════════════════════════════════
    # 第四层：【关键】RateLimitError 的特殊分流
    # ═══════════════════════════════════════════════════════════
    
    if isinstance(error, openai.RateLimitError):
        # ─────────────────────────────────────────────────────
        # 限流 + 无健康实例 + 有 fallback → 直接走 fallback！
        # ─────────────────────────────────────────────────────
        if (
            _num_healthy <= 0              # 无健康实例
            and regular_fallbacks is not None  # 有 fallback 配置
            and len(regular_fallbacks) > 0     # 非空列表
        ):
            raise error  # 【关键分流】直接抛异常，跳过重试！

    # ═══════════════════════════════════════════════════════════
    # 第五层：AuthenticationError 的特殊处理
    # ═══════════════════════════════════════════════════════════
    
    if isinstance(error, openai.AuthenticationError):
        # 只有一个 deployment 时才禁止重试
        if _num_all <= 1:
            raise error
        # 多个 deployments 时允许重试（切换到其他 deployment）

    # ═══════════════════════════════════════════════════════════
    # 第六层：【最终检查】无健康实例 → 禁止重试
    # ═══════════════════════════════════════════════════════════
    
    if _num_healthy <= 0:
        raise error  # 无健康实例，无法重试

    # ═══════════════════════════════════════════════════════════
    # 第七层：允许重试
    # ═══════════════════════════════════════════════════════════
    
    return True  # 只有到达这里，才会进入重试循环
```

### 2.2 决策表

| 条件 | 结果 |
|------|------|
| ContextWindowExceededError + 有 context_window_fallbacks | **RAISE** |
| ContentPolicyViolationError + 有 content_policy_fallbacks | **RAISE** |
| 状态码 400/404/406...（非 401/403）且 _should_retry=False | **RAISE** |
| NotFoundError | **RAISE** |
| RateLimitError + 无健康实例 + 有 fallback | **RAISE** ◄──── 关键！ |
| AuthenticationError + 只有一个 deployment | **RAISE** |
| 无健康实例（兜底检查） | **RAISE** |
| **其他所有情况** | **Return True → 进入重试循环** |

---

## 三、限流场景的分流逻辑（重点修正）

### 3.1 完整分流图

```
请求失败：RateLimitError (429)
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│  Step 1: 设置冷却状态                                     │
│  - 调用 _is_cooldown_required(429) → True               │
│  - 调用 _should_cooldown_deployment()                    │
│  - 将当前 deployment 加入冷却缓存                         │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│  Step 2: 获取 healthy_deployments                        │
│  - 调用 _async_get_healthy_deployments()                 │
│  - 过滤掉冷却中的 deployments                             │
│  - 结果可能是空列表或有剩余实例                           │
└───────────────────────────┬─────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────┐
│  Step 3: 【最高优先级】调用 should_retry_this_error()    │
│  - 传入 healthy_deployments, regular_fallbacks          │
└───────────────────────────┬─────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            │                               │
            ▼                               ▼
┌───────────────────────┐       ┌───────────────────────┐
│ 有健康实例？          │       │ 无健康实例？          │
│ (healthy_deployments  │       │ (healthy_deployments  │
│  非空)                 │       │  为空)                 │
└───────────┬───────────┘       └───────────┬───────────┘
            │                               │
            ▼                               ▼
┌───────────────────────┐       ┌──────────────────────────────────┐
│ should_retry_this_error│       │ 检查：有 regular_fallbacks？    │
│ 返回 True              │       │ (非 None 且 len > 0)            │
└───────────┬───────────┘       └──────────────┬───────────────────┘
            │                                    │
            ▼                    ┌───────────────┴───────────────┐
    进入重试循环                 │                               │
            │                    ▼                               ▼
            │            ┌───────────────┐           ┌───────────────────┐
            │            │ 有 fallback？  │           │ 无 fallback？      │
            │            └───────┬───────┘           └─────────┬─────────┘
            │                    │                               │
            ▼                    ▼                               ▼
    ┌───────────────┐    ┌───────────────────┐           ┌───────────────────┐
    │检查 num_retries│    │ should_retry_this │           │ should_retry_this │
    │               │    │ _error 抛出异常！  │           │ _error 抛出异常！  │
    │> 0 ?          │    │                   │           │                   │
    └───────┬───────┘    └─────────┬─────────┘           └─────────┬─────────┘
            │                        │                               │
            ▼                        ▼                               ▼
    ┌───────────────┐        直接走 fallback              直接抛出异常
    │ 是：重试循环  │        跳过重试！                    (num_retries 无效)
    │               │                                       │
    │ 否：抛出异常  │                                       │
    └───────────────┘                                       │
                                                            │
                                                            ▼
                                                    ┌───────────────────┐
                                                    │ 外层捕获异常，    │
                                                    │ 无 fallback 可用  │
                                                    │ 最终请求失败      │
                                                    └───────────────────┘
```

### 3.2 四种限流场景详解

#### 场景 A：限流 + 有健康实例（其他 deployment 正常）

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/eastus"}, "model_info": {"id": "dep-1"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/westus"}, "model_info": {"id": "dep-2"}},
]
router = Router(model_list=model_list, num_retries=2)
```

**执行流程**：
```
1. 选择 dep-1 → 429 RateLimitError
2. 设置 dep-1 冷却
3. 获取 healthy_deployments → [dep-2] (非空！)
4. 调用 should_retry_this_error():
   - 是 RateLimitError？是
   - _num_healthy <= 0？否 (dep-2 健康)
   - 不抛异常，继续
   - 最后检查 _num_healthy <= 0？否
   - 返回 True ✓
5. 检查 num_retries > 0？是 (num_retries=2)
6. 计算重试延迟 _time_to_sleep_before_retry():
   - 有健康实例 → 延迟 = 0 (即时重试)
7. sleep(0)
8. 进入重试循环：
   - 重新选择 deployment → dep-2
   - 调用成功
9. 返回结果
```

**结果**：透明切换到 dep-2，用户无感知。

---

#### 场景 B：限流 + 无健康实例 + **有 fallback**

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/eastus"}, "model_info": {"id": "dep-1"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/westus"}, "model_info": {"id": "dep-2"}},
    {"model_name": "claude", "litellm_params": {"model": "claude-3-haiku"}, "model_info": {"id": "dep-3"}},
]
router = Router(
    model_list=model_list, 
    num_retries=2,
    fallbacks=[{"gpt-35": ["claude"]}]  # 有 fallback 配置！
)
```

**假设**：dep-1 和 dep-2 都被冷却了（都返回 429）

**执行流程**：
```
1. 选择 dep-1 → 429 RateLimitError
2. 设置 dep-1 冷却
3. 选择 dep-2 (重试) → 又 429 RateLimitError
4. 设置 dep-2 冷却
5. 获取 healthy_deployments → [] (空！)
6. 调用 should_retry_this_error():
   - 是 RateLimitError？是
   - 检查条件：
     * _num_healthy <= 0？是 (0 <= 0) ✓
     * regular_fallbacks is not None？是 (fallbacks 已配置) ✓
     * len(regular_fallbacks) > 0？是 (len=1 > 0) ✓
   - 【抛出异常！】←───── 关键分流点
7. 异常直接传播，不检查 num_retries！
8. 外层 async_function_with_fallbacks 捕获异常
9. 查找 fallback_model_group → "claude"
10. 递归调用，切换到 model="claude"
11. 选择 dep-3 → 成功
12. 返回结果
```

**关键发现**：
- `num_retries=2` 完全被忽略！
- 因为 `should_retry_this_error()` 直接抛出了异常
- 直接走 fallback，跳过重试循环

---

#### 场景 C：限流 + 无健康实例 + **无 fallback**

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/eastus"}, "model_info": {"id": "dep-1"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/westus"}, "model_info": {"id": "dep-2"}},
]
router = Router(
    model_list=model_list, 
    num_retries=2,
    fallbacks=None  # 无 fallback 配置！
)
```

**假设**：dep-1 和 dep-2 都被冷却了

**执行流程**：
```
1. 选择 dep-1 → 429 RateLimitError
2. 设置 dep-1 冷却
3. 选择 dep-2 (重试) → 又 429 RateLimitError
4. 设置 dep-2 冷却
5. 获取 healthy_deployments → [] (空！)
6. 调用 should_retry_this_error():
   - 是 RateLimitError？是
   - 检查条件：
     * _num_healthy <= 0？是 ✓
     * regular_fallbacks is not None？否 (fallbacks is None) ✗
   - 第5步条件不满足，不抛出
   - 继续到第7步检查：
     * _num_healthy <= 0？是 ✓
   - 【第7步抛出异常！】
7. 异常直接传播，不检查 num_retries！
8. 外层 async_function_with_fallbacks 捕获异常
9. 查找 fallback_model_group → None (无 fallback)
10. 重新抛出异常
11. 请求最终失败
```

**关键发现**：
- `num_retries=2` 同样被忽略！
- 虽然没有 fallback，但 `should_retry_this_error()` 在第7步抛出了异常
- 无健康实例时，无论是否有 fallback，都不会进入重试循环

---

#### 场景 D：单实例限流 + 无 fallback

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "dep-1"}},
]
router = Router(
    model_list=model_list, 
    num_retries=2,
    fallbacks=None
)
```

**执行流程**：
```
1. 选择 dep-1 → 429 RateLimitError
2. 设置 dep-1 冷却
3. 获取 healthy_deployments → [] (单实例被冷却)
4. 调用 should_retry_this_error():
   - 是 RateLimitError？是
   - 检查条件：
     * _num_healthy <= 0？是 ✓
     * regular_fallbacks is not None？否 ✗
   - 第7步检查：_num_healthy <= 0 → 抛出异常
5. 异常传播，不检查 num_retries
6. 无 fallback，请求失败
```

**结果**：请求失败，deployment 冷却 5 秒。

---

### 3.3 分流决策表总结

| 场景 | healthy_deployments | 有 fallback？ | should_retry_this_error | num_retries 生效？ | 最终行为 |
|------|---------------------|---------------|--------------------------|-------------------|----------|
| A | 非空 | 任意 | 返回 True | **是** | 重试循环，切换实例 |
| B | 空 | 是 | **抛出异常** | **否** | 直接走 fallback |
| C | 空 | 否 | **抛出异常** | **否** | 直接失败 |
| D | 空 | 否 | **抛出异常** | **否** | 直接失败 |

**核心修正结论**：

```
num_retries 只在以下情况生效：
└──► should_retry_this_error() 返回 True
    └──► 有健康的 deployment 可以重试
```

---

## 四、冷却机制 (Cooldown)

### 4.1 触发条件

冷却机制在**单次请求失败后**、**重试决策前**执行：

```python
# 调用链：
_acompletion() 失败
    │
    ▼
_set_cooldown_deployments()
    │
    ├──► _should_run_cooldown_logic()  ── 检查前置条件
    │
    └──► _should_cooldown_deployment()  ── 检查是否应该冷却
            │
            ├──► _is_cooldown_required()  ── 检查状态码
            │
            └──► 检查错误率阈值
```

### 4.2 _is_cooldown_required() 状态码检查

```python
# cooldown_handlers.py:40-96
def _is_cooldown_required(exception_status, exception_str=None):
    # 排除 API 连接错误
    if "APIConnectionError" in exception_str:
        return False
    
    # 4xx 客户端错误
    if 400 <= status < 500:
        if status == 429:    return True  # 限流 → 冷却
        if status == 401:    return True  # 认证 → 冷却
        if status == 408:    return True  # 超时 → 冷却
        if status == 404:    return True  # 未找到 → 冷却
        return False                    # 其他 4xx → 不冷却
    
    # 5xx 服务端错误
    return True  # 所有 5xx → 冷却
```

### 4.3 _should_cooldown_deployment() 额外检查

即使状态码满足冷却条件，还要检查：

| 条件 | 说明 |
|------|------|
| 429 限流错误 | 非单 deployment model_group 时冷却 |
| 错误率 > 50% | 且有至少 5 个请求样本 |
| 单实例 100% 失败 | 且流量 >= 1000 请求 |
| _should_retry() 返回 False | 不可重试的错误 |

### 4.4 冷却不执行的前置条件

`_should_run_cooldown_logic()` 返回 False 的情况：

```python
# cooldown_handlers.py:98-163
def _should_run_cooldown_logic():
    # 任一条件满足则不冷却
    if router.disable_cooldowns:          return False
    if deployment is None:                 return False
    if time_to_cooldown ≈ 0:               return False
    if deployment in provider_default_deployment_ids: return False
    if _is_cooldown_required() == False:   return False
    if model_group not found:              return False
    
    return True  # 所有条件都满足才执行冷却
```

### 4.5 冷却状态存储

```python
# cooldown_cache.py
class CooldownCacheValue(TypedDict):
    exception_received: str      # 异常信息（脱敏）
    status_code: str             # HTTP 状态码
    timestamp: float             # 冷却开始时间
    cooldown_time: float         # 冷却持续时间

# 缓存键格式
key = f"deployment:{model_id}:cooldown"

# TTL = cooldown_time，到期自动清除
```

### 4.6 冷却影响路由选择

```python
# router.py:6536-6564
async def _async_get_healthy_deployments(self, model, parent_otel_span):
    # 1. 获取所有 deployments
    _, _all_deployments = self._common_checks_available_deployment(model=model)
    
    # 2. 获取冷却中的 deployments
    unhealthy_deployments = await _async_get_cooldown_deployments(...)
    
    # 3. 过滤掉冷却中的
    unhealthy_set = set(unhealthy_deployments)
    healthy_deployments = [
        d for d in _all_deployments 
        if d["model_info"]["id"] not in unhealthy_set
    ]
    
    return healthy_deployments, _all_deployments
```

**关键点**：
- 冷却状态是 **per-deployment** 的，以 `model_info["id"]` 为键
- 不同 provider 的 deployment 冷却状态完全隔离
- 冷却状态直接影响 `healthy_deployments` 的计算结果

---

## 五、回退机制 (Fallback)

### 5.1 触发时机

回退机制**只在以下情况触发**：

```
async_function_with_retries() 抛出异常
    │
    ▼
async_function_with_fallbacks() 捕获异常
    │
    ▼
调用 async_function_with_fallbacks_common_utils()
    │
    ├──► 检查 disable_fallbacks
    │
    ├──► 检查异常类型，选择对应 fallback 配置
    │       ├── ContextWindowExceededError → context_window_fallbacks
    │       ├── ContentPolicyViolationError → content_policy_fallbacks
    │       └── 其他异常 → regular_fallbacks
    │
    └──► 查找匹配的 fallback_model_group
            │
            ├──► 精确匹配：{"gpt-35": ["claude"]}
            │
            ├──► 通配符匹配：{"*": ["gpt-35"]}
            │
            └──► 非标准格式：["claude", "gpt-35"] 直接使用
```

### 5.2 回退类型优先级

```
异常发生
    │
    ▼
检查异常类型
    │
    ├──► ContextWindowExceededError
    │       │
    │       ├──► 有 context_window_fallbacks ──► 使用它
    │       │
    │       └──► 无 ──► 尝试 regular_fallbacks
    │
    ├──► ContentPolicyViolationError
    │       │
    │       ├──► 有 content_policy_fallbacks ──► 使用它
    │       │
    │       └──► 无 ──► 尝试 regular_fallbacks
    │
    └──► 其他异常
            │
            └──► 使用 regular_fallbacks
```

### 5.3 回退配置格式

#### 标准格式
```python
fallbacks = [
    {"gpt-35-turbo": ["claude-3-haiku", "llama-3-8b"]},
    {"gpt-4": ["gpt-4o", "claude-3-5-sonnet"]}
]
```

#### 通配符格式
```python
fallbacks = [
    {"*": ["gpt-35-turbo", "claude-3-haiku"]}  # 所有 model 都用这个
]
```

#### 非标准格式（直接列表）
```python
fallbacks = ["claude-3-haiku", "gpt-35-turbo"]  # 跳过匹配，直接按顺序尝试
```

#### 优先级回退（同 model_group 内）
```python
model_list = [
    {
        "model_name": "azure-gpt35",
        "litellm_params": {"model": "azure/eastus", "order": 1},  # 高优先级
        "model_info": {"id": "dep-1"}
    },
    {
        "model_name": "azure-gpt35",
        "litellm_params": {"model": "azure/westus", "order": 2},  # 低优先级
        "model_info": {"id": "dep-2"}
    }
]
# 先尝试 order=1，失败后自动回退到 order=2
```

### 5.4 回退深度限制

```python
# constants.py
ROUTER_MAX_FALLBACKS = 5  # 最大回退次数，防止无限循环

# 通过 fallback_depth 追踪
input_kwargs["fallback_depth"] = 0  # 初始为 0
# 每次回退 +1，超过限制则停止
```

---

## 六、完整状态流转图（修订版）

### 6.1 单次请求的完整生命周期

```
用户请求
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│  async_function_with_fallbacks()                             │
│  ├──► 设置 num_retries, litellm_trace_id, metadata          │
│  └──► 调用 async_function_with_retries()                     │
└───────────────────────────┬─────────────────────────────────┘
                            │
                            ├──────────────────────────────┐
                            │                              │
                            ▼                              ▼ (异常)
┌─────────────────────────────────────────────────────┐   │
│  async_function_with_retries()                        │   │
│  ├──► 首次调用 make_call()                             │   │
│  │       │                                             │   │
│  │       ├──► 成功 ──► 返回结果                       │   │
│  │       │                                             │   │
│  │       └──► 失败 (Exception e)                       │   │
│  │               │                                     │   │
│  │               ▼                                     │   │
│  │       ┌─────────────────────────────────────┐      │   │
│  │       │ 1. 获取 healthy_deployments        │      │   │
│  │       │    (过滤冷却中的)                   │      │   │
│  │       └───────────────────┬─────────────────┘      │   │
│  │                           │                        │   │
│  │                           ▼                        │   │
│  │       ┌─────────────────────────────────────┐      │   │
│  │       │ 2. 【最高优先级】调用                │      │   │
│  │       │    should_retry_this_error()       │      │   │
│  │       └───────────────────┬─────────────────┘      │   │
│  │                           │                        │   │
│  │            ┌──────────────┴──────────────┐        │   │
│  │            │                             │        │   │
│  │            ▼                             ▼        │   │
│  │   ┌─────────────────┐           ┌─────────────────┐│   │
│  │   │ 返回 True       │           │ 抛出异常        ││   │
│  │   │ (允许重试)      │           │ (禁止重试)      ││   │
│  │   └────────┬────────┘           └────────┬────────┘│   │
│  │            │                             │         │   │
│  │            ▼                             │         │   │
│  │   ┌─────────────────┐                    │         │   │
│  │   │ 3. 检查         │                    │         │   │
│  │   │    num_retries  │                    │         │   │
│  │   │    > 0 ?        │                    │         │   │
│  │   └────────┬────────┘                    │         │   │
│  │            │                             │         │   │
│  │     ┌──────┴──────┐                      │         │   │
│  │     │             │                      │         │   │
│  │     ▼             ▼                      │         │   │
│  │  ┌────────┐   ┌────────┐                 │         │   │
│  │  │ 是     │   │ 否     │                 │         │   │
│  │  └───┬────┘   └───┬────┘                 │         │   │
│  │      │            │                      │         │   │
│  │      ▼            ▼                      │         │   │
│  │  ┌─────────────────────────────────┐    │         │   │
│  │  │ 4. 进入重试循环                 │    │         │   │
│  │  │    for i in range(num_retries):│    │         │   │
│  │  │      │                          │    │         │   │
│  │  │      ├──► 计算重试延迟          │    │         │   │
│  │  │      │    - 有健康实例: 延迟=0  │    │         │   │
│  │  │      │    - 无健康实例: 指数退避│    │         │   │
│  │  │      │                          │    │         │   │
│  │  │      ├──► sleep(timeout)        │    │         │   │
│  │  │      │                          │    │         │   │
│  │  │      ├──► make_call()           │    │         │   │
│  │  │      │    │                     │    │         │   │
│  │  │      │    ├──► 成功 ──► 返回   │    │         │   │
│  │  │      │    │                     │    │         │   │
│  │  │      │    └──► 失败             │    │         │   │
│  │  │      │         │                 │    │         │   │
│  │  │      │         ▼                 │    │         │   │
│  │  │      │    再次调用               │    │         │   │
│  │  │      │    should_retry_this_error│   │         │   │
│  │  │      │         │                 │    │         │   │
│  │  │      │    ┌────┴────┐            │    │         │   │
│  │  │      │    │         │            │    │         │   │
│  │  │      │   抛出      │             │    │         │   │
│  │  │      │   异常      │ 返回 True   │    │         │   │
│  │  │      │    │         │            │    │         │   │
│  │  │      │    │         ▼            │    │         │   │
│  │  │      │    │    继续循环          │    │         │   │
│  │  │      │    │                     │    │         │   │
│  │  │   ┌──┴────┴─────────────────────┘    │         │   │
│  │  │   │                                   │         │   │
│  │  └───┼───────────────────────────────────┘         │   │
│  │      │                                             │   │
│  │      ▼                                             │   │
│  │ 重试耗尽，抛出异常                                  │   │
│  │      │                                             │   │
│  └──────┼─────────────────────────────────────────────┘   │
│         │                                                   │
└─────────┼───────────────────────────────────────────────────┘
          │
          │ (所有异常都到这里)
          │
          ▼
┌─────────────────────────────────────────────────────────────┐
│  async_function_with_fallbacks() 捕获异常                    │
│                                                               │
│  调用 async_function_with_fallbacks_common_utils()           │
│  ├──► 检查 disable_fallbacks                                 │
│  ├──► 检查异常类型，选择 fallback 配置                       │
│  ├──► 查找匹配的 fallback_model_group                        │
│  │       │                                                   │
│  │       ├──► 找到 ──► 递归调用                             │
│  │       │           async_function_with_fallbacks()         │
│  │       │           切换到新的 model_group                  │
│  │       │                                                   │
│  │       └──► 未找到 ──► 重新抛出异常                       │
│  │                                                           │
│  同时设置冷却状态（在失败时已设置）                          │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 冷却状态的更新时机

冷却状态在**最内层**更新，在 `should_retry_this_error()` **之前**：

```
make_call() → _acompletion() 失败
    │
    ▼ (在 _acompletion 的 except 块中)
_set_cooldown_deployments()
    │
    ├──► 检查 _should_run_cooldown_logic()
    ├──► 检查 _should_cooldown_deployment()
    └──► 更新 CooldownCache（如果需要）
    │
    ▼
异常继续抛出
    │
    ▼
回到 async_function_with_retries() 的 except 块
    │
    ▼
【重要】此时 CooldownCache 已更新！
    │
    ▼
调用 _async_get_healthy_deployments()
    │
    ▼
返回的 healthy_deployments 已排除刚冷却的 deployment
    │
    ▼
【关键】这就是为什么一次 429 后，healthy_deployments 可能变空
    │
    ▼
调用 should_retry_this_error(healthy_deployments=..., fallbacks=...)
```

---

## 七、优先级与协作总结（修订版）

### 7.1 决策优先级金字塔

```
                    ┌───────────────────┐
                    │  1. 冷却机制      │
                    │  (Cooldown)      │
                    │                   │
                    │  最内层，最先执行  │
                    │  影响 healthy_   │
                    │  deployments     │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  2. should_retry_ │
                    │     this_error()  │
                    │                   │
                    │  【最高优先级】   │
                    │  决定是否允许重试 │
                    │  返回 True 才检查 │
                    │  num_retries      │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  3. num_retries   │
                    │                   │
                    │  仅当 should_     │
                    │  retry_this_error │
                    │  返回 True 时生效 │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │  4. 回退机制      │
                    │  (Fallback)      │
                    │                   │
                    │  最外层           │
                    │  重试耗尽或禁止时 │
                    │  才触发           │
                    └───────────────────┘
```

### 7.2 关键决策点速查表

| 决策点 | 位置 | 输入 | 输出 | 优先级 |
|--------|------|------|------|--------|
| 冷却必要性 | `_is_cooldown_required()` | 状态码、异常类型 | bool | 1 (最先) |
| 冷却决策 | `_should_cooldown_deployment()` | 错误率、实例数 | bool | 2 |
| **重试决策** | `should_retry_this_error()` | healthy_deployments, fallbacks | True / Raise | **3 (最高)** |
| 重试延迟 | `_time_to_sleep_before_retry()` | healthy_deployments | 秒数 | 4 |
| 重试次数 | `num_retries` | 配置 | 循环次数 | 5 |
| 回退决策 | `async_function_with_fallbacks_common_utils()` | 异常类型、fallback 配置 | 新 model_group | 6 (最后) |

### 7.3 配置参数优先级

同一参数可以在多处配置，优先级如下：

```
num_retries 优先级：
┌─────────────────────────────────────────────────────────┐
│ 1. exception.num_retries (从 deployment 动态设置)        │
│ 2. retry_policy (按异常类型的细粒度策略)                 │
│ 3. model_group_retry_policy (按 model_group)            │
│ 4. kwargs["num_retries"] (请求级参数)                    │
│ 5. deployment.litellm_params["num_retries"]              │
│ 6. Router.num_retries (全局配置)                         │
│ 7. DEFAULT_MAX_RETRIES = 2 (常量)                        │
└─────────────────────────────────────────────────────────┘

cooldown_time 优先级：
┌─────────────────────────────────────────────────────────┐
│ 1. 动态 time_to_cooldown 参数                            │
│ 2. Router.cooldown_time                                  │
│ 3. DEFAULT_COOLDOWN_TIME_SECONDS = 5                    │
└─────────────────────────────────────────────────────────┘
```

---

## 八、典型场景深度分析（修订版）

### 场景 1：跨 Provider 透明切换（有健康实例）

**配置**：
```python
model_list = [
    # OpenAI
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "openai-1"}},
    # Azure
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/gpt35-eastus"}, "model_info": {"id": "azure-1"}},
]
router = Router(
    model_list=model_list, 
    num_retries=2,
    fallbacks=[{"gpt-35": ["claude"]}]  # 备用，但可能用不到
)
```

**执行流程**：
```
1. 选择 openai-1 → 429 RateLimitError
2. 设置 openai-1 冷却
3. 回到 async_function_with_retries
4. 获取 healthy_deployments → [azure-1] (非空！)
5. 调用 should_retry_this_error():
   - 是 RateLimitError？是
   - _num_healthy <= 0？否
   - 返回 True
6. 检查 num_retries > 0？是 (2 > 0)
7. 计算重试延迟：
   - 有健康实例 → 延迟 = 0
8. sleep(0)
9. 重试循环，选择 azure-1
10. 调用成功
11. 返回结果
```

**关键点**：
- 透明切换到 Azure，用户无感知
- fallback 配置**没有用到**，因为重试循环内就找到了健康实例
- num_retries**生效了**，因为 `should_retry_this_error()` 返回了 True

---

### 场景 2：限流 + 所有实例冷却 + 有 fallback（关键修正）

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "openai-1"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/gpt35-eastus"}, "model_info": {"id": "azure-1"}},
    {"model_name": "claude", "litellm_params": {"model": "claude-3-haiku"}, "model_info": {"id": "anthropic-1"}},
]
router = Router(
    model_list=model_list, 
    num_retries=2,
    fallbacks=[{"gpt-35": ["claude"]}]  # 有 fallback
)
```

**假设**：openai-1 和 azure-1 都被冷却了（连续 429）

**执行流程**：
```
1. 选择 openai-1 → 429
2. 设置 openai-1 冷却
3. 重试，选择 azure-1 → 又 429
4. 设置 azure-1 冷却
5. 获取 healthy_deployments → [] (空！)
6. 调用 should_retry_this_error():
   - 是 RateLimitError？是
   - 检查：
     * _num_healthy <= 0？是 ✓
     * regular_fallbacks is not None？是 ✓
     * len(regular_fallbacks) > 0？是 ✓
   - 【抛出异常！】←───── 关键！
7. 异常传播，不检查 num_retries
8. 外层捕获异常
9. 查找 fallback → claude
10. 递归调用，切换到 model="claude"
11. 选择 anthropic-1 → 成功
12. 返回结果
```

**关键发现（原分析错误）**：
- `num_retries=2` **没有生效**！
- 因为 `should_retry_this_error()` 直接抛出了异常
- 不会进入重试循环，直接走 fallback
- 这种设计是**合理的**：既然所有实例都冷却了，重试也没用，不如直接切到其他 model_group

---

### 场景 3：认证错误 (401) + 多实例

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo", "api_key": "bad-key-1"}, "model_info": {"id": "dep-1"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo", "api_key": "good-key-2"}, "model_info": {"id": "dep-2"}},
]
router = Router(model_list=model_list, num_retries=2)
```

**执行流程**：
```
1. 选择 dep-1 → 401 AuthenticationError
2. 检查 _is_cooldown_required(401) → True (认证错误需要冷却)
3. 设置 dep-1 冷却
4. 获取 healthy_deployments → [dep-2]
5. 调用 should_retry_this_error():
   - 状态码 401，_should_retry(401) → False
   - 但 401 在例外列表中，不立即抛出
   - 是 AuthenticationError？是
   - _num_all <= 1？否 (有 2 个实例)
   - _num_healthy <= 0？否
   - 返回 True
6. 检查 num_retries > 0？是
7. 重试延迟 = 0
8. 重试，选择 dep-2
9. 调用成功 (good-key-2)
10. 返回结果
```

**关键点**：
- 401 错误在多实例场景下允许重试
- 自动切换到其他实例（可能用正确的 key）
- 单实例场景下 401 会直接抛出

---

### 场景 4：不可重试错误 (400 Bad Request)

**配置**：
```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "dep-1"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "dep-2"}},
]
router = Router(model_list=model_list, num_retries=10, fallbacks=[{"gpt-35": ["claude"]}])
```

**请求**：传递了无效参数，导致 400 Bad Request

**执行流程**：
```
1. 选择 dep-1 → 400 Bad Request
2. 检查 _is_cooldown_required(400):
   - 400 不在 [401, 404, 408, 429] 中
   - 返回 False
3. 不设置冷却
4. 获取 healthy_deployments → [dep-1, dep-2]
5. 调用 should_retry_this_error():
   - 状态码 400，_should_retry(400) → False
   - 400 不在例外列表 (401, 403) 中
   - 【抛出异常！】
6. 异常传播，不检查 num_retries
7. 外层捕获异常
8. 查找 fallback → claude
9. 切换到 claude
10. 成功（如果参数对 claude 有效）
```

**关键点**：
- 400 错误**不会重试**，也**不会冷却**
- 400 通常是客户端参数错误，重试其他实例也没用
- 但会走 fallback（如果配置了）

---

## 九、跨 Provider 状态传递

### 9.1 状态隔离设计

LiteLLM 采用 **deployment-centric** 设计，状态完全隔离：

```
Model Group: "gpt-35-turbo"
├── Deployment A: openai/gpt-3.5-turbo
│   ├── id: "openai-001"
│   ├── provider: openai
│   └── 状态存储键: "deployment:openai-001:cooldown"
│
├── Deployment B: azure/gpt35-eastus
│   ├── id: "azure-001"
│   ├── provider: azure
│   └── 状态存储键: "deployment:azure-001:cooldown"
│
└── Deployment C: azure/gpt35-westus
    ├── id: "azure-002"
    ├── provider: azure
    └── 状态存储键: "deployment:azure-002:cooldown"
```

### 9.2 状态传递的唯一方式

状态通过 **`healthy_deployments`** 间接传递：

```
请求失败
    │
    ▼
设置当前 deployment 冷却状态
    │
    ▼
_async_get_healthy_deployments() 读取所有冷却状态
    │
    ▼
过滤掉冷却中的 deployments
    │
    ▼
返回 healthy_deployments 列表
    │
    ▼
should_retry_this_error(healthy_deployments=...)
    │
    ├──► 如果列表为空 + 有 fallback → 直接走 fallback
    │
    └──► 如果列表非空 → 允许重试，切换到其他 deployment
```

### 9.3 跨 Provider 切换的两种方式

#### 方式 A：同一 model_group 内的透明切换

```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "openai"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/gpt35"}, "model_info": {"id": "azure"}},
]

# 切换流程：
# 1. 选择 openai → 失败 → 冷却
# 2. healthy_deployments = [azure]
# 3. should_retry_this_error() 返回 True
# 4. 重试，选择 azure
# 5. 用户无感知
```

#### 方式 B：跨 model_group 的显式回退

```python
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "openai"}},
    {"model_name": "claude", "litellm_params": {"model": "claude-3-haiku"}, "model_info": {"id": "anthropic"}},
]
router = Router(
    model_list=model_list,
    fallbacks=[{"gpt-35": ["claude"]}]  # 显式配置
)

# 切换流程：
# 1. 选择 openai → 失败 → 冷却
# 2. healthy_deployments = [] (单实例)
# 3. should_retry_this_error() 抛出异常 (有 fallback)
# 4. 外层捕获，查找 fallback
# 5. 递归调用，model_group 切换到 claude
# 6. 选择 anthropic
```

### 9.4 共享配置 vs 隔离状态

| 配置/状态 | 作用范围 | 跨 Provider 共享 |
|-----------|----------|------------------|
| Cooldown 状态 | per-deployment | ❌ 隔离 |
| Success/Failure 计数 | per-deployment | ❌ 隔离 |
| num_retries | 可配置多层级 | ✅ 可共享 |
| retry_policy | per-model-group / per-exception | ✅ 可共享 |
| fallbacks | per-model-group | ✅ 可共享 |
| cooldown_time | 可配置多层级 | ✅ 可共享 |

---

## 十、关键常量与配置参考

### 10.1 核心常量

```python
# constants.py

# 重试相关
DEFAULT_MAX_RETRIES = 2                    # 默认重试次数
INITIAL_RETRY_DELAY = 0.5                  # 初始退避延迟 (秒)
MAX_RETRY_DELAY = 8.0                      # 最大退避延迟 (秒)
JITTER = 0.75                               # 抖动因子

# 冷却相关
DEFAULT_COOLDOWN_TIME_SECONDS = 5          # 默认冷却时间 (秒)
DEFAULT_FAILURE_THRESHOLD_PERCENT = 0.5    # 错误率阈值 (50%)
DEFAULT_FAILURE_THRESHOLD_MINIMUM_REQUESTS = 5  # 最小请求样本
SINGLE_DEPLOYMENT_TRAFFIC_FAILURE_THRESHOLD = 1000  # 单实例流量阈值
DEFAULT_ALLOWED_FAILS = 3                  # v1 逻辑的允许失败次数

# 回退相关
ROUTER_MAX_FALLBACKS = 5                   # 最大回退深度
```

### 10.2 _should_retry() 状态码检查

```python
# utils.py:6715-6741
def _should_retry(status_code: int) -> bool:
    if status_code == 408:   return True   # Request Timeout
    if status_code == 409:   return True   # Conflict
    if status_code == 429:   return True   # Too Many Requests
    if status_code >= 500:   return True   # Server Errors
    
    return False  # 其他 4xx 不重试
```

**注意**：这只是**第一层**检查，`should_retry_this_error()` 有更多逻辑。

### 10.3 Router 初始化参数（可靠性相关）

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

---

## 十一、总结与最佳实践

### 11.1 本次修订的核心修正

| 原分析（错误） | 修正后（正确） |
|----------------|----------------|
| num_retries 优先级高 | **should_retry_this_error() 优先级更高** |
| 限流 + 无健康实例会重试 | **限流 + 无健康实例 + 有 fallback → 直接走 fallback，跳过重试** |
| 限流 + 无健康实例 + 无 fallback 会重试 | **限流 + 无健康实例 → 直接抛出异常，num_retries 无效** |
| 重试循环内设置冷却 | **冷却在最内层设置，在重试决策前已生效** |

### 11.2 设计意图理解

LiteLLM 的可靠性设计体现了以下**智能决策**：

1. **有健康实例时**：快速重试，透明切换
   - 延迟 = 0，即时重试
   - 利用同一 model_group 内的其他实例

2. **无健康实例但有 fallback 时**：快速切换，避免无效等待
   - 直接走 fallback，不浪费时间重试
   - 这是**合理的优化**，不是缺陷

3. **无健康实例也无 fallback 时**：快速失败
   - 不做无意义的重试
   - 让调用方快速知道问题

### 11.3 最佳实践建议

#### 建议 1：同一 model_group 配置多实例

```python
# 推荐：同一 model_group 下配置多个 provider/region
model_list = [
    {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "openai"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/gpt35-eastus"}, "model_info": {"id": "azure-east"}},
    {"model_name": "gpt-35", "litellm_params": {"model": "azure/gpt35-westus"}, "model_info": {"id": "azure-west"}},
]
# 好处：
# - 一个实例限流/失败时，自动切换到其他实例
# - 用户无感知，不需要配置 fallback
```

#### 建议 2：配置 fallback 作为最终保障

```python
# 推荐：配置跨 model_group 的 fallback
router = Router(
    model_list=[...],
    fallbacks=[
        {"gpt-35": ["claude-3-haiku", "llama-3-8b"]},
        {"gpt-4": ["claude-3-5-sonnet"]},
        {"*": ["gpt-35"]}  # 兜底
    ]
)
# 好处：
# - 当整个 model_group 的实例都不可用时
# - 自动切换到其他类型的模型
```

#### 建议 3：理解 num_retries 的真正作用

```python
# num_retries 只在有健康实例时生效
# 以下场景 num_retries 不会生效：
# - 无健康实例 + 有 fallback → 直接走 fallback
# - 无健康实例 + 无 fallback → 直接失败
# - 不可重试的状态码 (400, 404 等) → 直接抛出

# 所以不要依赖 num_retries 作为主要的可靠性保障
# 而是依赖：
# 1. 多实例配置（同一 model_group）
# 2. fallback 配置（跨 model_group）
```

#### 建议 4：合理配置冷却时间

```python
# 对于生产环境，建议适当延长冷却时间
router = Router(
    model_list=[...],
    cooldown_time=30,  # 冷却 30 秒，而不是默认 5 秒
)
# 原因：
# - 限流通常不是瞬时的
# - 频繁重试可能加剧问题
# - 给 provider 一些恢复时间
```

### 11.4 常见问题排查

#### 问题 1：为什么配置了 num_retries=10，但请求失败后没有重试？

**可能原因**：
1. `should_retry_this_error()` 抛出了异常
   - 检查：是否所有实例都冷却了？
   - 检查：是否是不可重试的状态码？
   - 检查：是否是单实例的 401 错误？

2. 冷却状态阻止了实例选择
   - 检查：`healthy_deployments` 是否为空？

**排查方法**：
```python
# 启用详细日志
import litellm
litellm.set_verbose = True
verbose_router_logger.setLevel(logging.DEBUG)
```

#### 问题 2：为什么 fallback 没有生效？

**可能原因**：
1. `should_retry_this_error()` 没有抛出异常
   - 检查：还有健康实例？如果有，会先重试
   - 只有无健康实例 + 有 fallback 才会直接走 fallback

2. fallback 配置格式错误
   - 标准格式：`[{"source": ["target"]}]`
   - 确认：是否是 `list[dict]` 格式？

3. 没有匹配到 model_group
   - 检查：源 model_name 是否正确？
   - 或使用通配符：`{"*": ["fallback-model"]}`

#### 问题 3：为什么一个实例失败后，所有请求都切到另一个实例？

**原因**：冷却机制
- 失败的实例被设置了冷却状态
- `_async_get_healthy_deployments()` 会过滤掉它
- 后续请求只能选择其他实例

**这是预期行为**，可以：
- 等待冷却时间到期（默认 5 秒）
- 或手动调整 `cooldown_time`

---

## 附录：代码位置索引

### A.1 核心函数位置

| 函数名 | 文件 | 行号 | 作用 |
|--------|------|------|------|
| `async_function_with_fallbacks` | `router.py` | 5594 | 外层回退入口 |
| `async_function_with_fallbacks_common_utils` | `router.py` | 5334 | 回退核心逻辑 |
| `async_function_with_retries` | `router.py` | 5693 | 中层重试控制 |
| `should_retry_this_error` | `router.py` | **5942** | **最高优先级决策** |
| `_time_to_sleep_before_retry` | `router.py` | 6060 | 重试延迟计算 |
| `_async_get_healthy_deployments` | `router.py` | 6536 | 获取健康实例 |
| `_set_cooldown_deployments` | `cooldown_handlers.py` | 260 | 设置冷却状态 |
| `_should_cooldown_deployment` | `cooldown_handlers.py` | 166 | 冷却决策 |
| `_is_cooldown_required` | `cooldown_handlers.py` | 40 | 冷却状态码检查 |
| `_should_retry` | `utils.py` | 6715 | 状态码重试检查 |
| `_calculate_retry_after` | `utils.py` | 6778 | 指数退避计算 |

### A.2 关键数据结构

```python
# CooldownCacheValue (cooldown_cache.py:24-29)
{
    "exception_received": str,    # 异常信息（脱敏）
    "status_code": str,           # HTTP 状态码
    "timestamp": float,           # 冷却开始时间戳
    "cooldown_time": float,       # 冷却持续时间
}

# 缓存键格式
f"deployment:{model_id}:cooldown"
```

---

## 修订记录

| 版本 | 日期 | 修订内容 |
|------|------|----------|
| v1.0 | 2026-05-02 | 初始版本 |
| **v2.0** | **2026-05-02** | **核心修正**：<br>1. `should_retry_this_error()` 优先级高于 `num_retries`<br>2. 限流 + 无健康实例 + 有 fallback → 直接走 fallback，跳过重试<br>3. 无健康实例时 `num_retries` 无效<br>4. 冷却在最内层设置，重试决策前已生效 |
