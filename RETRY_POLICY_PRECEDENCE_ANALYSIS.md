# LiteLLM 重试策略优先级深度分析报告（v3）

## 核心发现

**本次分析的关键发现**：

```
优先级顺序（从高到低）：

1. deployment_num_retries (从异常动态获取)
2. retry_policy (按异常类型的细粒度策略)  ◄─── 【关键】可跳过通用检查
3. should_retry_this_error() (通用重试判断)  ◄─── 包含健康实例、fallback 检查
4. num_retries (全局/请求级配置)
5. fallback (最后保障)
```

**最关键的逻辑**：
- 当 `retry_policy` 中**明确配置了当前异常类型的重试次数**时，`_retry_policy_applies = True`
- 当 `_retry_policy_applies = True` 时，**跳过 `should_retry_this_error()` 检查**
- `should_retry_this_error()` 包含：
  - 无健康实例 + 有 fallback → 直接走 fallback
  - 无健康实例 + 无 fallback → 直接失败

---

## 一、关键代码位置与注释

### 1.1 核心逻辑位置

```python
# router.py:5756-5787

# Check retry policy FIRST, before should_retry_this_error
# This allows retry policies to override the healthy deployments check
_retry_policy_applies = False
if self.retry_policy is not None or model_group_retry_policy is not None:
    # get num_retries from retry policy
    _retry_policy_retries = _get_num_retries_from_retry_policy(
        exception=original_exception,
        model_group=_model_group_for_retry_policy,
        model_group_retry_policy=model_group_retry_policy,
        retry_policy=self.retry_policy,
    )
    if _retry_policy_retries is not None:
        num_retries = _retry_policy_retries
        _retry_policy_applies = True  # 【关键】标记 retry policy 应用

# raises an exception if this error should not be retries
# Skip this check if retry policy applies (retry policy takes precedence)
if not _retry_policy_applies:  # 【关键】只有不应用时才执行通用检查
    self.should_retry_this_error(
        error=e,
        healthy_deployments=_healthy_deployments,
        all_deployments=_all_deployments,
        context_window_fallbacks=context_window_fallbacks,
        regular_fallbacks=fallbacks,
        content_policy_fallbacks=content_policy_fallbacks,
    )
```

### 1.2 关键注释解读

| 注释 | 含义 |
|------|------|
| `Check retry policy FIRST, before should_retry_this_error` | 先检查 retry policy，再检查通用重试判断 |
| `This allows retry policies to override the healthy deployments check` | 允许 retry policy 覆盖健康实例检查 |
| `Skip this check if retry policy applies (retry policy takes precedence)` | 如果 retry policy 应用，跳过通用检查（retry policy 优先级更高） |

### 1.3 重试循环内同样的逻辑

```python
# router.py:5850-5865
# Check if this error is non-retryable (e.g., 400 context
# window exceeded). If so, raise immediately instead of
# continuing the retry loop. Respect retry policy
# precedence - only check when no retry policy applies.
if not _retry_policy_applies:  # 同样的逻辑
    try:
        self.should_retry_this_error(
            error=e,
            healthy_deployments=_healthy_deployments,
            all_deployments=_all_deployments,
            context_window_fallbacks=context_window_fallbacks,
            regular_fallbacks=fallbacks,
            content_policy_fallbacks=content_policy_fallbacks,
        )
    except Exception:
        raise e
```

---

## 二、`_retry_policy_applies` 的判断逻辑

### 2.1 什么时候 `_retry_policy_applies = True`？

```python
# router.py:5758-5775
_retry_policy_applies = False

# 条件 1: 配置了 retry_policy 或 model_group_retry_policy
if self.retry_policy is not None or model_group_retry_policy is not None:
    
    # 从 retry policy 获取重试次数
    _retry_policy_retries = _get_num_retries_from_retry_policy(
        exception=original_exception,  # 当前异常
        ...
    )
    
    # 条件 2: 返回非 None 值（即当前异常类型在 retry policy 中有配置）
    if _retry_policy_retries is not None:
        num_retries = _retry_policy_retries
        _retry_policy_applies = True  # 标记应用
```

