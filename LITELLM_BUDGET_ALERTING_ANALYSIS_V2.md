# LiteLLM 预算告警机制深度分析报告（修正版）

## 文档修订说明

**基于代码的精确分析修正了以下错误结论：**

| 之前错误结论 | 修正后正确结论 |
|-------------|---------------|
| Slack 的 5% 和 15% 阈值分别触发独立事件 | Slack 的 5%/15% 只触发 **一个** `threshold_crossed` 事件，先到先触发，触发后不再触发 |
| 邮件和 Slack 共享去重机制 | **完全独立**的缓存 key 格式，去重边界完全不同 |
| `max_budget_alert` 是单一阈值 | 邮件的 `max_budget_alert` 支持 **多阈值独立去重**（50%/75%/80%/90% 等各自独立）|
| 硬预算时邮件也会告警 | 邮件的 `max_budget_alert` 旧路径有 `spend < max_budget` 检查，**硬预算时不会触发** |

---

## 目录

1. [预算检测触发点总览](#预算检测触发点总览)
2. [各检测函数的触发类型与条件](#各检测函数的触发类型与条件)
3. [Slack 告警的事件判断逻辑](#slack-告警的事件判断逻辑)
4. [邮件告警的多阈值机制](#邮件告警的多阈值机制)
5. [去重边界的精确对比](#去重边界的精确对比)
6. [硬预算前后的行为差异](#硬预算前后的行为差异)
7. [完整触发顺序与状态流转](#完整触发顺序与状态流转)
8. [关键代码位置索引](#关键代码位置索引)

---

## 预算检测触发点总览

### 认证阶段的检测函数执行顺序

在 `auth_checks.py` 中，预算检测按以下顺序执行：

```
API 请求进入
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. _virtual_key_soft_budget_check()                            │
│     └─ 触发类型: type="soft_budget"                              │
│     └─ 条件: soft_budget 已设置 AND spend >= soft_budget        │
│     └─ 行为: 仅告警，不阻止请求                                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. _virtual_key_max_budget_alert_check()                       │
│     └─ 触发类型: type="max_budget_alert"                         │
│     └─ 条件:                                                      │
│        新路径: spend >= 最低配置阈值 (如 50%) AND               │
│               配置了 max_budget_alert_emails                     │
│        旧路径: spend >= 80% AND spend < max_budget             │
│     └─ 行为: 仅告警，不阻止请求                                   │
│     └─ ⚠️ 重要: 旧路径有 spend < max_budget 检查                 │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. _virtual_key_max_budget_check()                             │
│     └─ 触发类型: type="token_budget"                             │
│     └─ 条件: max_budget 已设置                                   │
│     └─ 行为:                                                      │
│        - spend < max_budget: 检查剩余阈值 (15%/5%) → 告警       │
│        - spend >= max_budget: 告警 + 抛出 BudgetExceededError  │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼ (如果 spend >= max_budget)
┌─────────────────────────────────────────────────────────────────┐
│  抛出 BudgetExceededError                                        │
│  HTTP 429 响应                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 其他层级的检测函数

除了 Key 级别的检测，还有以下层级：

| 检测函数 | 触发类型 | 层级 |
|---------|---------|------|
| `_team_soft_budget_check()` | `type="soft_budget"` | Team |
| `_check_team_member_budget()` | `type="team_budget"` (内部) | Team Member |
| `_project_soft_budget_check()` | `type="soft_budget"` | Project |
| `_project_max_budget_check()` | `type="project_budget"` | Project |
| `_organization_max_budget_check()` | `type="organization_budget"` | Organization |

---

## 各检测函数的触发类型与条件

### 1. `_virtual_key_soft_budget_check()`

**文件位置**: `litellm/proxy/auth/auth_checks.py:3086-3122`

```python
async def _virtual_key_soft_budget_check(
    valid_token: UserAPIKeyAuth,
    proxy_logging_obj: ProxyLogging,
    user_obj: Optional[LiteLLM_UserTable] = None,
):
    # 触发条件
    if valid_token.soft_budget and valid_token.spend >= valid_token.soft_budget:
        call_info = CallInfo(
            token=valid_token.token,
            spend=valid_token.spend,
            max_budget=valid_token.max_budget,
            soft_budget=valid_token.soft_budget,
            ...
            event_group=Litellm_EntityType.KEY,
        )
        
        # 触发告警任务
        asyncio.create_task(
            proxy_logging_obj.budget_alerts(
                type="soft_budget",      # ⬅️ 注意这个 type
                user_info=call_info,
            )
        )
```

### 2. `_virtual_key_max_budget_alert_check()`

**文件位置**: `litellm/proxy/auth/auth_checks.py:3171-3263`

#### 新路径（多阈值配置）

```python
# 配置示例 (metadata.max_budget_alert_emails):
# {
#     "50": ["finance@co.com"],
#     "75": ["finance@co.com", "bu_lead@co.com"],
#     "90": ["cto@co.com"],
# }

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
        type="max_budget_alert",      # ⬅️ 注意这个 type
        user_info=call_info,           # 包含 max_budget_alert_emails
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
    and valid_token.spend < valid_token.max_budget  # ⚠️ 必须小于 max_budget
):
    asyncio.create_task(
        proxy_logging_obj.budget_alerts(
            type="max_budget_alert",
            user_info=call_info,
        )
    )
```

**重要结论**: 旧路径的 `max_budget_alert` **不会在硬预算时触发**，因为有 `spend < max_budget` 的条件检查。

### 3. `_virtual_key_max_budget_check()`

**文件位置**: `litellm/proxy/auth/auth_checks.py:2996-3047`

```python
# 读取当前花费
spend = await get_current_spend(
    counter_key=f"spend:key:{valid_token.token}",
    fallback_spend=valid_token.spend or 0.0,
)

# 构建 CallInfo
call_info = CallInfo(
    token=valid_token.token,
    spend=spend,
    max_budget=valid_token.max_budget,
    soft_budget=valid_token.soft_budget,
    ...
    event_group=Litellm_EntityType.KEY,
)

# 触发告警 (无论是否超过都会触发，内部会判断具体事件)
asyncio.create_task(
    proxy_logging_obj.budget_alerts(
        type="token_budget",      # ⬅️ 注意这个 type
        user_info=call_info,
    )
)

# 超过预算则抛出异常
if spend >= valid_token.max_budget:
    raise litellm.BudgetExceededError(
        current_cost=spend,
        max_budget=valid_token.max_budget,
    )
```

---

## Slack 告警的事件判断逻辑

### 核心方法: `_get_event_and_event_message()`

**文件位置**: `litellm/integrations/SlackAlerting/slack_alerting.py:639-699`

无论传入的 `type` 是什么（`token_budget`、`soft_budget`、`max_budget_alert` 等），Slack 都会根据实际的 `spend` 和 `budget` 值重新判断事件类型。

```python
def _get_event_and_event_message(
    self,
    user_info: CallInfo,
    event: Optional[Literal["budget_crossed", "threshold_crossed", 
                             "soft_budget_crossed", "projected_limit_exceeded"]],
    event_message: str,
) -> Tuple[Optional[str], str]:
    
    percent_left: float = self._get_percent_of_max_budget_left(user_info)
    # percent_left = (max_budget - spend) / max_budget
    
    #####################################################################
    # 第一优先级: SOFT BUDGET 检查
    #####################################################################
    if user_info.soft_budget is not None:
        if user_info.spend >= user_info.soft_budget:
            event = "soft_budget_crossed"
            event_message += f"Total Soft Budget:`{user_info.soft_budget}`"
    
    #####################################################################
    # 第二优先级: MAX BUDGET 检查
    #####################################################################
    if user_info.max_budget is not None:
        # a. 已超过硬预算
        if user_info.spend >= user_info.max_budget:
            event = "budget_crossed"
            event_message += f"Budget Crossed\n Total Budget:`{user_info.max_budget}`"
        
        # b. 剩余 5% (percent_left <= 0.05)
        elif percent_left <= SLACK_ALERTING_THRESHOLD_5_PERCENT:  # 0.05
            event = "threshold_crossed"
            event_message += "5% Threshold Crossed "
        
        # c. 剩余 15% (percent_left <= 0.15)
        elif percent_left <= SLACK_ALERTING_THRESHOLD_15_PERCENT:  # 0.15
            event = "threshold_crossed"
            event_message += "15% Threshold Crossed"
    
    return event, event_message
```

### ⚠️ 关键发现: 15% 和 5% 只触发一次

注意代码中的 `elif` 结构：

```python
elif percent_left <= 0.05:
    event = "threshold_crossed"
    event_message += "5% Threshold Crossed "
elif percent_left <= 0.15:
    event = "threshold_crossed"
    event_message += "15% Threshold Crossed"
```

**这意味着**:
1. 当 `percent_left` 从 20% → 12%（剩余 12%），触发 `threshold_crossed`，消息是 "15% Threshold Crossed"
2. 当 `percent_left` 继续降到 3%（剩余 3%），**不会重新触发**，因为：
   - 缓存 key 是 `budget_alerts:threshold_crossed:{id}`
   - 同一个 `event` 类型 + 同一个实体 ID，在 TTL（24小时）内只发送一次

**之前的错误结论**: "15% 和 5% 会分别触发两次告警"

**修正后**: 15% 和 5% 共享同一个 `event="threshold_crossed"`，**只触发一次**，先到哪个阈值就发送哪个阈值的消息。

### Slack 缓存 Key 格式

**文件位置**: `litellm/integrations/SlackAlerting/slack_alerting.py:614-634`

```python
# 缓存 key 格式
_cache_key = "budget_alerts:{}:{}".format(event, _id)
# 例如:
# - budget_alerts:soft_budget_crossed:sk-xxx123
# - budget_alerts:threshold_crossed:sk-xxx123
# - budget_alerts:budget_crossed:sk-xxx123

# 检查缓存
result = await _cache.async_get_cache(key=_cache_key)
if result is None:
    # 发送告警
    await self.send_alert(...)
    
    # 标记已发送，TTL = 24 小时
    await _cache.async_set_cache(
        key=_cache_key,
        value="SENT",
        ttl=self.alerting_args.budget_alert_ttl,  # 24 * 60 * 60 秒
    )
```

### Slack 事件类型与触发条件汇总

| Event 类型 | 触发条件 | 缓存 Key 示例 |
|-----------|---------|-------------|
| `soft_budget_crossed` | `spend >= soft_budget` | `budget_alerts:soft_budget_crossed:{id}` |
| `threshold_crossed` | `percent_left <= 0.15` (剩余 15%) **或** `percent_left <= 0.05` (剩余 5%) | `budget_alerts:threshold_crossed:{id}` |
| `budget_crossed` | `spend >= max_budget` | `budget_alerts:budget_crossed:{id}` |
| `projected_limit_exceeded` | `type == "projected_limit_exceeded"` (特殊类型) | `budget_alerts:projected_limit_exceeded:{id}` |

---

## 邮件告警的多阈值机制

### 邮件的 `budget_alerts()` 方法

**文件位置**: `enterprise/litellm_enterprise/enterprise_callbacks/send_emails/base_email.py:414-669`

邮件告警与 Slack 完全不同，它根据传入的 `type` 走不同的分支：

```python
async def budget_alerts(
    self,
    type: Literal["token_budget", "soft_budget", "max_budget_alert", ...],
    user_info: CallInfo,
):
    #####################################################################
    # 分支 1: type == "soft_budget"
    #####################################################################
    if type == "soft_budget":
        if user_info.soft_budget is not None and user_info.spend >= user_info.soft_budget:
            # 缓存 key
            _cache_key = f"email_budget_alerts:soft_budget_crossed:{_id}"
            
            result = await _cache.async_get_cache(key=_cache_key)
            if result is None:
                # 发送邮件
                await self.send_soft_budget_alert_email(webhook_event)
                
                # 标记已发送
                await _cache.async_set_cache(
                    key=_cache_key,
                    value="SENT",
                    ttl=EMAIL_BUDGET_ALERT_TTL,
                )
        return
    
    #####################################################################
    # 分支 2: type == "max_budget_alert"
    #####################################################################
    if type == "max_budget_alert":
        if user_info.max_budget_alert_emails:
            # 新路径: 多阈值独立去重
            await self._handle_multi_threshold_max_budget_alert(
                user_info=user_info, _cache=_cache
            )
            return
        
        # 旧路径: 单一 80% 阈值
        alert_threshold = user_info.max_budget * EMAIL_BUDGET_ALERT_MAX_SPEND_ALERT_PERCENTAGE
        
        # ⚠️ 关键条件: spend < max_budget
        if (
            user_info.spend >= alert_threshold
            and user_info.spend < user_info.max_budget  # ⬅️ 不会在硬预算时触发
        ):
            _cache_key = f"email_budget_alerts:max_budget_alert:{_id}"
            # 发送逻辑...
```

### 多阈值处理: `_handle_multi_threshold_max_budget_alert()`

**文件位置**: `enterprise/litellm_enterprise/enterprise_callbacks/send_emails/base_email.py:589-668`

```python
async def _handle_multi_threshold_max_budget_alert(
    self,
    user_info: CallInfo,
    _cache: DualCache,
):
    """
    遍历 max_budget_alert_emails 中配置的所有阈值，
    每个阈值独立检查缓存，独立发送给配置的收件人。
    """
    # 配置示例:
    # user_info.max_budget_alert_emails = {
    #     "50": ["finance@co.com"],
    #     "75": ["finance@co.com", "bu_lead@co.com"],
    #     "90": ["cto@co.com"],
    # }
    
    for threshold_str, raw_emails in user_info.max_budget_alert_emails.items():
        try:
            threshold_pct = int(threshold_str)  # "50" → 50
        except (ValueError, TypeError):
            continue  # 跳过非数字阈值
        
        # 检查是否达到该阈值
        threshold_amount = user_info.max_budget * (threshold_pct / 100.0)
        if user_info.spend < threshold_amount:
            continue  # 未达到，跳过
        
        # ⚠️ 关键: 每个阈值有独立的缓存 key
        _id = user_info.token or user_info.user_id or "default_id"
        _cache_key = (
            f"email_budget_alerts:max_budget_alert:{threshold_pct}:{_id}"
        )
        # 例如:
        # - email_budget_alerts:max_budget_alert:50:sk-xxx123
        # - email_budget_alerts:max_budget_alert:75:sk-xxx123
        # - email_budget_alerts:max_budget_alert:90:sk-xxx123
        
        # 检查缓存
        result = await _cache.async_get_cache(key=_cache_key)
        if result is not None:
            continue  # 该阈值已发送过，跳过
        
        # 解析收件人 + 自动包含 Key 所有者
        emails = _parse_email_list(raw_emails)
        if user_info.user_email:
            emails.append(user_info.user_email)  # 自动添加所有者
        recipient_emails = list(set(emails))  # 去重
        
        # 发送邮件
        await self.send_max_budget_alert_email(
            webhook_event,
            threshold_pct=threshold_pct,
            recipient_emails=recipient_emails,
        )
        
        # 标记该阈值已发送
        await _cache.async_set_cache(
            key=_cache_key,
            value="SENT",
            ttl=EMAIL_BUDGET_ALERT_TTL,
        )
```

### 邮件缓存 Key 格式汇总

| `type` | 场景 | 缓存 Key 格式 | 示例 |
|--------|------|-------------|------|
| `soft_budget` | 软预算超限 | `email_budget_alerts:soft_budget_crossed:{id}` | `email_budget_alerts:soft_budget_crossed:sk-xxx123` |
| `max_budget_alert` | 新路径（多阈值） | `email_budget_alerts:max_budget_alert:{threshold}:{id}` | `email_budget_alerts:max_budget_alert:75:sk-xxx123` |
| `max_budget_alert` | 旧路径（单一 80%） | `email_budget_alerts:max_budget_alert:{id}` | `email_budget_alerts:max_budget_alert:sk-xxx123` |

### ⚠️ 重要发现

1. **多阈值独立去重**: 邮件的新路径中，50%、75%、90% 各自有独立的缓存 key，**会分别触发**
2. **硬预算时不会触发旧路径**: 旧路径有 `spend < max_budget` 检查，所以达到硬预算时 `max_budget_alert` 不会触发
3. **收件人自动合并**: 每个阈值的收件人列表会自动添加 `user_info.user_email`（Key 所有者）

---

## 去重边界的精确对比

### Slack vs 邮件：缓存 Key 对比

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SLACK 缓存 Key 格式                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  格式: budget_alerts:{event}:{id}                                          │
│                                                                             │
│  其中 event 只能是:                                                         │
│    - "soft_budget_crossed"    (spend >= soft_budget)                      │
│    - "threshold_crossed"      (剩余 15% 或 5%，⚠️ 共享同一个 event)        │
│    - "budget_crossed"         (spend >= max_budget)                        │
│    - "projected_limit_exceeded" (特殊类型)                                 │
│                                                                             │
│  示例:                                                                      │
│    budget_alerts:soft_budget_crossed:sk-xxx123                             │
│    budget_alerts:threshold_crossed:sk-xxx123    (⚠️ 15% 和 5% 都用这个)   │
│    budget_alerts:budget_crossed:sk-xxx123                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                           邮件 缓存 Key 格式                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  格式 1 (soft_budget):                                                      │
│    email_budget_alerts:soft_budget_crossed:{id}                            │
│                                                                             │
│  格式 2 (max_budget_alert 新路径, ⚠️ 每个阈值独立):                         │
│    email_budget_alerts:max_budget_alert:{threshold_pct}:{id}               │
│                                                                             │
│  格式 3 (max_budget_alert 旧路径):                                          │
│    email_budget_alerts:max_budget_alert:{id}                               │
│                                                                             │
│  示例:                                                                      │
│    email_budget_alerts:soft_budget_crossed:sk-xxx123                       │
│    email_budget_alerts:max_budget_alert:50:sk-xxx123   (50% 独立)         │
│    email_budget_alerts:max_budget_alert:75:sk-xxx123   (75% 独立)         │
│    email_budget_alerts:max_budget_alert:90:sk-xxx123   (90% 独立)         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 去重边界对比表

| 维度 | Slack | 邮件（新路径） |
|------|-------|--------------|
| **缓存 Key 前缀** | `budget_alerts:` | `email_budget_alerts:` |
| **15% 与 5%** | 共享 `threshold_crossed`，只触发一次 | 不适用（邮件没有 15%/5% 逻辑）|
| **多阈值 (50%/75%/90%)** | 不适用（Slack 没有这个机制） | 每个阈值独立 Key，分别触发 |
| **soft_budget** | `soft_budget_crossed` | `soft_budget_crossed` |
| **max_budget** | `budget_crossed` | 旧路径不触发（有 `spend < max_budget`）|
| **TTL** | 24 小时 (`budget_alert_ttl`) | 24 小时 (`EMAIL_BUDGET_ALERT_TTL`) |
| **缓存存储** | `SlackAlerting.internal_usage_cache` | `BaseEmailLogger.internal_usage_cache` |

### ⚠️ 关键结论：完全独立的去重

**Slack 和邮件的去重是完全独立的**，因为：
1. **缓存 Key 前缀不同**: `budget_alerts:` vs `email_budget_alerts:`
2. **事件判断逻辑不同**: Slack 用 `_get_event_and_event_message()` 重新判断，邮件按 `type` 走不同分支
3. **缓存实例不同**: 各自使用自己的 `internal_usage_cache`

这意味着：
- 同一个预算事件，**Slack 发送过不影响邮件**，反之亦然
- 可以配置为：只启用 Slack，或只启用邮件，或同时启用

---

## 硬预算前后的行为差异

### 状态定义

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  预算状态定义 (以 max_budget = $100 为例)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  spend = $0                    spend = $80              spend = $100+      │
│    │                              │                        │               │
│    ▼                              ▼                        ▼               │
│  ┌──────────────────────────────────────────────────────────────────────┐ │
│  │                         预算使用进度条                                  │ │
│  │  [──────────────────────────────────────────────────────────────────] │ │
│  │   0%              50%              80%         90%        100%       │ │
│  │                                                         │               │ │
│  │  硬预算前 (spend < max_budget)          │  硬预算时 (spend >= max_budget)│ │
│  │                                                         │               │ │
│  └──────────────────────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 硬预算前 (spend < max_budget)

#### 可能触发的告警

| 检测函数 | `type` | Slack 可能事件 | 邮件可能事件 |
|---------|--------|--------------|-------------|
| `_virtual_key_soft_budget_check()` | `soft_budget` | `soft_budget_crossed` (如果 spend >= soft_budget) | `soft_budget_crossed` |
| `_virtual_key_max_budget_alert_check()` | `max_budget_alert` | 走 `_get_event_and_event_message()`，可能是: `soft_budget_crossed` / `threshold_crossed` | 多阈值：50%/75%/80%/90% 分别触发 |
| `_virtual_key_max_budget_check()` | `token_budget` | `threshold_crossed` (如果剩余 15% 或 5%) | 邮件不处理 `token_budget` 类型（看下面说明）|

#### ⚠️ 邮件对 `type="token_budget"` 的处理

看邮件的 `budget_alerts()` 代码：

```python
async def budget_alerts(self, type, user_info):
    if type == "soft_budget":
        # 处理 soft_budget
        ...
        return
    
    if type == "max_budget_alert":
        # 处理 max_budget_alert
        ...
        return
    
    # ⚠️ type="token_budget" 会落到这里，没有任何处理逻辑！
    # 函数直接返回，什么都不做
```

**重要结论**: 邮件的 `budget_alerts()` 方法**只处理** `type="soft_budget"` 和 `type="max_budget_alert"`，**不处理** `type="token_budget"`。

这意味着：
- Slack 会在 `token_budget` 时触发 `threshold_crossed`（15%/5%）或 `budget_crossed`
- 邮件**不会**收到这些告警，除非通过 `max_budget_alert` 的多阈值配置

#### 行为特点

1. **不阻止请求**: 所有告警都是通过 `asyncio.create_task()` 异步触发，不阻塞主请求流程
2. **阈值判断**:
   - Slack: 15% 和 5% 共享 `threshold_crossed`，只触发一次
   - 邮件: 50%/75%/80%/90% 各自独立，分别触发
3. **缓存去重**: 24 小时内同一事件不重复发送

### 硬预算时 (spend >= max_budget)

#### 触发流程

```
spend >= max_budget
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  _virtual_key_max_budget_check() 执行                           │
│  ├─ 构建 CallInfo(spend, max_budget, ...)                      │
│  ├─ asyncio.create_task(                                        │
│  │       proxy_logging_obj.budget_alerts(                       │
│  │           type="token_budget",  ← 注意这个 type             │
│  │           user_info=call_info,                                │
│  │       )                                                       │
│  │   )                                                           │
│  └─ 抛出 BudgetExceededError                                     │
└─────────────────────────────────────────────────────────────────┘
    │
    ├──────────────────────────────┐
    │                              │
    ▼                              ▼
┌──────────────┐          ┌──────────────────────────┐
│   Slack 处理  │          │      邮件处理            │
├──────────────┤          ├──────────────────────────┤
│              │          │                          │
│ type="token_ │          │ type="token_budget"     │
│ budget"      │          │ 不匹配任何分支           │
│              │          │ 直接返回                 │
│ ▼            │          │                          │
│ _get_event_  │          │ ⚠️ 邮件不会发送！        │
│ and_event_   │          │                          │
│ message()    │          │                          │
│              │          │                          │
│ spend >=     │          │                          │
│ max_budget   │          │                          │
│              │          │                          │
│ ▼            │          │                          │
│ event="budg_ │          │                          │
│ et_crossed"  │          │                          │
│              │          │                          │
│ 发送 Slack   │          │                          │
│ 消息         │          │                          │
└──────────────┘          └──────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  抛出 BudgetExceededError                                        │
│  - 后续请求被阻止                                                 │
│  - HTTP 429 响应                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### 硬预算时的告警行为

| 渠道 | 是否触发 | 触发事件 | 说明 |
|------|---------|---------|------|
| **Slack** | ✅ 触发 | `budget_crossed` | 通过 `type="token_budget"` → `_get_event_and_event_message()` 判断 |
| **邮件** | ❌ 不触发 | - | 邮件不处理 `type="token_budget"` 类型 |
| **邮件 (max_budget_alert 旧路径)** | ❌ 不触发 | - | 有 `spend < max_budget` 条件检查 |

### 硬预算前后行为对比表

| 维度 | 硬预算前 (spend < max_budget) | 硬预算时 (spend >= max_budget) |
|------|-------------------------------|--------------------------------|
| **请求处理** | ✅ 正常处理 | ❌ 抛出 `BudgetExceededError`，阻止请求 |
| **HTTP 响应** | 200/正常 | 429 Too Many Requests |
| **Slack 可能事件** | `soft_budget_crossed`, `threshold_crossed` | `budget_crossed` |
| **邮件可能事件** | `soft_budget_crossed`, `max_budget_alert` (50%/75%/80%/90%) | ⚠️ **无**（邮件不处理 `token_budget`）|
| **异步告警** | `asyncio.create_task()` 不阻塞 | 同样异步触发，但请求已被阻止 |
| **缓存去重** | 24 小时 TTL | 同样 24 小时 TTL |

### ⚠️ 重要发现：硬预算时邮件不会告警

如果需要在硬预算时发送邮件通知，有两种方式：

1. **同时启用 Slack**: Slack 会发送 `budget_crossed` 事件
2. **配置 Webhook**: Webhook 会在所有预算事件时触发
3. **通过其他机制**: 比如监控 `BudgetExceededError` 日志

---

## 完整触发顺序与状态流转

### 场景示例

假设配置：
- `soft_budget = $70`
- `max_budget = $100`
- `max_budget_alert_emails = {"50": ["a@co.com"], "75": ["b@co.com"], "90": ["c@co.com"]}`
- 同时启用 Slack 和邮件告警

### 状态流转表

| spend | 状态 | 触发函数 | `type` | Slack 事件 | 邮件事件 |
|-------|------|---------|--------|-----------|---------|
| $0 | 正常 | 无 | - | - | - |
| $50 | 正常 | `_virtual_key_max_budget_alert_check()` | `max_budget_alert` | `threshold_crossed` (剩余 50% > 15%) | `max_budget_alert:50` |
| $70 | 软预算 | `_virtual_key_soft_budget_check()` | `soft_budget` | `soft_budget_crossed` | `soft_budget_crossed` |
| $75 | 正常 | `_virtual_key_max_budget_alert_check()` | `max_budget_alert` | `threshold_crossed` (剩余 25% > 15%) | `max_budget_alert:75` |
| $85 | 剩余 15% | `_virtual_key_max_budget_check()` | `token_budget` | `threshold_crossed` (剩余 15%) | ⚠️ 无（邮件不处理 `token_budget`）|
| $90 | 正常 | `_virtual_key_max_budget_alert_check()` | `max_budget_alert` | `threshold_crossed` (剩余 10% < 15%，但可能已发送过) | `max_budget_alert:90` |
| $95 | 剩余 5% | `_virtual_key_max_budget_check()` | `token_budget` | ⚠️ 无（`threshold_crossed` 已在 15% 时发送过）| ⚠️ 无 |
| $100 | 硬预算 | `_virtual_key_max_budget_check()` | `token_budget` | `budget_crossed` | ⚠️ 无 |

### 详细时序图

```
时间轴 ─────────────────────────────────────────────────────────────────►

spend = $0
│
│  请求正常处理，无任何告警
│
▼

spend = $50 (达到 50% max_budget)
│
├─ _virtual_key_max_budget_alert_check() 触发
│   ├─ type = "max_budget_alert"
│   ├─ max_budget_alert_emails = {"50": [...], "75": [...], "90": [...]}
│   └─ asyncio.create_task(proxy_logging_obj.budget_alerts(...))
│
├─ Slack 处理:
│   ├─ budget_alert_class = get_budget_alert_type("max_budget_alert")
│   │   └─ TokenBudgetAlert (因为 "max_budget_alert" 映射到 TokenBudgetAlert)
│   ├─ _get_event_and_event_message():
│   │   ├─ percent_left = (100 - 50) / 100 = 0.5 (50%)
│   │   ├─ 0.5 > 0.15，不满足 threshold_crossed
│   │   └─ ⚠️ 等等，让我再仔细看代码...
│   │
│   │  实际上:
│   │  if type == "max_budget_alert":
│   │      映射到的是 TokenBudgetAlert (看 budget_alert_types.py:104)
│   │      "max_budget_alert": TokenBudgetAlert()
│   │
│   │  但 _get_event_and_event_message() 是根据实际 spend 判断的:
│   │  spend = $50, max_budget = $100
│   │  percent_left = 0.5
│   │  0.5 > 0.15，所以不会触发 threshold_crossed
│   │  也没有 soft_budget 或 budget_crossed
│   │
│   └─ ⚠️ 结论: event = None，Slack 不会发送！
│
├─ 邮件处理:
│   ├─ type = "max_budget_alert"
│   ├─ 有 max_budget_alert_emails，走新路径
│   ├─ _handle_multi_threshold_max_budget_alert():
│   │   ├─ 遍历 "50", "75", "90"
│   │   ├─ "50": spend = $50 >= $50 (50% of $100) ✓
│   │   │   ├─ 缓存 key: email_budget_alerts:max_budget_alert:50:{id}
│   │   │   ├─ 发送邮件给 ["a@co.com", owner@co.com]
│   │   │   └─ 标记已发送
│   │   ├─ "75": spend = $50 < $75 ✗
│   │   └─ "90": spend = $50 < $90 ✗
│   └─ 只发送 50% 阈值的邮件
│
▼

spend = $70 (达到 soft_budget)
│
├─ _virtual_key_soft_budget_check() 触发
│   ├─ type = "soft_budget"
│   └─ asyncio.create_task(...)
│
├─ Slack 处理:
│   ├─ _get_event_and_event_message():
│   │   ├─ soft_budget = $70, spend = $70
│   │   └─ event = "soft_budget_crossed"
│   ├─ 缓存 key: budget_alerts:soft_budget_crossed:{id}
│   └─ 发送 Slack 消息
│
└─ 邮件处理:
    ├─ type = "soft_budget"
    ├─ spend >= soft_budget ✓
    ├─ 缓存 key: email_budget_alerts:soft_budget_crossed:{id}
    └─ 发送邮件
│
▼

spend = $75 (达到 75% max_budget)
│
├─ _virtual_key_max_budget_alert_check() 触发
│   └─ type = "max_budget_alert"
│
├─ Slack 处理:
│   ├─ percent_left = (100 - 75) / 100 = 0.25 (25%)
│   ├─ 0.25 > 0.15，仍不满足 threshold_crossed
│   └─ event = None，不发送
│
└─ 邮件处理:
    ├─ 遍历阈值:
    │   ├─ "50": 已发送 (缓存存在)
    │   ├─ "75": spend = $75 >= $75 ✓
    │   │   ├─ 发送邮件给 ["b@co.com", owner@co.com]
    │   │   └─ 标记已发送
    │   └─ "90": $75 < $90 ✗
    └─ 发送 75% 阈值的邮件
│
▼

spend = $85 (剩余 15%)
│
├─ _virtual_key_max_budget_check() 触发
│   └─ type = "token_budget"
│
├─ Slack 处理:
│   ├─ percent_left = (100 - 85) / 100 = 0.15 (刚好 15%)
│   ├─ _get_event_and_event_message():
│   │   └─ event = "threshold_crossed", message = "15% Threshold Crossed"
│   ├─ 缓存 key: budget_alerts:threshold_crossed:{id}
│   └─ 发送 Slack 消息
│
└─ 邮件处理:
    ├─ type = "token_budget"
    ├─ 不匹配 "soft_budget" 或 "max_budget_alert"
    └─ ⚠️ 直接返回，不发送邮件！
│
│  同时: _virtual_key_max_budget_alert_check()
│  └─ type = "max_budget_alert"，但 85% 不是配置的阈值 (50/75/90)
│     除非配置了 "80" 或 "85"，否则不会触发
│
▼

spend = $90 (达到 90% max_budget)
│
├─ _virtual_key_max_budget_alert_check() 触发
│   └─ type = "max_budget_alert"
│
├─ Slack 处理:
│   ├─ percent_left = 0.1 (10%)
│   ├─ 0.1 <= 0.15，满足 threshold_crossed
│   ├─ 但缓存 key: budget_alerts:threshold_crossed:{id} 已存在 (85% 时发送过)
│   └─ ⚠️ 不重复发送！
│
└─ 邮件处理:
    ├─ 遍历阈值:
    │   ├─ "50": 已发送
    │   ├─ "75": 已发送
    │   └─ "90": spend = $90 >= $90 ✓
    │       ├─ 发送邮件给 ["c@co.com", owner@co.com]
    │       └─ 标记已发送
    └─ 发送 90% 阈值的邮件
│
▼

spend = $95 (剩余 5%)
│
├─ _virtual_key_max_budget_check() 触发
│   └─ type = "token_budget"
│
├─ Slack 处理:
│   ├─ percent_left = 0.05 (5%)
│   ├─ _get_event_and_event_message():
│   │   └─ event = "threshold_crossed", message = "5% Threshold Crossed"
│   ├─ 但缓存 key: budget_alerts:threshold_crossed:{id} 已存在
│   └─ ⚠️ 不发送！(虽然消息不同，但 event 类型相同)
│
└─ 邮件处理:
    └─ type = "token_budget"，不发送
│
▼

spend = $100 (达到硬预算)
│
├─ _virtual_key_max_budget_check() 触发
│   ├─ type = "token_budget"
│   ├─ asyncio.create_task(...)
│   └─ 抛出 BudgetExceededError
│
├─ Slack 处理:
│   ├─ _get_event_and_event_message():
│   │   ├─ spend = $100 >= max_budget = $100
│   │   └─ event = "budget_crossed"
│   ├─ 缓存 key: budget_alerts:budget_crossed:{id}
│   └─ 发送 Slack 消息: "Budget Crossed"
│
├─ 邮件处理:
│   ├─ type = "token_budget"
│   └─ ⚠️ 不发送！
│
└─ 后续请求:
    └─ 抛出 BudgetExceededError，HTTP 429 响应
```

### 关键观察结论

1. **Slack 的 `max_budget_alert` 类型**:
   - `"max_budget_alert"` 映射到 `TokenBudgetAlert`（看 `budget_alert_types.py:104`）
   - 但事件判断仍走 `_get_event_and_event_message()`
   - 所以只有当 `percent_left <= 0.15` 时才会触发 `threshold_crossed`

2. **邮件的 `max_budget_alert`**:
   - 完全独立的逻辑，按配置的阈值分别触发
   - 每个阈值有独立的缓存 key

3. **`threshold_crossed` 的陷阱**:
   - Slack 中 15% 和 5% 共享同一个 event 类型
   - 如果先到 15%，发送消息是 "15% Threshold Crossed"
   - 后续到 5% 时，虽然消息会变成 "5% Threshold Crossed"，但缓存 key 相同，**不会重新发送**
   - **用户永远看不到 5% 的告警**，除非 15% 时没触发（比如直接从 20% 跳到 3%）

---

## 关键代码位置索引

### 预算检测触发

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| Key 软预算检测 | `litellm/proxy/auth/auth_checks.py` | 3086-3122 |
| Key 预算告警阈值检测 | `litellm/proxy/auth/auth_checks.py` | 3171-3263 |
| Key 硬预算检测 | `litellm/proxy/auth/auth_checks.py` | 2996-3047 |
| Team 软预算检测 | `litellm/proxy/auth/auth_checks.py` | 3464-3542 |
| 预算阈值配置合并 | `litellm/proxy/auth/auth_checks.py` | 3134-3168 |

### Slack 告警

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 预算告警主方法 | `litellm/integrations/SlackAlerting/slack_alerting.py` | 547-637 |
| 事件类型判断 | `litellm/integrations/SlackAlerting/slack_alerting.py` | 639-699 |
| 预算告警类型工厂 | `litellm/integrations/SlackAlerting/budget_alert_types.py` | 1-115 |
| 告警类型常量 | `litellm/types/integrations/slack_alerting.py` | 13-14 |

### 邮件告警

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| 预算告警主方法 | `enterprise/.../send_emails/base_email.py` | 414-587 |
| 多阈值处理 | `enterprise/.../send_emails/base_email.py` | 589-668 |
| SMTP 发送基础函数 | `litellm/proxy/utils.py` | 4702-4773 |

### 代理日志分发

| 功能 | 文件位置 | 行号 |
|------|---------|------|
| ProxyLogging.budget_alerts | `litellm/proxy/utils.py` | 1596-1643 |

---

## 修正要点总结

### 之前报告的主要错误

| 错误结论 | 修正后 | 代码依据 |
|---------|--------|---------|
| Slack 的 15% 和 5% 分别触发 | 共享 `threshold_crossed`，只触发一次 | `slack_alerting.py:692-697` 的 `elif` 结构 |
| 邮件和 Slack 共享去重 | 完全独立的缓存 key 和逻辑 | 前缀不同: `budget_alerts:` vs `email_budget_alerts:` |
| `max_budget_alert` 是单一阈值 | 邮件支持多阈值独立去重 | `base_email.py:589-668` 的循环和独立 key |
| 硬预算时邮件也会告警 | 邮件不处理 `type="token_budget"` | `base_email.py:414-587` 的分支结构 |
| 邮件处理所有预算类型 | 只处理 `soft_budget` 和 `max_budget_alert` | `base_email.py:414-587` |

### 关键设计意图

1. **Slack 的 `threshold_crossed` 设计**:
   - 意图是在"即将耗尽"时提醒一次
   - 不区分 15% 还是 5%，避免告警噪音

2. **邮件的多阈值设计**:
   - 意图是给不同角色在不同阶段发送告警
   - 例如: 50% 通知财务，75% 通知部门负责人，90% 通知 CTO

3. **硬预算时邮件不告警**:
   - 可能是设计遗漏，也可能是有意让 Slack 承担这个责任
   - 实际使用中建议同时启用 Slack 或配置 Webhook

### 配置建议

1. **如果需要邮件在硬预算时告警**:
   - 同时启用 Slack，Slack 会发送 `budget_crossed`
   - 或配置 Webhook，Webhook 会收到所有事件

2. **如果需要邮件多阶段告警**:
   - 配置 `max_budget_alert_emails`:
   ```yaml
   max_budget_alert_emails:
     "50": ["finance@company.com"]
     "75": ["tech-lead@company.com", "finance@company.com"]
     "90": ["cto@company.com", "tech-lead@company.com"]
   ```

3. **注意 Slack 的 `threshold_crossed` 行为**:
   - 如果需要在 5% 时也收到告警，不要依赖 Slack 的 `threshold_crossed`
   - 用邮件的多阈值配置或自定义 Webhook
