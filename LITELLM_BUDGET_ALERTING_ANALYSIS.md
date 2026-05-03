# LiteLLM 预算告警机制分析报告

## 目录

1. [概述](#概述)
2. [预算检测触发点](#预算检测触发点)
3. [预算阈值配置](#预算阈值配置)
4. [Slack 告警机制](#slack-告警机制)
5. [邮件告警机制](#邮件告警机制)
6. [模块协作关系](#模块协作关系)
7. [完整告警流程图](#完整告警流程图)

---

## 概述

LiteLLM 提供了一套完整的预算管理和告警系统，用于监控和控制 LLM API 使用成本。该系统支持：

- **多层级预算管理**：Key、User、Team、Organization、Proxy 级别
- **多种告警阈值**：Soft Budget（软预算）、Max Budget（硬预算）、百分比阈值（5%、15%、80%等）
- **多渠道通知**：Slack Webhook、SMTP 邮件、SendGrid、Resend
- **防重复告警**：基于缓存的 TTL 机制，防止同一事件重复触发

---

## 预算检测触发点

预算检测主要在以下几个关键位置触发：

### 1. 认证检查阶段 (`auth_checks.py`)

这是预算检测的主要触发点，在 API 请求的认证阶段执行：

| 函数名 | 触发时机 | 检测内容 |
|--------|----------|----------|
| `_virtual_key_max_budget_check()` | Key 认证后 | Max Budget 硬预算检测 |
| `_virtual_key_soft_budget_check()` | Key 认证后 | Soft Budget 软预算检测 |
| `_virtual_key_max_budget_alert_check()` | Key 认证后 | 预算百分比阈值检测 |
| `_virtual_key_multi_budget_check()` | Key 认证后 | 多窗口预算检测 |

**核心代码位置**：`litellm/proxy/auth/auth_checks.py:3000-3250`

```python
# 示例：硬预算检测
if valid_token.max_budget is not None:
    spend = await get_current_spend(
        counter_key=f"spend:key:{valid_token.token}",
        fallback_spend=valid_token.spend or 0.0,
    )
    
    call_info = CallInfo(
        token=valid_token.token,
        spend=spend,
        max_budget=valid_token.max_budget,
        soft_budget=valid_token.soft_budget,
        ...
        event_group=Litellm_EntityType.KEY,
    )
    
    # 触发告警任务（非阻塞）
    asyncio.create_task(
        proxy_logging_obj.budget_alerts(
            type="token_budget",
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

### 2. 前置调用钩子 (`max_budget_limiter.py`)

用于用户级别的预算限制：

**文件位置**：`litellm/proxy/hooks/max_budget_limiter.py`

```python
class _PROXY_MaxBudgetLimiter(CustomLogger):
    async def async_pre_call_hook(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        cache: DualCache,
        data: dict,
        call_type: str,
    ):
        max_budget = user_api_key_dict.user_max_budget
        user_id = user_api_key_dict.user_id
        
        curr_spend = await get_current_spend(
            counter_key=f"spend:user:{user_id}",
            fallback_spend=user_api_key_dict.user_spend or 0.0,
        )
        
        if curr_spend >= max_budget:
            raise HTTPException(status_code=429, detail="Max budget limit reached.")
```

### 3. 模型级预算限制 (`model_max_budget_limiter.py`)

用于 Key + 模型组合的精细预算控制：

**文件位置**：`litellm/proxy/hooks/model_max_budget_limiter.py`

```python
class _PROXY_VirtualKeyModelMaxBudgetLimiter(RouterBudgetLimiting):
    async def is_key_within_model_budget(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        model: str,
    ) -> bool:
        # 检查特定模型的预算配置
        _model_max_budget = user_api_key_dict.model_max_budget
        _current_spend = await self._get_virtual_key_spend_for_model(
            user_api_key_hash=user_api_key_dict.token,
            model=model,
            key_budget_config=_current_model_budget_info,
        )
        
        if _current_spend > _current_model_budget_info.max_budget:
            raise litellm.BudgetExceededError(...)
```

---

## 预算阈值配置

### 1. 预算类型

| 预算类型 | 说明 | 触发行为 |
|----------|------|----------|
| **Soft Budget** | 软预算，告警阈值 | 仅触发告警，不阻止请求 |
| **Max Budget** | 硬预算，绝对上限 | 触发告警 + 阻止后续请求 |
| **Max Budget Alert** | 百分比阈值告警 | 达到预算的 X% 时告警 |

### 2. 内置百分比阈值

**Slack 告警阈值**（`litellm/types/integrations/slack_alerting.py`）：

```python
SLACK_ALERTING_THRESHOLD_5_PERCENT = 0.05   # 剩余 5% 时告警
SLACK_ALERTING_THRESHOLD_15_PERCENT = 0.15  # 剩余 15% 时告警
```

**邮件告警阈值**（常量定义）：

```python
EMAIL_BUDGET_ALERT_MAX_SPEND_ALERT_PERCENTAGE  # 默认 80%
```

### 3. 预算周期配置

预算支持周期性重置，通过 `budget_duration` 字段配置：

- `daily` - 每日重置
- `weekly` - 每周重置  
- `monthly` - 每月重置
- `yearly` - 每年重置

**预算重置作业**：`litellm/proxy/common_utils/reset_budget_job.py`

该作业定期检查到期的预算并重置 spend 计数器：

```python
class ResetBudgetJob:
    async def reset_budget(self):
        # 按顺序重置各层级预算
        await self.reset_budget_for_litellm_keys()      # Keys
        await self.reset_budget_for_litellm_users()     # Users
        await self.reset_budget_for_litellm_teams()     # Teams
        await self.reset_budget_for_litellm_budget_table()  # EndUsers
        await self.reset_budget_windows()               # Multi-window
```

---

## Slack 告警机制

### 1. 核心类

**主类**：`SlackAlerting`（`litellm/integrations/SlackAlerting/slack_alerting.py`）

继承自 `CustomBatchLogger`，提供批量发送和周期性刷新功能。

### 2. 预算告警方法

**核心方法**：`budget_alerts()`（`slack_alerting.py:547-637`）

```python
async def budget_alerts(
    self,
    type: Literal[
        "token_budget",
        "user_budget", 
        "soft_budget",
        "max_budget_alert",
        "team_budget",
        "organization_budget",
        "proxy_budget",
        "projected_limit_exceeded",
        "project_budget",
    ],
    user_info: CallInfo,
):
    # 1. 获取预算告警类型处理器
    budget_alert_class = get_budget_alert_type(type)
    _id = budget_alert_class.get_id(user_info)
    
    # 2. 检查预算状态
    event, event_message = self._get_event_and_event_message(
        event=event,
        user_info=user_info,
        event_message=event_message,
    )
    
    # 3. 防重复告警检查（缓存）
    _cache_key = "budget_alerts:{}:{}".format(event, _id)
    result = await _cache.async_get_cache(key=_cache_key)
    
    if result is None:  # 未发送过
        # 4. 发送告警
        await self.send_alert(
            message=event_message + "\n\n" + user_info_str,
            level="High",
            alert_type=AlertType.budget_alerts,
            user_info=webhook_event,
            alerting_metadata={},
        )
        # 5. 标记已发送（TTL 24小时）
        await _cache.async_set_cache(
            key=_cache_key,
            value="SENT",
            ttl=self.alerting_args.budget_alert_ttl,  # 24小时
        )
```

### 3. 预算告警类型工厂

**文件**：`litellm/integrations/SlackAlerting/budget_alert_types.py`

使用工厂模式根据告警类型返回对应的处理器：

```python
def get_budget_alert_type(type: Literal[...]) -> BaseBudgetAlertType:
    alert_types = {
        "proxy_budget": ProxyBudgetAlert(),
        "soft_budget": SoftBudgetAlert(),
        "user_budget": UserBudgetAlert(),
        "max_budget_alert": TokenBudgetAlert(),
        "team_budget": TeamBudgetAlert(),
        "organization_budget": OrganizationBudgetAlert(),
        "token_budget": TokenBudgetAlert(),
        "projected_limit_exceeded": ProjectedLimitExceededAlert(),
        "project_budget": ProjectBudgetAlert(),
    }
    return alert_types.get(type, ProxyBudgetAlert())
```

每种类型都实现了：
- `get_event_message()` - 返回告警消息前缀
- `get_id()` - 返回用于缓存的唯一 ID

### 4. 事件和消息生成

**方法**：`_get_event_and_event_message()`（`slack_alerting.py:639-699`）

检测具体的事件类型：

```python
def _get_event_and_event_message(self, user_info, event, event_message):
    percent_left = self._get_percent_of_max_budget_left(user_info)
    
    # Soft Budget 检查
    if user_info.soft_budget is not None:
        if user_info.spend >= user_info.soft_budget:
            event = "soft_budget_crossed"
            event_message += f"Total Soft Budget:`{user_info.soft_budget}`"
    
    # Max Budget 检查
    if user_info.max_budget is not None:
        if user_info.spend >= user_info.max_budget:
            event = "budget_crossed"
            event_message += f"Budget Crossed\n Total Budget:`{user_info.max_budget}`"
        elif percent_left <= SLACK_ALERTING_THRESHOLD_5_PERCENT:
            event = "threshold_crossed"
            event_message += "5% Threshold Crossed "
        elif percent_left <= SLACK_ALERTING_THRESHOLD_15_PERCENT:
            event = "threshold_crossed"
            event_message += "15% Threshold Crossed"
    
    return event, event_message
```

### 5. 消息发送

**方法**：`send_alert()`（`slack_alerting.py:1374-...`）

支持：
- 多 Webhook URL 配置（不同告警类型到不同频道）
- 批量发送（digest 模式）
- 周期性刷新

---

## 邮件告警机制

### 1. 核心类

**主类**：`BaseEmailLogger`（`enterprise/litellm_enterprise/enterprise_callbacks/send_emails/base_email.py`）

### 2. 支持的邮件发送方式

| 实现类 | 文件位置 | 说明 |
|--------|----------|------|
| `SMTPEmailLogger` | `smtp_email.py` | SMTP 协议发送 |
| `SendGridEmailLogger` | `sendgrid_email.py` | SendGrid API |
| `ResendEmailLogger` | `resend_email.py` | Resend API |

### 3. SMTP 发送实现

**基础发送函数**：`litellm/proxy/utils.py:4702-4773`

```python
async def send_email(
    receiver_email: Optional[str] = None,
    subject: Optional[str] = None,
    html: Optional[str] = None,
):
    # 从环境变量读取配置
    smtp_host = os.getenv("SMTP_HOST")
    smtp_port = int(os.getenv("SMTP_PORT", "587"))
    smtp_username = os.getenv("SMTP_USERNAME")
    smtp_password = os.getenv("SMTP_PASSWORD")
    sender_email = os.getenv("SMTP_SENDER_EMAIL")
    
    # 构建邮件
    email_message = MIMEMultipart()
    email_message["From"] = sender_email
    email_message["To"] = receiver_email
    email_message["Subject"] = subject
    email_message.attach(MIMEText(html, "html"))
    
    # 发送
    with smtplib.SMTP(host=smtp_host, port=smtp_port) as server:
        if os.getenv("SMTP_TLS", "True") != "False":
            server.starttls()
        if smtp_username and smtp_password:
            server.login(user=smtp_username, password=smtp_password)
        server.send_message(msg=email_message, ...)
```

### 4. 邮件预算告警

**方法**：`budget_alerts()`（`base_email.py:414-...`）

```python
async def budget_alerts(
    self,
    type: Literal[...],
    user_info: CallInfo,
):
    # 软预算告警
    if type == "soft_budget":
        if user_info.soft_budget and user_info.spend >= user_info.soft_budget:
            event = "soft_budget_crossed"
            # 检查缓存，防止重复发送
            _cache_key = f"email_budget_alerts:{event}:{_id}"
            result = await self.internal_usage_cache.async_get_cache(key=_cache_key)
            if result is None:
                # 发送邮件
                await self.send_soft_budget_alert_email(event=webhook_event)
                # 标记已发送
                await self.internal_usage_cache.async_set_cache(
                    key=_cache_key,
                    value="SENT",
                    ttl=EMAIL_BUDGET_ALERT_TTL,
                )
    
    # 最大预算告警（百分比阈值）
    elif type == "max_budget_alert":
        # 检查各百分比阈值：50%, 75%, 80%, 90%
        for threshold_pct, _ in alert_email_config.items():
            threshold = float(threshold_pct) / 100.0
            if user_info.spend >= (user_info.max_budget * threshold):
                event = f"max_budget_alert:{threshold_pct}"
                # 发送 + 缓存标记
                ...
```

### 5. 邮件模板

**位置**：`litellm/integrations/email_templates/`

| 模板 | 用途 |
|------|------|
| `SOFT_BUDGET_ALERT_EMAIL_TEMPLATE` | 软预算超限告警 |
| `MAX_BUDGET_ALERT_EMAIL_TEMPLATE` | 硬预算超限告警 |
| `TEAM_SOFT_BUDGET_ALERT_EMAIL_TEMPLATE` | 团队软预算告警 |
| `KEY_CREATED_EMAIL_TEMPLATE` | Key 创建通知 |
| `USER_INVITATION_EMAIL_TEMPLATE` | 用户邀请邮件 |

---

## 模块协作关系

### 1. 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           API 请求流程                                     │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    认证检查层 (auth_checks.py)                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────────┐  │
│  │ 预算检测函数  │  │ 预算检测函数  │  │     预算检测函数             │  │
│  │_max_budget_  │  │_soft_budget_ │  │_max_budget_alert_check()    │  │
│  │   check()    │  │   check()    │  │                              │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┬───────────────┘  │
└─────────┼──────────────────┼──────────────────────────┼──────────────────┘
          │                  │                          │
          └──────────────────┼──────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                 代理日志层 (proxy/utils.py: ProxyLogging)                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                    budget_alerts() 方法                            │   │
│  │  ┌─────────────────┐      ┌──────────────────────────────────┐   │   │
│  │  │ Slack 告警路径   │      │         邮件告警路径               │   │   │
│  │  │ ("slack" in     │      │ ("email" in alerting) OR         │   │   │
│  │  │  alerting)      │      │ (soft_budget with alert_emails)  │   │   │
│  │  └────────┬────────┘      └──────────────┬───────────────────┘   │   │
│  └───────────┼───────────────────────────────┼───────────────────────┘   │
└──────────────┼───────────────────────────────┼────────────────────────────┘
               │                               │
               ▼                               ▼
┌──────────────────────────────┐    ┌────────────────────────────────────┐
│   SlackAlerting 类           │    │      BaseEmailLogger 类            │
│  (slack_alerting.py)         │    │      (enterprise/.../base_email.py) │
│  ┌────────────────────────┐  │    │  ┌──────────────────────────────┐  │
│  │ - budget_alerts()      │  │    │  │ - budget_alerts()            │  │
│  │ - send_alert()         │  │    │  │ - send_soft_budget_alert()  │  │
│  │ - 缓存去重 (TTL 24h)   │  │    │  │ - send_max_budget_alert()   │  │
│  │ - 批量发送支持          │  │    │  │ - 缓存去重 (可配置 TTL)      │  │
│  └────────────────────────┘  │    │  └──────────────────────────────┘  │
└──────────────┬───────────────┘    └───────────────┬────────────────────┘
               │                                      │
               ▼                                      ▼
┌──────────────────────────────┐    ┌────────────────────────────────────┐
│   Slack Webhook API          │    │    SMTP / SendGrid / Resend       │
│   (外部服务)                  │    │         (邮件发送服务)              │
└──────────────────────────────┘    └────────────────────────────────────┘
```

### 2. 关键数据结构

#### CallInfo（预算告警信息）

**文件**：`litellm/proxy/_types.py`

```python
class CallInfo(BaseModel):
    token: Optional[str]           # API Key token
    spend: float                   # 当前花费
    max_budget: Optional[float]    # 硬预算上限
    soft_budget: Optional[float]   # 软预算阈值
    user_id: Optional[str]         # 用户 ID
    team_id: Optional[str]         # 团队 ID
    organization_id: Optional[str] # 组织 ID
    user_email: Optional[str]      # 用户邮箱（邮件告警用）
    key_alias: Optional[str]       # Key 别名
    team_alias: Optional[str]      # 团队别名
    event_group: Litellm_EntityType # 实体类型：KEY/USER/TEAM/ORGANIZATION
    alert_emails: Optional[List[str]] # 额外告警邮箱列表
```

#### WebhookEvent（Webhook 事件）

用于结构化的 Webhook 和邮件告警：

```python
class WebhookEvent(BaseModel):
    event: Literal[
        "budget_crossed",
        "threshold_crossed",
        "soft_budget_crossed",
        "projected_limit_exceeded",
        "spend_tracked",
        ...
    ]
    event_message: str
    spend: Optional[float]
    max_budget: Optional[float]
    soft_budget: Optional[float]
    token: Optional[str]
    user_id: Optional[str]
    team_id: Optional[str]
    user_email: Optional[str]
    key_alias: Optional[str]
    alert_emails: Optional[List[str]]
    ...
```

### 3. ProxyLogging 分发逻辑

**文件**：`litellm/proxy/utils.py:1596-1643`

```python
async def budget_alerts(self, type, user_info: CallInfo):
    # 特殊情况：soft_budget 且配置了 alert_emails，即使全局 alerting 未启用也发送
    is_soft_budget_with_alert_emails = (
        type == "soft_budget"
        and user_info.alert_emails is not None
        and len(user_info.alert_emails) > 0
    )
    
    if self.alerting is None and not is_soft_budget_with_alert_emails:
        return  # 告警未启用
    
    # Slack 告警路径
    if self.alerting is not None and "slack" in self.alerting:
        if self.slack_alerting_instance is not None:
            await self.slack_alerting_instance.budget_alerts(
                type=type, user_info=user_info
            )
    
    # 邮件告警路径
    should_send_email = (
        self.alerting is not None and "email" in self.alerting
    ) or is_soft_budget_with_alert_emails
    
    if should_send_email and self.email_logging_instance is not None:
        await self.email_logging_instance.budget_alerts(
            type=type, user_info=user_info
        )
```

---

## 完整告警流程图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                         预算告警完整流程图                                      │
└──────────────────────────────────────────────────────────────────────────────┘

1. 请求进入
     │
     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 1: 认证检查 (auth_checks.py)                                             │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  1.1 解析 API Key → valid_token (UserAPIKeyAuth)                         │ │
│  │                                                                           │ │
│  │  1.2 多层预算检测（并行触发告警）:                                          │ │
│  │      ┌─────────────────────────────────────────────────────────────────┐│ │
│  │      │ _virtual_key_max_budget_check()                                  ││ │
│  │      │ - 检查 max_budget                                                 ││ │
│  │      │ - 构建 CallInfo(spend, max_budget, soft_budget, ...)            ││ │
│  │      │ - asyncio.create_task( proxy_logging_obj.budget_alerts(         ││ │
│  │      │         type="token_budget", user_info=call_info ) )             ││ │
│  │      │ - 若 spend >= max_budget → 抛出 BudgetExceededError              ││ │
│  │      └─────────────────────────────────────────────────────────────────┘│ │
│  │                                                                           │ │
│  │      ┌─────────────────────────────────────────────────────────────────┐│ │
│  │      │ _virtual_key_soft_budget_check()                                 ││ │
│  │      │ - 检查 soft_budget                                                ││ │
│  │      │ - 若 spend >= soft_budget → 触发告警 type="soft_budget"         ││ │
│  │      │ - 仅告警，不阻止请求                                               ││ │
│  │      └─────────────────────────────────────────────────────────────────┘│ │
│  │                                                                           │ │
│  │      ┌─────────────────────────────────────────────────────────────────┐│ │
│  │      │ _virtual_key_max_budget_alert_check()                            ││ │
│  │      │ - 检查百分比阈值 (默认 80%, 可配置 50%/75%/90%)                  ││ │
│  │      │ - 支持全局配置 + 单 Key 配置合并                                   ││ │
│  │      │ - 触发告警 type="max_budget_alert"                                ││ │
│  │      └─────────────────────────────────────────────────────────────────┘│ │
│  │                                                                           │ │
│  │      ┌─────────────────────────────────────────────────────────────────┐│ │
│  │      │ _virtual_key_multi_budget_check()                                 ││ │
│  │      │ - 检查多窗口预算 (budget_limits)                                   ││ │
│  │      │ - 每个窗口独立计数器: spend:key:{token}:window:{duration}        ││ │
│  │      └─────────────────────────────────────────────────────────────────┘│ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     │ (asyncio.create_task 后台任务)
     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 2: 告警分发 (ProxyLogging.budget_alerts())                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  输入: type (预算类型), user_info (CallInfo)                              │ │
│  │                                                                           │ │
│  │  2.1 判断是否发送 Slack:                                                  │ │
│  │      if "slack" in self.alerting:                                        │ │
│  │          await self.slack_alerting_instance.budget_alerts(...)          │ │
│  │                                                                           │ │
│  │  2.2 判断是否发送邮件:                                                     │ │
│  │      if ("email" in self.alerting) OR                                    │ │
│  │         (type=="soft_budget" AND user_info.alert_emails 存在):          │ │
│  │          await self.email_logging_instance.budget_alerts(...)            │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     ├───────────────────────┐
     │                       │
     ▼                       ▼
┌──────────────────┐   ┌──────────────────────────────────────────────────────┐
│   Slack 路径      │   │                    邮件路径                          │
└──────────────────┘   └──────────────────────────────────────────────────────┘
     │                       │
     ▼                       ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 3: Slack 告警处理 (SlackAlerting.budget_alerts())                       │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  3.1 获取预算告警类型处理器:                                               │ │
│  │      budget_alert_class = get_budget_alert_type(type)                    │ │
│  │      # 如: TokenBudgetAlert, SoftBudgetAlert, TeamBudgetAlert 等        │ │
│  │                                                                           │ │
│  │  3.2 判断具体事件类型:                                                     │ │
│  │      event, event_message = _get_event_and_event_message(...)            │ │
│  │                                                                           │ │
│  │      可能的事件:                                                           │ │
│  │      ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │      │ "soft_budget_crossed"  → soft_budget 已超过                      │ │ │
│  │      │ "budget_crossed"       → max_budget 已超过                       │ │ │
│  │      │ "threshold_crossed"    → 剩余 15% 或 5%                          │ │ │
│  │      │ "projected_limit_exceeded" → 预测将超限                          │ │ │
│  │      └─────────────────────────────────────────────────────────────────┘ │ │
│  │                                                                           │ │
│  │  3.3 防重复告警检查 (缓存):                                                │ │
│  │      _cache_key = f"budget_alerts:{event}:{_id}"                         │ │
│  │      result = await cache.async_get_cache(_cache_key)                    │ │
│  │      if result is not None:  # 已发送过                                   │ │
│  │          return  # 直接跳过                                                │ │
│  │                                                                           │ │
│  │  3.4 构建 WebhookEvent:                                                    │ │
│  │      webhook_event = WebhookEvent(                                        │ │
│  │          event=event,                                                      │ │
│  │          event_message=event_message,                                      │ │
│  │          **user_info.model_dump()                                          │ │
│  │      )                                                                     │ │
│  │                                                                           │ │
│  │  3.5 发送告警:                                                             │ │
│  │      await self.send_alert(                                                │ │
│  │          message=event_message + "\n\n" + user_info_str,                 │ │
│  │          level="High",                                                     │ │
│  │          alert_type=AlertType.budget_alerts,                               │ │
│  │          user_info=webhook_event,                                          │ │
│  │          alerting_metadata={}                                              │ │
│  │      )                                                                     │ │
│  │                                                                           │ │
│  │  3.6 标记已发送 (设置缓存 TTL):                                             │ │
│  │      await cache.async_set_cache(                                          │ │
│  │          key=_cache_key,                                                   │ │
│  │          value="SENT",                                                     │ │
│  │          ttl=24 * 60 * 60  # 24 小时内不重复发送                         │ │
│  │      )                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 4: 邮件告警处理 (BaseEmailLogger.budget_alerts())                        │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  与 Slack 类似的流程，但有额外特性:                                         │ │
│  │                                                                           │ │
│  │  4.1 多阈值支持 (max_budget_alert):                                        │ │
│  │      支持 50%, 75%, 80%, 90% 等多个百分比阈值，每个阈值独立告警           │ │
│  │                                                                           │ │
│  │  4.2 团队告警邮箱支持:                                                     │ │
│  │      user_info.alert_emails 可包含多个收件人                              │ │
│  │      从 team_object.metadata.soft_budget_alerting_emails 读取            │ │
│  │                                                                           │ │
│  │  4.3 邮件模板:                                                             │ │
│  │      - SOFT_BUDGET_ALERT_EMAIL_TEMPLATE                                   │ │
│  │      - MAX_BUDGET_ALERT_EMAIL_TEMPLATE                                    │ │
│  │      - TEAM_SOFT_BUDGET_ALERT_EMAIL_TEMPLATE                              │ │
│  │                                                                           │ │
│  │  4.4 发送方式选择:                                                          │ │
│  │      ┌─────────────────────────────────────────────────────────────────┐ │ │
│  │      │ SMTPEmailLogger     → 使用 smtplib 发送 SMTP 邮件               │ │ │
│  │      │ SendGridEmailLogger  → 使用 SendGrid API                          │ │ │
│  │      │ ResendEmailLogger    → 使用 Resend API                            │ │ │
│  │      └─────────────────────────────────────────────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│  阶段 5: 外部通知投递                                                          │
│                                                                               │
│  Slack:  POST 请求到配置的 Webhook URL                                        │
│         支持: 单 URL / 多 URL / 按告警类型分频道                              │
│                                                                               │
│  邮件:  通过 SMTP 或邮件服务 API 发送到指定邮箱                               │
│         支持: logo 自定义、支持邮箱自定义 (企业版功能)                         │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 关键配置项

### 环境变量配置

| 变量名 | 说明 | 默认值 |
|--------|------|--------|
| `ALERTING` | 启用的告警渠道，如 `["slack", "email"]` | `None` |
| `SLACK_WEBHOOK_URL` | Slack Webhook URL | `None` |
| `WEBHOOK_URL` | 通用 Webhook URL | `None` |
| `SMTP_HOST` | SMTP 服务器地址 | `None` |
| `SMTP_PORT` | SMTP 端口 | `587` |
| `SMTP_USERNAME` | SMTP 用户名 | `None` |
| `SMTP_PASSWORD` | SMTP 密码 | `None` |
| `SMTP_SENDER_EMAIL` | 发件人邮箱 | `None` |
| `SMTP_TLS` | 是否启用 TLS | `"True"` |
| `EMAIL_LOGO_URL` | 邮件 Logo URL | LiteLLM 默认 |
| `EMAIL_SUPPORT_CONTACT` | 支持邮箱 | `support@berri.ai` |
| `SLACK_DAILY_REPORT_FREQUENCY` | 日报频率（秒） | `12*60*60` |
| `SLACK_BUDGET_ALERT_TTL` | 预算告警缓存 TTL | `24*60*60` |

### 配置示例

**proxy config.yaml 示例**:

```yaml
alerting: ["slack", "email"]

alert_types:
  - budget_alerts
  - llm_exceptions
  - daily_reports

slack_webhook_url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"

# 分频道配置
alert_to_webhook_url:
  budget_alerts: "https://hooks.slack.com/services/AAA/BBB/CCC"
  llm_exceptions: "https://hooks.slack.com/services/DDD/EEE/FFF"

# SMTP 配置
smtp_host: "smtp.gmail.com"
smtp_port: 587
smtp_username: "your-email@gmail.com"
smtp_password: "your-app-password"
smtp_sender_email: "notifications@yourcompany.com"
```

---

## 总结

LiteLLM 的预算告警系统是一个设计完善的多层级、多渠道告警体系：

### 核心特点

1. **分层预算管理**：支持 Key、User、Team、Organization、Proxy 多个层级
2. **多阈值告警**：软预算、硬预算、百分比阈值（5%/15%/80%等）
3. **防重复机制**：基于缓存的 TTL（默认 24 小时）防止告警风暴
4. **灵活配置**：支持全局配置和单实体（Key/Team）配置合并
5. **企业级特性**：多邮件收件人、自定义邮件模板、Logo 品牌化

### 模块职责

| 模块 | 主要职责 | 文件位置 |
|------|----------|----------|
| `auth_checks.py` | 预算检测、触发告警任务 | `litellm/proxy/auth/auth_checks.py` |
| `ProxyLogging` | 告警分发（Slack + 邮件） | `litellm/proxy/utils.py` |
| `SlackAlerting` | Slack 消息格式化、发送、去重 | `litellm/integrations/SlackAlerting/` |
| `BaseEmailLogger` | 邮件模板、发送、去重 | `enterprise/.../send_emails/` |
| `ResetBudgetJob` | 周期性预算重置 | `litellm/proxy/common_utils/reset_budget_job.py` |

### 告警触发顺序

当一次请求触发预算检查时，可能按以下顺序产生告警：

1. **Max Budget Alert**（如 80%）- 最先达到的百分比阈值
2. **15% Threshold** - 剩余 15% 预算
3. **5% Threshold** - 剩余 5% 预算
4. **Soft Budget Crossed** - 超过软预算（如配置）
5. **Budget Crossed** - 超过硬预算（同时阻止请求）

每个告警类型独立计算、独立缓存，确保不会因为一个告警触发而影响其他告警。