### 2.2 `get_num_retries_from_retry_policy()` 实现

```python
# router_utils/get_retry_from_policy.py:19-68
def get_num_retries_from_retry_policy(
    exception: Exception,
    retry_policy: Optional[Union[RetryPolicy, dict]] = None,
    ...
):
    # 先检查 model_group_retry_policy（更细粒度）
    if (
        model_group_retry_policy is not None
        and model_group is not None
        and model_group in model_group_retry_policy
    ):
        retry_policy = model_group_retry_policy.get(model_group, None)
    
    # 没有配置任何 policy → 返回 None
    if retry_policy is None:
        return None
    
    # 按异常类型匹配
    if isinstance(exception, AuthenticationError):
        return retry_policy.AuthenticationErrorRetries  # 可能是 None
    
    if isinstance(exception, Timeout):
        return retry_policy.TimeoutErrorRetries  # 可能是 None
    
    if isinstance(exception, RateLimitError):
        return retry_policy.RateLimitErrorRetries  # 可能是 None
    
    if isinstance(exception, ContentPolicyViolationError):
        return retry_policy.ContentPolicyViolationErrorRetries
    
    if isinstance(exception, BadRequestError):
        return retry_policy.BadRequestErrorRetries

    # 没有匹配到 → 返回 None（隐式）
```

### 2.3 RetryPolicy 类型定义

```python
# types/router.py 中的 RetryPolicy 定义（推测）
class RetryPolicy:
    """
    按异常类型配置的细粒度重试策略
    
    注意：所有字段都是 Optional[int] = None
    """
    BadRequestErrorRetries: Optional[int] = None
    AuthenticationErrorRetries: Optional[int] = None
    TimeoutErrorRetries: Optional[int] = None
    RateLimitErrorRetries: Optional[int] = None
    ContentPolicyViolationErrorRetries: Optional[int] = None
```

---

## 三、四种配置场景的行为对比

### 3.1 场景矩阵

| 场景 | 配置 | 当前异常类型 | `_retry_policy_applies` | 执行 `should_retry_this_error()` |
|------|------|-------------|------------------------|----------------------------------|
| **A** | 无 retry_policy | RateLimitError | False | ✅ 执行 |
| **B** | `RetryPolicy()` 空对象 | RateLimitError | False (RateLimitErrorRetries=None) | ✅ 执行 |
| **C** | `RetryPolicy(AuthenticationErrorRetries=5)` | RateLimitError | False (RateLimitErrorRetries=None) | ✅ 执行 |
| **D** | `RetryPolicy(RateLimitErrorRetries=5)` | RateLimitError | **True** | ❌ **跳过** |

### 3.2 场景详解

#### 场景 A：无 retry_policy 配置

```python
router = Router(
    model_list=[...],
    # 没有配置 retry_policy
    num_retries=2,
)
```

**流程**：
```
1. 条件检查：
   - self.retry_policy is not None? → 否
   - 不进入 if 块
   - _retry_policy_applies = False

2. 执行通用检查：
   if not _retry_policy_applies:  # True
       self.should_retry_this_error(...)  # 执行
```

**结果**：执行 `should_retry_this_error()` 检查，包括：
- 无健康实例 + 有 fallback → 直接走 fallback
- 无健康实例 + 无 fallback → 直接失败

---

#### 场景 B：RetryPolicy 空对象

```python
from litellm.types.router import RetryPolicy

router = Router(
    model_list=[...],
    retry_policy=RetryPolicy(),  # 所有字段都是 None
    num_retries=2,
)
```

**流程**：
```
1. 条件检查：
   - self.retry_policy is not None? → 是
   - 进入 if 块
   
2. 调用 get_num_retries_from_retry_policy(RateLimitError):
   - retry_policy 非 None
   - isinstance(exception, RateLimitError)? → 是
   - retry_policy.RateLimitErrorRetries is not None? → 否（是 None）
   - 返回 None

3. 结果：
   - _retry_policy_retries is not None? → 否
   - _retry_policy_applies = False

4. 执行通用检查：
   if not _retry_policy_applies:  # True
       self.should_retry_this_error(...)  # 执行
```

