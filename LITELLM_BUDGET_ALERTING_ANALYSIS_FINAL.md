# LiteLLM 预算告警机制最终分析报告

## ⚠️ 核心修正声明

本文档基于**真实代码执行顺序**进行分析，修正了之前报告中的多处关键错误。

---

## 目录

1. [真实调用顺序（核心修正）](#真实调用顺序核心修正)
2. [各检查函数的职责与行为](#各检查函数的职责与行为)
3. [阶段之间的影响关系](#阶段之间的影响关系)
4. [告警类型映射与事件判断](#告警类型映射与事件判断)
5. [完整状态流转图](#完整状态流转图)
6. [关键代码位置索引](#关键代码位置索引)

---

## 真实调用顺序（核心修正）

### 代码原文

**文件位置**: `litellm/proxy/auth/user_api_key_auth.py:1360-1384`

```python
if not skip_budget_checks:
    with tracer.trace("litellm.proxy.auth.budget_checks"):
        # Check 4. Max Budget Alert Check (runs before budget enforcement
        # so multi-threshold 100% alerts fire on the request that crosses
        # max_budget, before BudgetExceededError is raised below)
        await _virtual_key_max_budget_alert_check(
            valid_token=valid_token,
            proxy_logging_obj=proxy_logging_obj,
            user_obj=user_obj,
        )

        # Check 5. Token Spend is under budget
        if RouteChecks.is_llm_api_route(route=route):
            await _virtual_key_max_budget_check(
                valid_token=valid_token,
                proxy_logging_obj=proxy_logging_obj,
                user_obj=user_obj,
            )

        # Check 6. Soft Budget Check
        await _virtual_key_soft_budget_check(
            valid_token=valid_token,
            proxy_logging_obj=proxy_logging_obj,
            user_obj=user_obj,
        )
```

### ⚠️ 真实执行顺序

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         真实调用顺序（按代码）                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  第 1 步: _virtual_key_max_budget_alert_check()                            │
│          ├─ 触发类型: type="max_budget_alert"                              │
│          ├─ 行为: 只触发告警，不抛异常                                      │
│          ├─ 执行条件: 总是执行（无路由限制）                                 │
│          └─ 用途: 邮件多阈值预警（50%/75%/80%/90% 等）                    │
│                                                                             │
│  第 2 步: _virtual_key_max_budget_check()                                  │
│          ├─ 触发类型: type="token_budget"                                  │
│          ├─ 行为: 总是触发告警 + 可能抛出 BudgetExceededError              │
│          ├─ ⚠️ 执行条件: 仅在 LLM 路由执行 (RouteChecks.is_llm_api_route)│
│          └─ 用途: 硬预算检查（15%/5% 阈值预警 + 硬预算超限阻止）            │
│                                                                             │
│  第 3 步: _virtual_key_soft_budget_check()                                 │
│          ├─ 触发类型: type="soft_budget"                                   │
│          ├─ 行为: 只触发告警，不抛异常                                      │
│          ├─ 执行条件: 总是执行（无路由限制）                                 │
│          └─ 用途: 软预算预警                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ⚠️ 之前报告的错误对比

| 维度 | 之前错误结论 | 修正后真实结论 | 代码依据 |
|------|------------|---------------|---------|
| **调用顺序** | 1. soft_budget → 2. max_budget_alert → 3. max_budget | 1. max_budget_alert → 2. max_budget (仅LLM) → 3. soft_budget | `user_api_key_auth.py:1360-1384` |
| **max_budget_check 执行条件** | 总是执行 | 仅在 LLM 路由执行 | `user_api_key_auth.py:1372` 的 `if RouteChecks.is_llm_api_route()` |
| **各阶段影响** | 独立执行，互不影响 | 如果第 2 步抛出异常，第 3 步**不会执行** | Python 异常传播机制 |
| **max_budget_check 的告警触发** | 只在超过预算时触发 | **总是**触发 `type="token_budget"` 告警 | `auth_checks.py:3031-3036` 没有条件判断 |

---

## 各检查函数的职责与行为

### 1. `_virtual_key_max_budget_alert_check()`

**文件位置**: `litellm/proxy/auth/auth_checks.py:3171-3263`

#### 职责
- 邮件多阈值预算预警
- 在硬预算检查**之前**执行，确保即使达到 100% 也能触发邮件告警

#### 触发条件

```python
# 新路径（多阈值配置）
alert_email_config = _merge_budget_alert_email_configs(
    global_cfg=litellm.default_key_max_budget_alert_emails,
    per_key_cfg=(valid_token.metadata or {}).get("max_budget_alert_emails"),
)

# 只在达到最低阈值时才创建告警任务
min_pct = min((int(k) for k in alert_email_config if k.isdigit()), default=None)
if min_pct is None or valid_token.spend < valid_token.max_budget * (min_pct / 100.0):
    return  # 未达到最低阈值，直接返回

# 触发告警
asyncio.create_task(
    proxy_logging_obj.budget_alerts(
        type="max_budget_alert",
        user_info=call_info,  # 包含 max_budget_alert_emails
    )
)
```

#### 旧路径（单一 80% 阈值）

```python
# 旧路径: existing single 80% threshold
alert_threshold = valid_token.max_budget * EMAIL_BUDGET_ALERT_MAX_SPEND_ALERT_PERCENTAGE

# ⚠️ 关键条件: spend < max_budget
if (
    valid_token.spend >= alert_threshold
    and valid_token.spend < valid_token.max_budget  # ⬅️ 必须小于 max_budget
):
    asyncio.create_task(
        proxy_logging_obj.budget_alerts(
            type="max_budget_alert",
            user_info=call_info,
        )
    )
```

#### 行为总结

| 维度 | 行为 |
|------|------|
| **触发类型** | `type="max_budget_alert"` |
| **是否抛异常** | ❌ 从不抛异常 |
| **执行条件** | 总是执行（无路由限制）|
| **新路径触发条件** | `spend >= max_budget * (min_pct / 100.0)` |
| **旧路径触发条件** | `spend >= 80% AND spend < max_budget` |

---

### 2. `_virtual_key_max_budget_check()`

**文件位置**: `litellm/proxy/auth/auth_checks.py:2990-3047`

#### 职责
- 硬预算检查
- 剩余预算预警（15% / 5%）
- 预算超限时阻止请求

#### ⚠️ 关键发现：总是触发告警

```python
async def _virtual_key_max_budget_check(...):
    if valid_token.max_budget is not None:
        # 读取 spend...
        spend = await get_current_spend(...)
        
        # 构建 CallInfo...
        call_info = CallInfo(...)
        
        # ⚠️ 总是触发告警任务（无论是否超过预算）
        # 这里没有任何条件判断！
        asyncio.create_task(
            proxy_logging_obj.budget_alerts(
                type="token_budget",
                user_info=call_info,
            )
        )
        
        # ⚠️ 只有超过时才抛异常
        if spend >= valid_token.max_budget:
            raise litellm.BudgetExceededError(
                current_cost=spend,
                max_budget=valid_token.max_budget,
            )
```

#### 行为总结

| 维度 | 行为 |
|------|------|
| **触发类型** | `type="token_budget"` |
| **是否抛异常** | ✅ 当 `spend >= max_budget` 时抛出 `BudgetExceededError` |
| **执行条件** | 仅在 LLM 路由执行 (`RouteChecks.is_llm_api_route`) |
| **告警触发** | **总是**触发（无论是否超过预算）|

---

### 3. `_virtual_key_soft_budget_check()`

**文件位置**: `litellm/proxy/auth/auth_checks.py:3086-3123`

#### 职责
- 软预算预警

#### 触发条件

```python
async def _virtual_key_soft_budget_check(...):
    # 只有当 soft_budget 已设置且 spend >= soft_budget 时才触发
    if valid_token.soft_budget and valid_token.spend >= valid_token.soft_budget:
        call_info = CallInfo(...)
        
        asyncio.create_task(
            proxy_logging_obj.budget_alerts(
                type="soft_budget",
                user_info=call_info,
            )
        )
```

#### 行为总结

| 维度 | 行为 |
|------|------|
| **触发类型** | `type="soft_budget"` |
| **是否抛异常** | ❌ 从不抛异常 |
| **执行条件** | 总是执行（无路由限制）|
| **告警触发条件** | `soft_budget is not None AND spend >= soft_budget` |

---

## 阶段之间的影响关系

### 核心影响链路

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         阶段影响关系图                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 第 1 步: _virtual_key_max_budget_alert_check()                       │ │
│  │                                                                       │ │
│  │  行为:                                                                 │ │
│  │  - 总是执行（无路由限制）                                              │ │
│  │  - 只触发告警，不抛异常                                                │ │
│  │  - 不会影响后续阶段的执行                                              │ │
│  │                                                                       │ │
│  │  影响: 无（从不抛异常）                                                │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 第 2 步: _virtual_key_max_budget_check()                             │ │
│  │                                                                       │ │
│  │  行为:                                                                 │ │
│  │  - 仅在 LLM 路由执行                                                  │ │
│  │  - 总是触发 type="token_budget" 告警                                  │ │
│  │  - 可能抛出 BudgetExceededError                                       │ │
│  │                                                                       │ │
│  │  影响:                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │ 分支 A: spend < max_budget                                    │   │ │
│  │  │   - 不抛异常                                                  │   │ │
│  │  │   - 继续执行第 3 步                                           │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  │                              │                                        │ │
│  │                              ▼                                        │ │
│  │  ┌─────────────────────────────────────────────────────────────┐   │ │
│  │  │ 分支 B: spend >= max_budget                                   │   │ │
│  │  │   - 抛出 BudgetExceededError                                  │   │ │
│  │  │   - ⚠️ 第 3 步 不会执行！                                     │   │ │
│  │  │   - 请求被阻止，返回 HTTP 429                                  │   │ │
│  │  └─────────────────────────────────────────────────────────────┘   │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                    │                                        │
│                                    ▼ (仅在分支 A 时)                        │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ 第 3 步: _virtual_key_soft_budget_check()                            │ │
│  │                                                                       │ │
│  │  行为:                                                                 │ │
│  │  - 总是执行（无路由限制）                                              │ │
│  │  - 只触发告警，不抛异常                                                │ │
│  │                                                                       │ │
│  │  ⚠️ 注意: 如果第 2 步抛出异常，这一步不会执行！                        │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 关键影响总结

| 场景 | 第 1 步 | 第 2 步 | 第 3 步 | 最终结果 |
|------|--------|--------|--------|---------|
| **非 LLM 路由** | ✅ 执行 | ❌ 不执行 | ✅ 执行 | 请求正常 |
| **LLM 路由，spend < max_budget** | ✅ 执行 | ✅ 执行（不抛异常）| ✅ 执行 | 请求正常 |
| **LLM 路由，spend >= max_budget** | ✅ 执行 | ✅ 执行（抛异常）| ❌ **不执行** | HTTP 429 |

---

## 告警类型映射与事件判断

### 告警类型 → 事件类型 映射

当 `proxy_logging_obj.budget_alerts(type=..., user_info=...)` 被调用后：

#### 1. Slack 的事件判断逻辑

**文件位置**: `litellm/integrations/SlackAlerting/slack_alerting.py:639-699`

Slack 会忽略传入的 `type`，根据实际的 `spend` 和 `budget` 值**重新判断**事件类型：

```python
def _get_event_and_event_message(...):
    percent_left = (max_budget - spend) / max_budget
    
    # 第一优先级: soft_budget
    if user_info.soft_budget is not None:
        if user_info.spend >= user_info.soft_budget:
            event = "soft_budget_crossed"
    
    # 第二优先级: max_budget
    if user_info.max_budget is not None:
        if user_info.spend >= user_info.max_budget:
            event = "budget_crossed"
        elif percent_left <= 0.05:  # 剩余 5%
            event = "threshold_crossed"
        elif percent_left <= 0.15:  # 剩余 15%
            event = "threshold_crossed"
    
    return event, event_message
```

#### 2. 邮件的事件判断逻辑

**文件位置**: `enterprise/.../send_emails/base_email.py:414-668`

邮件**严格按照**传入的 `type` 走不同分支：

```python
async def budget_alerts(self, type, user_info):
    # 分支 1: type == "soft_budget"
    if type == "soft_budget":
        if user_info.soft_budget is not None and user_info.spend >= user_info.soft_budget:
            # 发送 soft_budget_crossed 邮件
            ...
        return  # ← 直接返回
    
    # 分支 2: type == "max_budget_alert"
    if type == "max_budget_alert":
        if user_info.max_budget_alert_emails:
            # 多阈值独立处理
            await self._handle_multi_threshold_max_budget_alert(...)
        else:
            # 旧路径: 单一 80% 阈值
            if spend >= 80% AND spend < max_budget:
                # 发送邮件
        return  # ← 直接返回
    
    # ⚠️ type == "token_budget" 会落到这里！
    # 没有任何处理逻辑，直接返回
    # 什么都不做！
```

### ⚠️ 关键发现：邮件不处理 `type="token_budget"`

| 传入的 `type` | Slack 行为 | 邮件行为 |
|--------------|-----------|---------|
| `"soft_budget"` | 重新判断事件类型 | ✅ 处理 `soft_budget_crossed` |
| `"max_budget_alert"` | 重新判断事件类型 | ✅ 处理多阈值或单一 80% |
| `"token_budget"` | 重新判断事件类型 | ❌ **不处理！直接返回** |

**这意味着**：
- Slack 会收到 `threshold_crossed`（15%/5%）和 `budget_crossed`（硬预算）
- 邮件**不会**收到这些，除非通过 `max_budget_alert` 的多阈值配置

---

## 完整状态流转图

### 场景配置

假设：
- `soft_budget = $70`
- `max_budget = $100`
- `max_budget_alert_emails = {"50": [...], "75": [...], "90": [...]}`
- 启用 Slack 和邮件告警
- 请求是 LLM 路由

### 状态流转表

| spend | 第 1 步 | 第 2 步 | 第 3 步 | Slack 事件 | 邮件事件 | 最终结果 |
|-------|--------|--------|--------|-----------|---------|---------|
| **$0** | 不触发 | 触发 `token_budget` (spend=0 < 100) | 不触发 | ❌ 无 | ❌ 无 | 请求正常 |
| **$50** | 触发 `max_budget_alert` (达到 50%) | 触发 `token_budget` (spend=50 < 100) | 不触发 | ❌ 无 (percent_left=50% > 15%) | ✅ `50%` | 请求正常 |
| **$70** | 触发 `max_budget_alert` (已达到 50%) | 触发 `token_budget` (spend=70 < 100) | 触发 `soft_budget` | ✅ `soft_budget_crossed` | ✅ `soft_budget_crossed` | 请求正常 |
| **$75** | 触发 `max_budget_alert` (达到 75%) | 触发 `token_budget` (spend=75 < 100) | 触发 `soft_budget` | ❌ 无 (percent_left=25% > 15%) | ✅ `75%` | 请求正常 |
| **$85** | 触发 `max_budget_alert` (已达到 75%) | 触发 `token_budget` (spend=85 < 100) | 触发 `soft_budget` | ✅ `threshold_crossed` (剩余 15%) | ❌ 无 (85% 不是配置的阈值) | 请求正常 |
| **$90** | 触发 `max_budget_alert` (达到 90%) | 触发 `token_budget` (spend=90 < 100) | 触发 `soft_budget` | ❌ 无 (threshold_crossed 已发送) | ✅ `90%` | 请求正常 |
| **$95** | 触发 `max_budget_alert` (已达到 90%) | 触发 `token_budget` (spend=95 < 100) | 触发 `soft_budget` | ❌ 无 (threshold_crossed 已发送) | ❌ 无 | 请求正常 |
| **$100** | 触发 `max_budget_alert` (已达到 90%) | 触发 `token_budget` (spend=100 >= 100) | ❌ **不执行** | ✅ `budget_crossed` | ⚠️ 看说明 | HTTP 429 |

### 关键状态说明

#### $50 时
- **第 1 步**: `spend=$50 >= max_budget*50% = $50`，触发 `type="max_budget_alert"`
- **第 2 步**: `spend=$50 < max_budget=$100`，触发 `type="token_budget"`，不抛异常
- **第 3 步**: `spend=$50 < soft_budget=$70`，不触发
- **Slack**: 收到 `type="max_budget_alert"` 和 `type="token_budget"`，但 `percent_left=50% > 15%`，不触发任何事件
- **邮件**: 收到 `type="max_budget_alert"`，处理 `50%` 阈值，发送邮件

#### $70 时
- **第 3 步**: `spend=$70 >= soft_budget=$70`，触发 `type="soft_budget"`
- **Slack**: 收到 `type="soft_budget"`，`spend >= soft_budget`，触发 `soft_budget_crossed`
- **邮件**: 收到 `type="soft_budget"`，处理 `soft_budget_crossed`，发送邮件

#### $85 时
- **第 2 步**: `spend=$85 < max_budget=$100`，触发 `type="token_budget"`
- **Slack**: 收到 `type="token_budget"`，`percent_left=(100-85)/100=0.15`，触发 `threshold_crossed`
- **邮件**: 收到 `type="token_budget"`，**不处理**，什么都不做

#### $100 时（硬预算）
- **第 1 步**: `spend=$100 >= max_budget*90% = $90`，触发 `type="max_budget_alert"`
  - **新路径**: 多阈值配置，会处理已达到的阈值（50%/75%/90%）
  - **旧路径**: `spend=$100 NOT < max_budget=$100`，**不触发**
- **第 2 步**: `spend=$100 >= max_budget=$100`，触发 `type="token_budget"`，**抛出 BudgetExceededError**
- **第 3 步**: **不执行**（因为第 2 步抛异常）
- **Slack**: 收到 `type="max_budget_alert"` 和 `type="token_budget"`
  - `type="token_budget"` 时 `spend >= max_budget`，触发 `budget_crossed`
- **邮件**: 收到 `type="max_budget_alert"`
  - 新路径: 处理已达到的阈值（50%/75%/90%），但缓存可能已存在
  - 旧路径: 不触发（因为 `spend < max_budget` 不成立）

---

## 关键代码位置索引

### 调用顺序

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 预算检查真实调用顺序 | `litellm/proxy/auth/user_api_key_auth.py` | 1360-1384 |
| LLM 路由判断 | `litellm/proxy/auth/user_api_key_auth.py` | 1372 |

### 检查函数

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| `_virtual_key_max_budget_alert_check()` | `litellm/proxy/auth/auth_checks.py` | 3171-3263 |
| `_virtual_key_max_budget_check()` | `litellm/proxy/auth/auth_checks.py` | 2990-3047 |
| `_virtual_key_soft_budget_check()` | `litellm/proxy/auth/auth_checks.py` | 3086-3123 |
| 预算阈值配置合并 | `litellm/proxy/auth/auth_checks.py` | 3147-3168 |

### 告警处理

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| Slack 事件判断 | `litellm/integrations/SlackAlerting/slack_alerting.py` | 639-699 |
| 邮件预算告警处理 | `enterprise/.../send_emails/base_email.py` | 414-668 |
| 邮件多阈值处理 | `enterprise/.../send_emails/base_email.py` | 589-668 |

---

## 最终结论汇总

### 1. 真实调用顺序

```
1. _virtual_key_max_budget_alert_check()  ← 总是先执行，不抛异常
2. _virtual_key_max_budget_check()        ← 仅 LLM 路由，可能抛异常
3. _virtual_key_soft_budget_check()       ← 最后执行，不抛异常
```

### 2. 关键影响关系

- 如果 `_virtual_key_max_budget_check()` 抛出 `BudgetExceededError`，`_virtual_key_soft_budget_check()` **不会执行**
- `_virtual_key_max_budget_alert_check()` 总是会执行（因为在最前面且不抛异常）

### 3. 告警类型与渠道响应

| 触发函数 | 传入的 `type` | Slack | 邮件 |
|---------|--------------|-------|------|
| `_virtual_key_max_budget_alert_check()` | `"max_budget_alert"` | 重新判断事件 | ✅ 处理多阈值或单一 80% |
| `_virtual_key_max_budget_check()` | `"token_budget"` | 重新判断事件 | ❌ **不处理** |
| `_virtual_key_soft_budget_check()` | `"soft_budget"` | 重新判断事件 | ✅ 处理 |

### 4. Slack 事件类型

| 条件 | Slack 事件 |
|------|-----------|
| `spend >= soft_budget` | `soft_budget_crossed` |
| `spend >= max_budget` | `budget_crossed` |
| `percent_left <= 0.15` (剩余 15%) | `threshold_crossed` |
| `percent_left <= 0.05` (剩余 5%) | `threshold_crossed` ⚠️ |

⚠️ **注意**: 15% 和 5% 共享同一个 `event="threshold_crossed"`，只触发一次。

### 5. 邮件事件类型

| 条件 | 邮件事件 |
|------|---------|
| `type="soft_budget"` AND `spend >= soft_budget` | `soft_budget_crossed` |
| `type="max_budget_alert"` 新路径 | `max_budget_alert:{threshold}`（每个阈值独立）|
| `type="max_budget_alert"` 旧路径 | `max_budget_alert`（单一 80%，且 `spend < max_budget`）|

### 6. 硬预算时的行为

| 渠道 | 行为 | 原因 |
|------|------|------|
| **Slack** | ✅ 触发 `budget_crossed` | `type="token_budget"` 时 `spend >= max_budget` |
| **邮件（新路径）** | ⚠️ 可能触发已达到的阈值 | 多阈值配置，但 100% 需要显式配置 |
| **邮件（旧路径）** | ❌ 不触发 | 有 `spend < max_budget` 条件检查 |

### 7. 配置建议

1. **如果需要邮件在硬预算时告警**：
   - 使用新路径，配置 `"100": ["..."]` 在 `max_budget_alert_emails` 中
   - 或同时启用 Slack，Slack 会发送 `budget_crossed`

2. **如果需要邮件在 15%/5% 时告警**：
   - 不能依赖 `type="token_budget"`（邮件不处理）
   - 需要在 `max_budget_alert_emails` 中显式配置 `"85": [...]` 或 `"95": [...]`

3. **软预算告警**：
   - 如果硬预算可能先于软预算触发，`_virtual_key_soft_budget_check()` 可能不会执行
   - 建议 `soft_budget < max_budget`，且考虑是否需要在 `max_budget_alert_emails` 中也配置对应阈值