**结果**：虽然配置了 `retry_policy`，但当前异常类型没有明确配置重试次数，仍执行通用检查。

---

#### 场景 C：配置了其他异常类型的策略

```python
from litellm.types.router import RetryPolicy

router = Router(
    model_list=[...],
    retry_policy=RetryPolicy(
        AuthenticationErrorRetries=5,  # 只配置了认证错误
        TimeoutErrorRetries=3,         # 和超时错误
    ),
    num_retries=2,
)
```

**当前异常**：RateLimitError (429)

**流程**：
```
1. 条件检查：
   - self.retry_policy is not None? → 是
   - 进入 if 块
   
2. 调用 get_num_retries_from_retry_policy(RateLimitError):
   - isinstance(exception, AuthenticationError)? → 否
   - isinstance(exception, Timeout)? → 否
   - isinstance(exception, RateLimitError)? → 是
   - retry_policy.RateLimitErrorRetries is not None? → 否（是 None）
   - 返回 None

3. 结果：
   - _retry_policy_applies = False

4. 执行通用检查：
   if not _retry_policy_applies:  # True
       self.should_retry_this_error(...)  # 执行
```

**结果**：当前异常类型（RateLimitError）没有配置，仍执行通用检查。

---

#### 场景 D：配置了当前异常类型的策略【关键场景】

```python
from litellm.types.router import RetryPolicy

router = Router(
    model_list=[...],
    retry_policy=RetryPolicy(
        RateLimitErrorRetries=5,  # 【关键】配置了限流错误的重试次数
    ),
    num_retries=2,  # 会被覆盖
)
```

**当前异常**：RateLimitError (429)

**流程**：
```
1. 条件检查：
   - self.retry_policy is not None? → 是
   - 进入 if 块
   
2. 调用 get_num_retries_from_retry_policy(RateLimitError):
   - isinstance(exception, RateLimitError)? → 是
   - retry_policy.RateLimitErrorRetries is not None? → 是（=5）
   - 返回 5

3. 结果：
   - _retry_policy_retries = 5
   - num_retries = 5  # 覆盖原配置
   - _retry_policy_applies = True  # 【关键】标记应用

4. 跳过通用检查：
   if not _retry_policy_applies:  # False（因为 _retry_policy_applies = True）
       self.should_retry_this_error(...)  # ❌ 不执行！跳过！

5. 进入重试循环：
   if num_retries > 0:  # 5 > 0 → 是
       # 进入重试循环
       for current_attempt in range(num_retries):  # range(5)
           # 重试 5 次
           # 注意：循环内同样跳过 should_retry_this_error()
```

**关键发现**：
- `should_retry_this_error()` 被**完全跳过**
- 这意味着：
  - **不检查是否有健康实例**
  - **不检查是否有 fallback 配置**
  - **无论什么情况，都会进入重试循环**

---

## 四、限流场景的完整行为分析

### 4.1 四种限流场景 + 两种配置组合

让我们分析 **限流 (RateLimitError)** + **无健康实例** + **有/无 fallback** + **有/无 retry_policy 配置** 的组合。

#### 组合 1：限流 + 无健康实例 + 无 retry_policy + 有 fallback

```python
router = Router(
    model_list=[...],
    # 无 retry_policy
    num_retries=2,
    fallbacks=[{"gpt-35": ["claude"]}],  # 有 fallback
)
```

**执行流程**：
```
1. 限流错误 (RateLimitError)
2. 设置当前 deployment 冷却
3. healthy_deployments = [] (无健康实例)
4. _retry_policy_applies = False
5. 执行 should_retry_this_error():
   - 是 RateLimitError? → 是
   - 检查条件：
     * _num_healthy <= 0? → 是 ✓
     * regular_fallbacks is not None? → 是 ✓
     * len(regular_fallbacks) > 0? → 是 ✓
   - 【抛出异常！】
6. 异常传播，不检查 num_retries
7. 外层捕获，走 fallback
```

**结果**：直接走 fallback，不重试。

---

#### 组合 2：限流 + 无健康实例 + 无 retry_policy + 无 fallback

```python
router = Router(
    model_list=[...],
    # 无 retry_policy
    num_retries=2,
    # 无 fallback
)
```

**执行流程**：
```
1. 限流错误
2. 设置冷却
3. healthy_deployments = []
4. _retry_policy_applies = False
5. 执行 should_retry_this_error():
   - 是 RateLimitError? → 是
   - 检查条件：
     * _num_healthy <= 0? → 是 ✓
     * regular_fallbacks is not None? → 否 ✗
   - RateLimitError 条件不满足，继续
   - 最后检查：_num_healthy <= 0? → 是 ✓
   - 【抛出异常！】
6. 异常传播
7. 无 fallback，请求失败
```

**结果**：直接失败，不重试。

---

#### 组合 3：限流 + 无健康实例 + 有 retry_policy + 有 fallback【关键】

```python
router = Router(
    model_list=[...],
    retry_policy=RetryPolicy(
        RateLimitErrorRetries=5,  # 【关键】配置了限流错误的重试次数
    ),
    num_retries=2,
    fallbacks=[{"gpt-35": ["claude"]}],  # 有 fallback
)
```

**执行流程**：
```
1. 限流错误
2. 设置冷却
3. healthy_deployments = [] (无健康实例)
4. _retry_policy_applies = True  (因为配置了 RateLimitErrorRetries)
5. 【跳过 should_retry_this_error()】
   if not _retry_policy_applies:  # False，不执行
       self.should_retry_this_error(...)  # 跳过！

6. 检查 num_retries > 0:
   - num_retries = 5 (从 retry_policy 获取)
   - 5 > 0 → 是

7. 进入重试循环：
   for current_attempt in range(5):
       # 计算重试延迟
       _timeout = self._time_to_sleep_before_retry(
           healthy_deployments=[],  # 空列表！
           ...
       )
       
       # _time_to_sleep_before_retry() 逻辑：
       # - healthy_deployments is not None and len > 0? → 否
       # - 走指数退避路径
       
       await asyncio.sleep(_timeout)  # 等待指数退避时间
       
       # 重试调用
       response = await self.make_call(...)
       
       # 又失败（因为无健康实例）
       original_exception = e
       
       # 重试循环内同样跳过通用检查
       if not _retry_policy_applies:  # False，不执行
           self.should_retry_this_error(...)  # 跳过！

8. 重试 5 次后耗尽
9. 抛出异常
10. 外层捕获，走 fallback
```

**关键发现**：
- `should_retry_this_error()` 被跳过
- **无健康实例的检查被跳过**
- **有 fallback 应该直接走 fallback 的逻辑被跳过**
- 而是**先重试 5 次**（每次都等待指数退避时间）
- 重试耗尽后才走 fallback

**这是一个风险点**：
- 如果所有实例都被冷却，重试 5 次都是浪费时间
- 但这是**设计意图**（注释说 "allows retry policies to override the healthy deployments check"）

---

#### 组合 4：限流 + 无健康实例 + 有 retry_policy + 无 fallback【关键】

```python
router = Router(
    model_list=[...],
    retry_policy=RetryPolicy(
        RateLimitErrorRetries=5,  # 配置了限流错误的重试次数
    ),
    num_retries=2,
    # 无 fallback
)
```

**执行流程**：
```
1. 限流错误
2. 设置冷却
3. healthy_deployments = []
4. _retry_policy_applies = True
5. 跳过 should_retry_this_error()
6. 进入重试循环 (5 次)
7. 每次重试都等待指数退避时间
8. 每次重试都失败（无健康实例）
9. 重试 5 次后耗尽
10. 抛出异常
11. 无 fallback，请求失败
```

**关键发现**：
- 无 retry_policy 时：直接失败
- 有 retry_policy 时：先重试 5 次，再失败
- **retry_policy 让请求在无健康实例时仍然重试**

---

### 4.2 限流场景行为对比表

| 配置 | 无健康实例 + 有 fallback | 无健康实例 + 无 fallback |
|------|--------------------------|--------------------------|
| **无 retry_policy** | 直接走 fallback，不重试 | 直接失败，不重试 |
| **配置了 RateLimitErrorRetries** | 先重试 N 次，再走 fallback | 先重试 N 次，再失败 |

---

## 五、should_retry_this_error() 完整逻辑

### 5.1 完整决策树

```python
# router.py:5942-6012
def should_retry_this_error(
    error,
    healthy_deployments,      # 当前健康实例列表
    all_deployments,          # 所有实例列表
    context_window_fallbacks, # 上下文窗口回退
    content_policy_fallbacks, # 内容策略回退
    regular_fallbacks,        # 普通回退
):
    """
    返回 True 表示允许重试
    抛出异常表示禁止重试
    """
    _num_healthy = len(healthy_deployments) if healthy_deployments else 0
    _num_all = len(all_deployments) if all_deployments else 0

    # ═══════════════════════════════════════════════════════════
    # 第一层：特殊错误 + 有对应 fallback → 直接走 fallback
    # ═══════════════════════════════════════════════════════════
    
    if isinstance(error, ContextWindowExceededError) and context_window_fallbacks:
        raise error  # 直接走 context_window_fallback
    
    if isinstance(error, ContentPolicyViolationError) and content_policy_fallbacks:
        raise error  # 直接走 content_policy_fallback

    # ═══════════════════════════════════════════════════════════
    # 第二层：不可重试的状态码
    # ═══════════════════════════════════════════════════════════
    
    status_code = getattr(error, "status_code", None)
    if status_code is not None and not litellm._should_retry(status_code):
        # _should_retry() 返回 False 的情况：400, 401, 403, 404 等
        # 但 401/403 是例外
        if status_code not in (401, 403):
            raise error  # 其他 4xx 直接抛出

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
            raise error  # 【关键】直接走 fallback，跳过重试！

    # ═══════════════════════════════════════════════════════════
    # 第五层：AuthenticationError 的特殊处理
    # ═══════════════════════════════════════════════════════════
    
    if isinstance(error, openai.AuthenticationError):
        # 只有一个实例时才禁止重试
        if _num_all <= 1:
            raise error
        # 多个实例时允许重试

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

### 5.2 决策表

| 条件组合 | 结果 |
|----------|------|
| ContextWindowExceededError + 有 context_window_fallbacks | **RAISE** |
| ContentPolicyViolationError + 有 content_policy_fallbacks | **RAISE** |
| 状态码 400/404/406...（非 401/403）且 _should_retry=False | **RAISE** |
| NotFoundError | **RAISE** |
| RateLimitError + 无健康实例 + 有 fallback | **RAISE** ◄──── 关键！ |
| AuthenticationError + 只有一个实例 | **RAISE** |
| 无健康实例（兜底检查） | **RAISE** |
| **其他所有情况** | **Return True → 进入重试循环** |

---

## 六、优先级总览

### 6.1 完整优先级金字塔

```
                    ┌───────────────────────────────┐
                    │  1. deployment_num_retries    │
                    │     (从异常动态获取)           │
                    │                               │
                    │  最高优先级，覆盖所有其他配置  │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  2. retry_policy              │
                    │     (按异常类型的细粒度策略)   │
                    │                               │
                    │  【关键】可跳过通用检查       │
                    │  当配置了当前异常类型时       │
                    │  _retry_policy_applies = True │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  3. should_retry_this_error() │
                    │     (通用重试判断)            │
                    │                               │
                    │  包含：                       │
                    │  - 健康实例检查               │
                    │  - fallback 检查              │
                    │  - 异常类型检查               │
                    │                               │
                    │  【关键】被 retry_policy 跳过 │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  4. num_retries               │
                    │     (全局/请求级配置)         │
                    │                               │
                    │  只有前面都通过时才生效       │
                    └───────────────┬───────────────┘
                                    │
                                    ▼
                    ┌───────────────────────────────┐
                    │  5. fallback                  │
                    │     (最后保障)                │
                    │                               │
                    │  只有前面都失败时才触发       │
                    └───────────────────────────────┘
```

### 6.2 配置获取顺序

```python
# router.py:5739-5775

# 1. 最高优先级：从异常获取 deployment 级别的配置
deployment_num_retries = getattr(e, "num_retries", None)
if deployment_num_retries is not None and isinstance(deployment_num_retries, int):
    num_retries = deployment_num_retries

# 2. 次高优先级：retry_policy（按异常类型）
if self.retry_policy is not None or model_group_retry_policy is not None:
    _retry_policy_retries = _get_num_retries_from_retry_policy(...)
    if _retry_policy_retries is not None:
        num_retries = _retry_policy_retries
        _retry_policy_applies = True  # 标记应用

# 3. 只有不应用 retry_policy 时，才执行通用检查
if not _retry_policy_applies:
    self.should_retry_this_error(...)

# 4. 最后检查 num_retries > 0
if num_retries > 0:
    # 进入重试循环
else:
    raise
```

---

## 七、设计意图与风险分析

### 7.1 设计意图

代码中的注释明确说明了设计意图：

```python
# Check retry policy FIRST, before should_retry_this_error
# This allows retry policies to override the healthy deployments check
```

**意图**：
- 允许用户通过 `retry_policy` 覆盖默认的健康实例检查逻辑
- 这是一种**高级特性**，给用户更大的控制权

**场景说明**：
- 默认行为：无健康实例时直接走 fallback 或失败
- 覆盖行为：用户可能希望在无健康实例时仍然重试（例如：等待实例恢复）

### 7.2 潜在风险

**风险场景**：
```python
router = Router(
    model_list=[
        {"model_name": "gpt-35", "litellm_params": {"model": "gpt-3.5-turbo"}, "model_info": {"id": "dep-1"}},
    ],
    retry_policy=RetryPolicy(
        RateLimitErrorRetries=10,  # 配置 10 次重试
    ),
    num_retries=2,
)
```

**问题**：
- 单实例场景，实例被限流并冷却
- `healthy_deployments = []`
- 默认行为：直接失败（不浪费时间）
- 配置 `retry_policy` 后：先重试 10 次，每次等待指数退避时间
- 总共等待时间可能很长（例如：0.5s → 1s → 2s → 4s → 8s → ...）

**建议**：
1. **理解 `retry_policy` 的影响**：它会跳过健康实例检查
2. **单实例场景慎用 `retry_policy`**：因为重试可能都是无效的
3. **合理配置重试次数**：不要配置过大的重试次数
4. **考虑配合较短的冷却时间**：让实例更快回到可用池

### 7.3 最佳实践

#### 推荐配置 1：多实例场景

```python
# 多实例，有其他实例可以切换
# 配置 retry_policy 可以让限流错误重试更多次
# 但注意：如果所有实例都冷却了，会先重试再走 fallback

router = Router(
    model_list=[
        {"model_name": "gpt-35", ..., "model_info": {"id": "dep-1"}},
        {"model_name": "gpt-35", ..., "model_info": {"id": "dep-2"}},
        {"model_name": "gpt-35", ..., "model_info": {"id": "dep-3"}},
    ],
    retry_policy=RetryPolicy(
        RateLimitErrorRetries=5,      # 限流错误重试 5 次
        TimeoutErrorRetries=3,         # 超时错误重试 3 次
    ),
    fallbacks=[{"gpt-35": ["claude"]}],  # 最终 fallback
)
```

**预期行为**：
- 一个实例限流 → 冷却 → 切换到其他实例（透明）
- 所有实例限流 → 冷却 → 先重试 5 次 → 再走 fallback

#### 推荐配置 2：单实例场景

```python
# 单实例，没有其他实例可以切换
# 不配置 retry_policy，让默认行为生效

router = Router(
    model_list=[
        {"model_name": "gpt-35", ..., "model_info": {"id": "dep-1"}},
    ],
    # 不配置 retry_policy
    num_retries=2,  # 有健康实例时生效
    fallbacks=[{"gpt-35": ["claude"]}],
)
```

**预期行为**：
- 实例限流 → 冷却 → `healthy_deployments = []`
- 执行 `should_retry_this_error()` → 发现无健康实例 + 有 fallback
- **直接走 fallback**，不重试

---

## 八、总结

### 8.1 核心结论

| 问题 | 答案 |
|------|------|
| retry_policy 与通用检查的先后关系？ | **retry_policy 先检查，可跳过通用检查** |
| 什么时候 `_retry_policy_applies = True`？ | **当 retry_policy 中明确配置了当前异常类型的重试次数时** |
| 限流 + 无健康实例 + 有 fallback 时会走哪条路？ | **无 retry_policy：直接走 fallback；有 retry_policy：先重试 N 次再走 fallback** |
| `should_retry_this_error()` 包含什么检查？ | **健康实例检查、fallback 检查、异常类型检查** |

### 8.2 关键决策点

```
请求失败
    │
    ▼
设置冷却状态（如果需要）
    │
    ▼
获取 healthy_deployments（过滤冷却中的）
    │
    ▼
检查 deployment_num_retries（从异常获取）
    │
    ▼
检查 retry_policy 是否应用
    │
    ├──► 应用 (_retry_policy_applies = True)
    │       │
    │       ▼
    │  【跳过 should_retry_this_error()】
    │       │
    │       ▼
    │  检查 num_retries > 0
    │       │
    │       ├──► 是 → 进入重试循环
    │       │
    │       └──► 否 → 抛出异常 → 走 fallback（如果有）
    │
    └──► 不应用 (_retry_policy_applies = False)
            │
            ▼
        执行 should_retry_this_error()
            │
            ├──► 抛出异常
            │       │
            │       ├──► 无健康实例 + 有 fallback → 直接走 fallback
            │       │
            │       └──► 其他情况 → 走 fallback 或失败
            │
            └──► 返回 True
                    │
                    ▼
                检查 num_retries > 0
                    │
                    ├──► 是 → 进入重试循环
                    │
                    └──► 否 → 抛出异常
```

### 8.3 配置建议

| 场景 | 建议 |
|------|------|
| **多实例 + 需要精细控制** | 使用 `retry_policy`，理解其会跳过健康实例检查 |
| **多实例 + 保守策略** | 不使用 `retry_policy`，依赖默认的 `should_retry_this_error()` 行为 |
| **单实例** | **不推荐使用 `retry_policy`**，因为重试可能都是无效的 |
| **需要快速失败** | 不使用 `retry_policy`，让无健康实例时直接走 fallback 或失败 |
| **需要等待实例恢复** | 使用 `retry_policy`，但配置合理的重试次数和冷却时间 |

---

## 附录：代码位置索引

| 函数/逻辑 | 文件 | 行号 | 作用 |
|-----------|------|------|------|
| `_retry_policy_applies` 标记 | `router.py` | 5756-5775 | 检查 retry_policy 是否应用 |
| 跳过通用检查的条件 | `router.py` | 5779-5787 | `if not _retry_policy_applies` |
| 重试循环内同样的逻辑 | `router.py` | 5854-5865 | 循环内也跳过通用检查 |
| `should_retry_this_error()` | `router.py` | 5942-6012 | 通用重试判断 |
| `get_num_retries_from_retry_policy()` | `router_utils/get_retry_from_policy.py` | 19-68 | 从 policy 获取重试次数 |

---

## 修订记录

| 版本 | 日期 | 修订内容 |
|------|------|----------|
| v1.0 | 2026-05-02 | 初始版本 |
| v2.0 | 2026-05-02 | 修正优先级：`should_retry_this_error()` 高于 `num_retries` |
| **v3.0** | **2026-05-02** | **深度分析 `retry_policy` 与通用检查的先后关系**：<br>1. `retry_policy` 优先级更高<br>2. 当配置了当前异常类型时，**跳过 `should_retry_this_error()`**<br>3. 这意味着：<br>   - 无健康实例的检查被跳过<br>   - 有 fallback 时直接走 fallback 的逻辑被跳过<br>   - 会先重试 N 次再走 fallback（或失败）<br>4. 这是**设计意图**（注释说明），但需注意潜在风险 |
