# LiteLLM 路由器多策略架构深度分析报告

## 目录
1. [概述](#1-概述)
2. [策略基类与共享机制](#2-策略基类与共享机制)
3. [策略动态实例化机制](#3-策略动态实例化机制)
4. [自适应策略的信号与评分机制](#4-自适应策略的信号与评分机制)
5. [多模型组与后备链协作机制](#5-多模型组与后备链协作机制)
6. [架构总结与设计亮点](#6-架构总结与设计亮点)

---

## 1. 概述

LiteLLM 路由器是一个功能强大的多模型负载均衡与故障转移组件，支持多种智能分发策略。本文档深入分析其多策略架构的核心设计原理和实现机制。

### 1.1 核心能力矩阵

| 策略类型 | 策略名称 | 核心目标 | 适用场景 |
|---------|---------|---------|---------|
| 基础策略 | simple-shuffle | 随机轮询 | 简单负载均衡 |
| 负载策略 | least-busy | 最低连接数 | 流量均分 |
| 成本策略 | cost-based-routing | 最低成本 | 成本优化 |
| 延迟策略 | latency-based-routing | 最低延迟 | 性能优先 |
| 用量策略 | usage-based-routing | TPM/RPM 均衡 | 速率限制 |
| 用量策略 v2 | usage-based-routing-v2 | TPM/RPM 均衡（跨实例） | 多实例部署 |
| 复杂度策略 | complexity-router | 按请求复杂度路由 | 智能分层 |
| 自适应策略 | adaptive-router | 多臂赌博机学习 | 持续优化 |

### 1.2 架构层次

```
┌─────────────────────────────────────────────────────────────┐
│                      应用层 (Application)                     │
│              Router.acompletion / Router.completion           │
├─────────────────────────────────────────────────────────────┤
│                    策略决策层 (Strategy Decision)                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────┐    │
│  │ 预路由钩子   │ │ 策略选择器  │ │  后路由回调      │    │
│  │ Pre-Routing  │ │Strategy    │ │ Post-Callbacks   │    │
│  └─────────────┘ └─────────────┘ └─────────────────┘    │
├─────────────────────────────────────────────────────────────┤
│                   策略实现层 (Strategy Implementation)        │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │LowestCost│ │LowestLatency│ │LowestTPM/RPM│ │AdaptiveRouter│ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘ │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐ │
│  │Complexity│ │  LeastBusy  │ │SimpleShuffle│ │BudgetLimiter │ │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘ │
├─────────────────────────────────────────────────────────────┤
│                     基础设施层 (Infrastructure)                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────┐    │
│  │ BaseRouting  │ │ DualCache    │ │ CustomLogger     │    │
│  │ Strategy     │ │ (Memory+Redis) │ │ (Callback System)│    │
│  └──────────────┘ └──────────────┘ └──────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. 策略基类与共享机制

### 2.1 BaseRoutingStrategy 基类设计

所有需要跨实例同步的策略（如 `LowestTPMLoggingHandler_v2`）继承自 `BaseRoutingStrategy`，该基类提供了以下核心能力：

#### 2.1.1 类结构

```python
class BaseRoutingStrategy(ABC):
    def __init__(
        self,
        dual_cache: DualCache,
        should_batch_redis_writes: bool,
        default_sync_interval: Optional[Union[int, float]],
    ):
        self.dual_cache = dual_cache
        self.redis_increment_operation_queue: List[RedisPipelineIncrementOperation] = []
        self._sync_task: Optional[asyncio.Task[None]] = None
        self.in_memory_keys_to_update: set[str] = set()
```

**文件位置**: `litellm/router_strategy/base_routing_strategy.py:15-31`

#### 2.1.2 核心共享机制

| 机制 | 实现方式 | 设计目的 |
|-----|---------|---------|
| 双缓存架构 | `DualCache` = 内存缓存 + Redis 缓存 | 性能优化 + 跨实例同步 |
| 批量 Redis 写入 | `redis_increment_operation_queue` | 减少 Redis  round-trips |
| 定期同步任务 | `periodic_sync_in_memory_spend_with_redis` | 多实例间状态一致性 |
| 增量操作压缩 | 同键多操作合并为单次更新 | 进一步优化 Redis 写入 |

#### 2.1.3 同步流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ 内存缓存    │────▶│ 操作队列     │────▶│ Redis Pipeline│
│ In-Memory │     │  Queue       │     │   Batch    │
│   Cache    │     │            │     │            │
└──────────────┘     └──────────────┘     └──────────────┘
       ▲                                              │
       │                                              │
       └──────────────── 定期同步 ◀──────────────────┘
```

**关键代码流程**:
1. 请求到达时，先更新本地内存缓存（微秒级）
2. 将增量操作推入队列
3. 后台任务定期（默认 0.1 秒间隔）批量推送到 Redis
4. 同时从 Redis 拉取最新状态同步回内存

**文件位置**: `litellm/router_strategy/base_routing_strategy.py:95-261`

### 2.2 CustomLogger 回调机制

所有策略同时继承 `CustomLogger`，通过回调机制接收请求生命周期事件：

```python
class LowestCostLoggingHandler(CustomLogger):
    def log_success_event(self, kwargs, response_obj, start_time, end_time):
        # 更新成功指标
    
    def async_log_success_event(self, kwargs, response_obj, start_time, end_time):
        # 异步更新成功指标
    
    def async_get_available_deployments(self, model_group, healthy_deployments, ...):
        # 选择最优部署
```

**文件位置**: `litellm/router_strategy/lowest_cost.py:13-330`

### 2.3 策略接口统一

所有策略实现统一的接口模式：

| 接口方法 | 职责 | 调用时机 |
|---------|------|---------|
| `log_success_event` | 记录成功请求的指标更新 | 请求成功后 |
| `async_log_success_event` | 异步记录成功指标 | 异步请求成功后 |
| `log_failure_event` | 记录失败请求 | 请求失败后 |
| `get_available_deployments` | 同步选择最优部署 | 同步路由决策 |
| `async_get_available_deployments` | 异步选择最优部署 | 异步路由决策 |

---

## 3. 策略动态实例化机制

### 3.1 策略初始化入口

路由器通过 `routing_strategy_init` 方法根据配置动态实例化策略：

```python
def routing_strategy_init(
    self, 
    routing_strategy: Union[RoutingStrategy, str], 
    routing_strategy_args: dict
):
    # 1. 验证策略合法性
    valid_strategy_strings = ["simple-shuffle"] + [s.value for s in RoutingStrategy]
    
    # 2. 根据策略类型实例化
    if routing_strategy == RoutingStrategy.LEAST_BUSY.value:
        self.leastbusy_logger = LeastBusyLoggingHandler(router_cache=self.cache)
        litellm.logging_callback_manager.add_litellm_callback(self.leastbusy_logger)
    
    elif routing_strategy == RoutingStrategy.COST_BASED.value:
        self.lowestcost_logger = LowestCostLoggingHandler(
            router_cache=self.cache,
            routing_args={},
        )
        litellm.logging_callback_manager.add_litellm_callback(self.lowestcost_logger)
    
    # ... 其他策略类似
```

**文件位置**: `litellm/router.py:809-886`

### 3.2 支持的策略类型

`RoutingStrategy` 枚举定义了所有内置策略：

```python
class RoutingStrategy(enum.Enum):
    LEAST_BUSY = "least-busy"
    LATENCY_BASED = "latency-based-routing"
    COST_BASED = "cost-based-routing"
    USAGE_BASED_ROUTING_V2 = "usage-based-routing-v2"
    USAGE_BASED_ROUTING = "usage-based-routing"
    PROVIDER_BUDGET_LIMITING = "provider-budget-routing"
```

**文件位置**: `litellm/types/router.py:717-724`

### 3.3 策略实例化流程图

```
配置文件 / 构造参数
        │
        ▼
┌───────────────────┐
│ Router.__init__() │
└─────────┬─────────┘
          │
          ▼
┌───────────────────────────┐
│ routing_strategy_init()   │
│ 1. 验证策略名称合法性    │
│ 2. 实例化对应策略对象      │
│ 3. 注册为回调监听器        │
└───────────┬───────────────┘
            │
            ▼
    ┌───────────────┐
    │ 策略对象激活   │
    │ - 初始化缓存   │
    │ - 启动同步任务 │
    └───────────────┘
```

### 3.4 运行时策略切换

路由器支持运行时通过 `update_settings` 动态切换策略：

```python
# 在 update_settings 中检测策略变化
if var == "routing_strategy" and _existing_router_settings["routing_strategy"] != kwargs[var]:
    self.routing_strategy_init(
        routing_strategy=kwargs[var],
        routing_strategy_args=kwargs.get("routing_strategy_args", {})
    )
```

**文件位置**: `litellm/router.py:9037-9043`

### 3.5 高级路由钩子

除了传统策略，LiteLLM 还支持通过 **Pre-Routing Hook 模式：

```python
# 在路由决策前，高级策略可以修改目标模型：

class ComplexityRouter(CustomLogger):
    async def async_pre_routing_hook(
        self,
        model: str,
        request_kwargs: Dict,
        messages: Optional[List[Dict[str, Any]]] = None,
        ...
    ) -> Optional["PreRoutingHookResponse"]:
        # 1. 分析请求复杂度
        tier, score, signals = self.classify(user_message, system_prompt)
        
        # 2. 根据复杂度选择模型
        routed_model = self.get_model_for_tier(tier)
        
        # 3. 返回修改后的模型
        return PreRoutingHookResponse(model=routed_model, messages=messages)
```

**文件位置**: `litellm/router_strategy/complexity_router/complexity_router.py:414-477`

---

## 4. 自适应策略的信号与评分机制

### 4.1 AdaptiveRouter 核心设计

自适应路由器采用 **多臂赌博机 (Multi-Armed Bandit)** 算法，通过持续学习优化路由决策。

#### 4.1.1 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      AdaptiveRouter 实例                        │
│  (每个 router_name 一个实例)                                    │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐     │
│  │              _cells: 赌博机后验分布                        │     │
│  │  Dict[Tuple[RequestType, str], BanditCell]         │     │
│  │  - RequestType: code_generation, writing, etc.         │     │
│  │  - BanditCell: Beta(alpha, beta) 后验              │     │
│  └─────────────────────────────────────────────────────────┘     │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐ ┌──────────────────────────────────┐     │
│  │ _owner_cache     │ │ _session_states                 │     │
│  │ 会话所有权映射   │ │ 会话状态增量跟踪                │     │
│  │ 防止跨模型归因   │ │ 检测信号变化                    │     │
│  └──────────────────┘ └──────────────────────────────────┘     │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐     │
│  │         AdaptiveRouterUpdateQueue                      │     │
│  │         状态持久化队列 (刷新到 Postgres)              │     │
│  └─────────────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

**文件位置**: `litellm/router_strategy/adaptive_router/adaptive_router.py:72-99`

#### 4.1.2 输入信号来源

信号检测机制通过 `signals.py` 定义了完整的信号检测系统：

```python
@dataclass
class SignalDelta:
    """单次轮次触发的信号（0或1）"""
    misalignment: int = 0      # 用户重新表述问题
    stagnation: int = 0        # 助手重复回答
    disengagement: int = 0       # 用户表示放弃
    satisfaction: int = 0      # 用户表示满意
    failure: int = 0             # 工具调用失败
    loop: int = 0                # 工具调用循环
    exhaustion: int = 0         # 资源耗尽（超时、限流等）
```

**文件位置**: `litellm/router_strategy/adaptive_router/signals.py:31-41`

#### 4.1.3 信号检测规则

| 信号类型 | 检测条件 | 代码位置 |
|---------|---------|---------|
| **Misalignment** | 连续用户消息 Jaccard 相似度 >0 且 < 阈值 | `signals.py:136-143` |
| **Stagnation** | 连续助手消息 Jaccard 相似度 >= 阈值 | `signals.py:146-151` |
| **Disengagement** | 匹配 "forget it", "give up", "talk to human" 等 | `signals.py:118-124` |
| **Satisfaction** | 匹配 "thanks", "that worked", "perfect" 等 | `signals.py:126-133` |
| **Failure** | 工具结果 `is_error` 为 True | `signals.py:166-176` |
| **Loop** | 相同工具签名重复出现 >= 阈值 | `signals.py:190-200` |
| **Exhaustion** | HTTP 状态码 408/413/429/503/504 或关键字 | `signals.py:215-224` |

### 4.2 评分计算机制

#### 4.2.1 BanditCell 结构

```python
@dataclass(frozen=True)
class BanditCell:
    """单个 (router, request_type, model) 单元格的后验状态"""
    alpha: float  # 伪成功数
    beta: float   # 伪失败数
    
    @property
    def mean(self) -> float:
        total = self.alpha + self.beta
        return self.alpha / total if total > 0 else 0.5
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:28-42`

#### 4.2.2 信号到评分的映射

```python
@staticmethod
def _compute_bandit_delta(delta: SignalDelta) -> Tuple[float, float]:
    """
    将信号转换为赌博机更新
    
    映射规则:
    - satisfaction → +1 alpha (成功)
    - misalignment, stagnation, disengagement, failure → +1 beta 每个
    - loop → +0.5 beta (弱信号，可能是用户或模型问题)
    - exhaustion → 0 (服务端问题，单独跟踪)
    """
    d_alpha = float(delta.satisfaction)
    d_beta = (
        float(
            delta.misalignment
            + delta.stagnation
            + delta.disengagement
            + delta.failure
        )
        + 0.5 * delta.loop
    )
    return d_alpha, d_beta
```

**文件位置**: `litellm/router_strategy/adaptive_router/adaptive_router.py:432-454`

#### 4.2.3 多目标评分函数

```python
def score(
    quality_sample: float,
    model_cost: float,
    all_costs: List[float],
    quality_weight: float = DEFAULT_QUALITY_WEIGHT,  # 默认 0.7
    cost_weight: float = DEFAULT_COST_WEIGHT,    # 默认 0.3
) -> float:
    """
    多目标评分：质量 + 成本的加权线性组合
    
    score = quality_weight * quality_sample + cost_weight * normalized_cost
    
    其中 normalized_cost ∈ [0, 1], 0=最贵, 1=最便宜
    """
    cost_score = normalized_cost(model_cost, all_costs)
    return quality_weight * quality_sample + cost_weight * cost_score
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:102-114`

#### 4.2.4 Thompson Sampling 选择

```python
def pick_best(
    cells: Dict[str, BanditCell],
    model_costs: Dict[str, float],
    quality_weight: float = 0.7,
    cost_weight: float = 0.3,
) -> str:
    """
    每个模型采样一次，评分后返回最高分模型
    
    1. 对每个模型：
       - 从 Beta(alpha, beta) 采样质量估计
       - 计算多目标评分
    2. 选择最高分模型
    """
    all_costs = list(model_costs.values())
    best_model: Optional[str] = None
    best_score = float("-inf")
    
    for model, cell in cells.items():
        q = thompson_sample(cell)  # Beta 分布采样
        s = score(q, model_costs[model], all_costs, quality_weight, cost_weight)
        if s > best_score:
            best_score = s
            best_model = model
    
    return best_model
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:117-142`

### 4.3 冷启动与持续学习

#### 4.3.1 冷启动先验

```python
def initial_cell(
    prefs: AdaptiveRouterPreferences, 
    request_type: RequestType
) -> BanditCell:
    """
    冷启动先验初始化
    
    mean = base_tier_weight[tier] + (strength_bonus if匹配)
    total_mass = COLD_START_MASS (约10个伪样本)
    
    这样约10个真实样本才能显著移动后验
    """
    base = BASE_TIER_WEIGHT[prefs.quality_tier]
    bonus = STRENGTH_BONUS if request_type in prefs.strengths else 0.0
    mean = min(0.95, base + bonus)
    alpha = mean * COLD_START_MASS
    beta = (1.0 - mean) * COLD_START_MASS
    return BanditCell(alpha=alpha, beta=beta)
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:45-66`

#### 4.3.2 请求类型分类

```python
class RequestType(str, enum.Enum):
    """固定的 v0 分类体系"""
    CODE_GENERATION = "code_generation"
    CODE_UNDERSTANDING = "code_understanding"
    TECHNICAL_DESIGN = "technical_design"
    ANALYTICAL_REASONING = "analytical_reasoning"
    WRITING = "writing"
    FACTUAL_LOOKUP = "factual_lookup"
    GENERAL = "general"
```

**文件位置**: `litellm/types/router.py:805-815`

### 4.4 会话归因机制

为避免跨模型归因错误，自适应路由器使用 **会话所有权** 机制：

```python
def claim_or_check_owner(self, session_key: str, current_model: str) -> bool:
    """
    解决无状态路由下的归因问题
    
    - 第一次调用：claim 所有权，返回 True
    - 后续调用：检查是否匹配 owner_model
      - 匹配 → 可更新
      - 不匹配 → 跳过更新（避免错误归因）
    """
    now = time.time()
    existing = self._owner_cache.get(session_key)
    
    if existing is not None and existing[1] > now:
        owner_model, _ = existing
        if owner_model == current_model:
            return True  # 匹配，可以更新
        self._skipped_updates_total += 1
        return False  # 不匹配，跳过
    
    # 无所有者，claim 新所有权
    self._owner_cache[session_key] = (current_model, now + OWNER_CACHE_TTL_SECONDS)
    return True
```

**文件位置**: `litellm/router_strategy/adaptive_router/adaptive_router.py:217-247`

---

## 5. 多模型组与后备链协作机制

### 5.1 模型组概念

#### 5.1.1 模型组定义

模型组（Model Group）是路由器路由的基本单位：

```python
# 配置示例：
model_list:
  - model_name: "gpt-3.5-turbo"
    litellm_params:
      model: "azure/gpt-35-turbo"
      api_key: "os.environ/AZURE_API_KEY"
    tpm: 100000
    rpm: 1000
    
  - model_name: "gpt-3.5-turbo"
    litellm_params:
      model: "openai/gpt-3.5-turbo"
      api_key: "os.environ/OPENAI_API_KEY"
    tpm: 80000
    rpm: 600
```

**两个部署同属一个模型组 `"gpt-3.5-turbo"`，路由器会在组内进行负载均衡。

#### 5.1.2 模型组别名

```python
# 支持模型组别名配置
model_group_alias: {
    "my-gpt-3.5": [
        {"model": "gpt-3.5-turbo", "hidden": false},
        {"model": "claude-3-haiku", "hidden": true}
    ]
}
```

**文件位置**: `litellm/types/router.py:278-279`

### 5.2 后备链机制

#### 5.2.1 后备链配置格式

```python
# 标准格式：[{源模型组: [目标模型组列表]}
fallbacks: [
    {"gpt-3.5-turbo": ["claude-3-haiku", "gemini-pro"]},
    {"gpt-4": ["claude-3-opus"]},
    {"*": ["gpt-3.5-turbo"]}  # 通配符后备
]
```

#### 5.2.2 后备链查找逻辑

```python
def get_fallback_model_group(
    fallbacks: List[Any], 
    model_group: str
) -> Tuple[Optional[List[str]], Optional[int]]:
    """
    查找后备模型组的优先级：
    1. 精确匹配
    2. 去除 provider 前缀后的匹配 (如 "openai/gpt-3.5" → "gpt-3.5")
    3. 通配符 "*" 匹配
    """
    generic_fallback_idx: Optional[int] = None
    stripped_model_fallback: Optional[List[str]] = None
    fallback_model_group: Optional[List[str]] = None
    
    for idx, item in enumerate(fallbacks):
        if isinstance(item, dict):
            if list(item.keys())[0] == model_group:
                fallback_model_group = item[model_group]  # 精确匹配
                break
            elif _check_stripped_model_group(model_group, list(item.keys())[0]):
                stripped_model_fallback = item[list(item.keys())[0]]  # 去除前缀匹配
            elif list(item.keys())[0] == "*":
                generic_fallback_idx = idx  # 通配符
    
    # 优先级：精确 > 去除前缀 > 通配符
    if fallback_model_group is None:
        if stripped_model_fallback is not None:
            fallback_model_group = stripped_model_fallback
        elif generic_fallback_idx is not None:
            fallback_model_group = fallbacks[generic_fallback_idx]["*"]
    
    return fallback_model_group, generic_fallback_idx
```

**文件位置**: `litellm/router_utils/fallback_event_handlers.py:45-82`

#### 5.2.3 后备执行流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    请求执行流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 策略选择部署                                               │
│     ┌─────────┐                                                 │
│     │ Router│                                                 │
│     │.acomp │───▶ 策略.get_available_deployments()          │
│     │ letion│                                                 │
│     └────┬──┘                                                 │
│          │                                                       │
│          ▼                                                       │
│  2. 执行请求                                                   │
│     ┌──────────────────────────────────────────────────────────┐  │
│     │ litellm.acompletion(model=selected_deployment)       │  │
│     └──────────────────────┬───────────────────────────────┘  │
│                            │                                     │
│              ┌─────────────┴─────────────┐                    │
│              │                           │                    │
│              ▼                           ▼                    │
│         ┌────────┐              ┌──────────────┐          │
│         │ 成功    │              │ 失败         │          │
│         │ 返回响应 │              │ 触发后备链   │          │
│         └────────┘              └──────┬───────┘          │
│                                         │                       │
│                                         ▼                       │
│                              ┌──────────────────────────┐      │
│                              │ get_fallback_model_group() │      │
│                              │ 查找后备模型组          │      │
│                              └──────────┬───────────────┘      │
│                                         │                       │
│                                         ▼                       │
│                              ┌──────────────────────────┐      │
│                              │ run_async_fallback()  │      │
│                              │ 遍历后备模型组重试    │      │
│                              └──────────┬───────────────┘      │
│                                         │                       │
│                              ┌──────────┴───────────┐           │
│                              │                      │           │
│                              ▼                      ▼           │
│                        ┌────────┐           ┌──────────────┐    │
│                        │ 成功  │           │ 全部失败     │    │
│                        │ 返回  │           │ 抛出异常     │    │
│                        └────────┘           └──────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**文件位置**: `litellm/router_utils/fallback_event_handlers.py:85-161`

### 5.3 策略与后备链的协作

#### 5.3.1 协作架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                         一次请求的完整路径                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段 1: 预路由钩子 (Pre-Routing Hook)                            │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ ComplexityRouter / AdaptiveRouter.async_pre_routing_hook()   │ │
│  │ - 分析请求特征                                               │ │
│  │ - 选择目标模型组                                              │ │
│  │ - 返回 PreRoutingHookResponse(model=target_model)            │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  阶段 2: 策略选择 (Strategy Selection)                             │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ strategy.async_get_available_deployments(                       │ │
│  │     model_group=target_model,                                │ │
│  │     healthy_deployments=[...]                                 │ │
│  │ )                                                              │ │
│  │                                                                │ │
│  │ 不同策略的选择逻辑：                                           │ │
│  │ - LowestCost: 按成本排序                                     │ │
│  │ - LowestLatency: 按延迟排序 + 缓冲区选择                   │ │
│  │ - LowestTPM: 按当前用量选择最低的                             │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  阶段 3: 部署选择 (Deployment Selection)                          │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ selected_deployment = 策略返回的具体部署                       │ │
│  │                                                                │ │
│  │ 预调用检查 (Pre-call Check):                                  │ │
│  │ - TPM/RPM 限额检查                                        │ │
│  │ - 冷却状态检查                                                │ │
│  │ - 健康状态检查                                                │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  阶段 4: 执行与回调 (Execution & Callbacks)                      │
│  ┌───────────────────────────────────────────────────────────────┐ │
│  │ response = litellm.acompletion(**selected_deployment)        │ │
│  │                                                                │ │
│  │ 成功:                                                         │ │
│  │ - strategy.async_log_success_event()                        │ │
│  │   更新指标（成本、延迟、TPM、RPM）                            │ │
│  │                                                                │ │
│  │ 失败:                                                          │ │
│  │ - strategy.async_log_failure_event() (如有)                   │ │
│  │ - 触发后备链: get_fallback_model_group()                     │ │
│  │ - 重试: run_async_fallback()                                 │ │
│  └───────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 5.3.2 关键代码协作点

**路由器中的协作代码：

```python
# 在 async_function_with_fallbacks 中的协作
async def async_function_with_fallbacks(self, **kwargs):
    # 1. 获取模型组
    original_model_group = kwargs.get("model")
    
    # 2. 获取健康部署
    healthy_deployments = self._get_healthy_deployments(original_model_group)
    
    # 3. 策略选择部署
    if self.routing_strategy == "cost-based-routing":
        deployment = await self.lowestcost_logger.async_get_available_deployments(
            model_group=original_model_group,
            healthy_deployments=healthy_deployments,
            ...
        )
    elif self.routing_strategy == "latency-based-routing":
        deployment = await self.lowestlatency_logger.async_get_available_deployments(...)
    # ...
    
    # 4. 执行请求
    try:
        response = await litellm.acompletion(**deployment["litellm_params"])
        # 5. 成功回调
        # 策略通过 callback 机制自动记录成功事件
        return response
    except Exception as e:
        # 6. 失败触发后备
        fallback_model_group, _ = get_fallback_model_group(
            self.fallbacks, original_model_group
        )
        if fallback_model_group:
            return await run_async_fallback(
                litellm_router=self,
                fallback_model_group=fallback_model_group,
                original_model_group=original_model_group,
                original_exception=e,
                ...
            )
        raise
```

**文件位置**: `litellm/router.py:1713-1870`

### 5.4 特殊后备类型

| 后备类型 | 触发条件 | 配置字段 |
|---------|---------|---------|
| **普通后备 | 任何异常 | `fallbacks` |
| **上下文窗口后备 | 上下文长度超限 | `context_window_fallbacks` |
| **内容策略后备 | 内容违规 | `content_policy_fallbacks` |
| **通配符后备 | 任何模型组无特定后备 | `{"*": [...]}` |

---

## 6. 架构总结与设计亮点

### 6.1 架构设计原则

#### 6.1.1 分层架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    设计原则                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 接口统一                                                   │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ 所有策略实现统一接口：                               │   │
│     │ - log_success_event / async_log_success_event     │   │
│     │ - get_available_deployments / async_*            │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                 │
│  2. 回调驱动                                                   │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ 通过 CustomLogger 回调机制解耦：                     │   │
│     │ - 策略注册为回调，自动接收生命周期事件            │   │
│     │ - 路由器无需知道具体策略实现                       │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                 │
│  3. 双缓存同步                                                 │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ 内存 + Redis 双层架构：                              │   │
│     │ - 内存：微秒级读取，高性能                          │   │
│     │ - Redis：跨实例同步，一致性                          │   │
│     │ - 批量写入 + 定期同步，平衡性能与一致性              │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                 │
│  4. 无状态路由 + 有状态学习                                    │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ 自适应路由器的精妙设计：                               │   │
│     │ - 每轮 Thompson 采样：无状态，可水平扩展           │   │
│     │ - 会话所有权归因：防止跨模型错误归因              │   │
│     │ - 增量信号检测：O(1) 每轮，不重放历史              │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                 │
│  5. 故障透明后备                                                   │
│     ┌──────────────────────────────────────────────────────┐   │
│     │ 策略选择与后备链分离：                               │   │
│     │ - 策略：选择最优部署                                  │   │
│     │ - 后备：失败时的故障转移                             │   │
│     │ - 两者独立工作，互不干扰                          │   │
│     └──────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 策略对比总结

| 策略 | 继承关系 | 核心算法 | 信号来源 | 适用场景 |
|-----|-------|---------|---------|---------|
| **Simple Shuffle** | 独立实现 | 随机选择 | 无 | 简单负载均衡 |
| **Least Busy** | CustomLogger | 连接数计数 | 回调事件 | 连接数均衡 |
| **Lowest Cost** | CustomLogger | 成本排序 | 模型定价 + token 用量 | 成本优化 |
| **Lowest Latency** | CustomLogger | 延迟排序 + 缓冲区 | 响应时间历史 | 性能优先 |
| **Lowest TPM/RPM** | CustomLogger | 用量排序 | TPM/RPM 计数 | 速率限制 |
| **Lowest TPM/RPM v2** | BaseRoutingStrategy + CustomLogger | 用量排序 + Redis 同步 | TPM/RPM 计数 | 多实例部署 |
| **Complexity Router** | CustomLogger | 规则评分 + 关键词匹配 | 请求文本分析 | 智能分层 |
| **Adaptive Router** | 独立实现 (Pre-Routing Hook) | 多臂赌博机 + Thompson Sampling | 会话信号 + 用户反馈 | 持续优化 |

### 6.3 关键设计亮点

#### 6.3.1 批量 Redis 写入优化

```python
# BaseRoutingStrategy 中的批量写入机制
async def _push_in_memory_increments_to_redis(self):
    """
    压缩同键多次增量为单次操作：
    - 收集所有 increment 操作
    - 同键合并（加法交换律：a + b + c = (a + b) + c
    - 单次 pipeline 推送
    """
    compressed_ops: Dict[str, RedisPipelineIncrementOperation] = {}
    for op in self.redis_increment_operation_queue:
        if op["key"] in compressed_ops:
            compressed_ops[op["key"]]["increment_value"] += op["increment_value"]
        else:
            compressed_ops[op["key"]] = op
    
    # 单次 pipeline 执行
    await self.dual_cache.redis_cache.async_increment_pipeline(
        increment_list=list(compressed_ops.values())
    )
```

**文件位置**: `litellm/router_strategy/base_routing_strategy.py:116-173`

#### 6.3.2 自适应路由器的冷启动处理

```python
# 冷启动先验 + 持续学习
def initial_cell(prefs, request_type):
    """
    冷启动不盲目探索：
    - 使用 quality_tier → base_tier_weight
    使用 strengths → STRENGTH_BONUS
    total_mass = COLD_START_MASS (约10个伪样本)
    
    这样：
    1. 初始有意义的先验，不是均匀分布
    2. 约10个真实样本才能显著移动后验
    3. 避免冷启动时的随机探索浪费
    """
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:45-66`

#### 6.3.3 会话所有权防止归因错误

```python
# 无状态路由下的归因问题解决方案
def claim_or_check_owner(self, session_key, current_model):
    """
    问题：
    - 每轮独立 Thompson 采样
    - 同一 session 可能路由到不同模型
    - 如果模型 A 选了，模型 B 执行了，谁该得到 credit？
    
    解决方案：
    - 第一个模型 claim 所有权
    - 后续只有同一 session 只有匹配 owner 才更新
    - 不匹配则跳过，避免错误归因
    """
```

**文件位置**: `litellm/router_strategy/adaptive_router/adaptive_router.py:217-247`

### 6.4 扩展点

#### 6.4.1 自定义策略

```python
# 实现自定义策略的接口
class CustomRoutingStrategyBase:
    async def async_get_available_deployment(
        self,
        model: str,
        messages: Optional[List[Dict[str, str]]] = None,
        input: Optional[Union[str, List]]] = None,
        specific_deployment: Optional[bool] = False,
        request_kwargs: Optional[Dict] = None,
    ):
        """返回 model_list 中的一个元素"""
        pass
    
    def get_available_deployment(self, ...):
        """同步版本"""
        pass
```

**文件位置**: `litellm/types/router.py:616-663`

#### 6.4.2 预路由钩子

```python
# PreRoutingHookResponse 模式
class PreRoutingHookResponse(BaseModel):
    """
    预路由钩子可以修改：
    - model: 目标模型组
    - messages: 消息内容（可选）
    """
    model: str
    messages: Optional[List[Dict[str, Any]]]
```

**文件位置**: `litellm/types/router.py:792-803`

---

## 附录

### A. 关键文件位置

| 组件 | 文件路径 |
|-----|---------|
| 路由策略基类 | `litellm/router_strategy/base_routing_strategy.py` |
| 路由器核心 | `litellm/router.py` |
| 路由器类型定义 | `litellm/types/router.py` |
| 最低成本策略 | `litellm/router_strategy/lowest_cost.py` |
| 最低延迟策略 | `litellm/router_strategy/lowest_latency.py` |
| TPM/RPM 策略 v2 | `litellm/router_strategy/lowest_tpm_rpm_v2.py` |
| 复杂度路由器 | `litellm/router_strategy/complexity_router/` |
| 自适应路由器 | `litellm/router_strategy/adaptive_router/` |
| 后备链处理 | `litellm/router_utils/fallback_event_handlers.py` |

### B. 配置示例

```yaml
# router_settings:
#   routing_strategy: "cost-based-routing"
#   routing_strategy_args:
#     lowest_latency_buffer: 0.1  # 延迟策略：10%缓冲区
#
# model_list:
#   - model_name: "gpt-3.5-turbo"
#     litellm_params:
#       model: "openai/gpt-3.5-turbo"
#       input_cost_per_token: 0.0015
#       output_cost_per_token: 0.002
#     tpm: 100000
#     rpm: 1000
#
# fallbacks:
#   - {"gpt-3.5-turbo": ["claude-3-haiku"]}
#   - {"*": ["gpt-3.5-turbo"]}
```

---

## 7. 深入分析补充

### 7.1 后备链全部耗尽时的终止行为

#### 7.1.1 终止决策流程

当策略选出的模型及所有后备模型均失败后，路由器通过多层决策机制终止请求并上抛异常：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    后备链耗尽终止流程                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  请求入口: async_function_with_fallbacks()                          │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 1. 执行 async_function_with_retries()                         │ │
│  │    - 重试策略指定的次数                                         │ │
│  │    - 失败 → 跳转到后备处理                                     │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  async_function_with_fallbacks_common_utils()                       │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 2. 检查终止条件                                           │ │
│  │    - disable_fallbacks is True → 直接抛出 original_exception  │ │
│  │    - original_model_group is None → 直接抛出               │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 3. 查找后备模型组                                           │ │
│  │    - get_fallback_model_group(fallbacks, model_group)        │ │
│  │    - 无后备 → 在 original_exception.message 追加调试信息    │ │
│  │    - 无后备 → 抛出 original_exception                       │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  run_async_fallback()                                                │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 4. 遍历后备模型组                                           │ │
│  │    ┌────────────────────────────────────────────────────┐  │ │
│  │    │ for mg in fallback_model_group:                     │  │ │
│  │    │   try:                                               │  │ │
│  │    │     ① log_retry() - 记录重试日志                  │  │ │
│  │    │     ② 调用 async_function_with_fallbacks()         │  │ │
│  │    │        - 递归进入新模型组                              │  │ │
│  │    │        - 检查 fallback_depth >= max_fallbacks       │  │ │
│  │    │          → 抛出 original_exception (深度超限终止)     │  │ │
│  │    │     ③ 成功 → 返回响应 + log_success_fallback_event() │  │ │
│  │    │   except Exception as e:                             │  │ │
│  │    │     error_from_fallbacks = e                         │  │ │
│  │    │     log_failure_fallback_event() - 通知策略层     │  │ │
│  │    └────────────────────────────────────────────────────┘  │ │
│  │                                                              │ │
│  │ 5. 全部耗尽 → 抛出 error_from_fallbacks                     │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 7.1.2 关键终止代码

**1. 深度超限终止** (`run_async_fallback` 中的基例：

```python
async def run_async_fallback(
    ...,
    max_fallbacks: int,
    fallback_depth: int,
    **kwargs,
) -> Any:
    ### BASE CASE ### MAX FALLBACK DEPTH REACHED
    if fallback_depth >= max_fallbacks:
        raise original_exception  # 直接抛出原始异常，不包装
    
    error_from_fallbacks = original_exception
    
    for mg in fallback_model_group:
        if mg == original_model_group:
            continue
        try:
            # ... 重试计数 +1
            fallback_depth = fallback_depth + 1
            kwargs["fallback_depth"] = fallback_depth
            kwargs["max_fallbacks"] = max_fallbacks
            # 递归调用
            response = await litellm_router.async_function_with_fallbacks(
                *args, **kwargs
            )
            return response
        except Exception as e:
            error_from_fallbacks = e  # 保存最后一个异常
    
    raise error_from_fallbacks  # 全部耗尽，抛出最后一个
```

**文件位置**: `litellm/router_utils/fallback_event_handlers.py:116-161

**2. 无后备时的终止** (`async_function_with_fallbacks_common_utils`):

```python
async def async_function_with_fallbacks_common_utils(
    self,
    e: Exception,
    disable_fallbacks: Optional[bool],
    ...
):
    if disable_fallbacks is True or original_model_group is None:
        raise e  # 直接上抛原始异常
    
    # ... 查找后备模型组
    
    if fallback_model_group is None:
        verbose_router_logger.info(
            f"No fallback model group found for original model_group={model_group}"
        )
        # 在异常消息中追加调试信息
        if hasattr(original_exception, "message"):
            original_exception.message += (
                f"No fallback model group found for original model_group={model_group}. "
                f"Fallbacks={fallbacks}"
            )
        raise original_exception  # 上抛原始异常
```

**文件位置**: `litellm/router.py:5354-5549

#### 7.1.3 异常信息增强

当后备链耗尽时，路由器会在原始异常中追加丰富的调试信息：

```python
if hasattr(original_exception, "message"):
    # 追加已尝试的模型组信息
    original_exception.message += (
        ". Received Model Group={}\n"
        "Available Model Group Fallbacks={}".format(
            model_group,
            fallback_model_group,
        )
    )
    # 追加后备失败的具体错误
    if len(fallback_failure_exception_str) > 0:
        original_exception.message += (
            "\nError doing the fallback: {}".format(
                fallback_failure_exception_str
            )
        )

raise original_exception
```

**文件位置**: `litellm/router.py:5578-5591

#### 7.1.4 策略层与执行层的信号传递

| 信号类型 | 传递方式 | 调用时机 | 策略层响应 |
|---------|---------|---------|-----------|
| **重试开始 | `log_retry(kwargs, e)` | 每次后备尝试前 | 记录重试日志 |
| **后备成功** | `log_success_fallback_event()` | 后备成功后 | 策略可记录成功模型 |
| **后备失败** | `log_failure_fallback_event()` | 单个后备失败后 | 策略可记录失败模型 |
| **异常上抛** | `raise original_exception` | 全部耗尽后 | 调用方捕获处理 |

**信号传递代码**：

```python
# 重试前日志
kwargs = litellm_router.log_retry(kwargs=kwargs, e=original_exception)

# 后备成功回调
await log_success_fallback_event(
    original_model_group=original_model_group,
    kwargs=kwargs,
    original_exception=original_exception,
)

# 后备失败回调
await log_failure_fallback_event(
    original_model_group=original_model_group,
    kwargs=kwargs,
    original_exception=original_exception,
)
```

**文件位置**: `litellm/router_utils/fallback_event_handlers.py:127-160

---

### 7.2 自适应路由冷启动的先验初始化

#### 7.2.1 先验配置体系

自适应路由器的冷启动先验基于三层质量层级 + 能力标签系统：

```python
# 从 config.py 中定义的核心配置常量

# D4 — Cold-start prior 配置
BASE_TIER_WEIGHT: Dict[int, float] = {1: 0.3, 2: 0.5, 3: 0.7}
#   Tier 1: 最低质量预期成功率 30% (经济型模型)
#   Tier 2: 中等质量预期成功率 50% (标准型模型)
#   Tier 3: 最高质量预期成功率 70% (旗舰型模型)

STRENGTH_BONUS: float = 0.3
#   如果模型声明擅长某类任务类型，+0.3 预期成功率加成

COLD_START_MASS: float = 10.0
#   总伪样本数 = 10
#   意味着约 10 个真实样本才能显著移动后验
```

**文件位置**: `litellm/router_strategy/adaptive_router/config.py:16-20

#### 7.2.2 初始参数计算逻辑

`initial_cell` 函数根据模型的质量层级和能力标签计算 Beta 分布的初始 alpha/beta：

```python
def initial_cell(
    prefs: AdaptiveRouterPreferences, 
    request_type: RequestType
) -> BanditCell:
    """
    (model, request_type) 单元格的冷启动先验
    
    计算公式:
    1. base_mean = BASE_TIER_WEIGHT[quality_tier]
    2. bonus = STRENGTH_BONUS if request_type in strengths else 0.0
    3. mean = min(0.95, base_mean + bonus)  # 上限 0.95 防止过度自信
    4. alpha = mean * COLD_START_MASS
    5. beta = (1.0 - mean) * COLD_START_MASS
    
    示例:
    - Tier 3 模型, 无能力标签:
      mean = 0.7, alpha=7.0, beta=3.0
      
    - Tier 3 模型, 擅长 code_generation:
      mean = min(0.95, 0.7 + 0.3) = 0.95
      alpha=9.5, beta=0.5
    """
    # 验证质量层级
    if prefs.quality_tier not in BASE_TIER_WEIGHT:
        valid = sorted(BASE_TIER_WEIGHT)
        raise ValueError(
            f"quality_tier={prefs.quality_tier} is not supported; "
            f"valid tiers are {valid}"
        )
    
    base = BASE_TIER_WEIGHT[prefs.quality_tier]
    bonus = STRENGTH_BONUS if request_type in prefs.strengths else 0.0
    mean = min(0.95, base + bonus)
    
    alpha = mean * COLD_START_MASS
    beta = (1.0 - mean) * COLD_START_MASS
    
    return BanditCell(alpha=alpha, beta=beta)
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:45-66

#### 7.2.3 质量层级与能力标签配置

`AdaptiveRouterPreferences` 定义了模型的先验配置：

```python
# 从 types/router.py 推断的配置结构

class AdaptiveRouterPreferences(BaseModel):
    """单个模型的先验偏好配置"""
    quality_tier: int  # 1, 2, 3 - 基础质量层级
    strengths: List[RequestType]  # 模型擅长的请求类型列表
    model_cost: float  # $/1k tokens，用于多目标评分

# 请求类型枚举
class RequestType(str, enum.Enum):
    """固定的 v0 分类体系"""
    CODE_GENERATION = "code_generation"
    CODE_UNDERSTANDING = "code_understanding"
    TECHNICAL_DESIGN = "technical_design"
    ANALYTICAL_REASONING = "analytical_reasoning"
    WRITING = "writing"
    FACTUAL_LOOKUP = "factual_lookup"
    GENERAL = "general"
```

**文件位置**: `litellm/types/router.py:805-815

#### 7.2.4 先验对早期路由决策的影响机制

```
┌─────────────────────────────────────────────────────────────────────┐
│              冷启动先验 → 后验更新 → 决策影响              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段 1: 冷启动 (0 真实样本)                                     │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ BanditCell(alpha=7.0, beta=3.0)  (Tier 3, 无 strength) │ │
│  │                                                              │ │
│  │ mean = 7.0 / (7.0 + 3.0) = 0.7                            │ │
│  │ total_samples = 7.0 + 3.0 - 10.0 = 0 (真实样本数)        │ │
│  │                                                              │ │
│  │ Thompson 采样: Beta(7, 3)                                   │ │
│  │   - 95% 置信区间: ~(0.35, 0.93)                        │ │
│  │   - 高不确定性，但整体偏向高质量预期                             │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  阶段 2: 早期学习 (N < 10 真实样本)                             │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 假设收到 3 个成功, 1 个失败:                                │ │
│  │   new_alpha = 7.0 + 3.0 = 10.0                              │ │
│  │   new_beta  = 3.0 + 1.0 = 4.0                               │ │
│  │                                                              │ │
│  │ mean = 10.0 / 14.0 ≈ 0.71 (仅移动 0.01)                  │ │
│  │ total_samples = 14.0 - 10.0 = 4                           │ │
│  │                                                              │ │
│  │ 后验仍被先验主导，真实样本影响较小                           │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                              │                                      │
│                              ▼                                      │
│  阶段 3: 后期学习 (N >= 10 真实样本)                            │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 假设收到 15 个成功, 5 个失败:                             │ │
│  │   new_alpha = 7.0 + 15.0 = 22.0                         │ │
│  │   new_beta  = 3.0 + 5.0 = 8.0                           │ │
│  │                                                              │ │
│  │ mean = 22.0 / 30.0 ≈ 0.73                                │ │
│  │ total_samples = 30.0 - 10.0 = 20                          │ │
│  │                                                              │ │
│  │ 先验质量 10 个样本 vs 真实 20 个样本                     │ │
│  │ 后验开始由真实数据主导                                     │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 7.2.5 样本上限机制

```python
SAMPLE_CAP: int = 200  # 硬上限

def apply_delta(cell: BanditCell, delta_alpha: float, delta_beta: float) -> BanditCell:
    """
    应用学习更新，强制执行样本上限
    
    SAMPLE_CAP 是 (alpha + beta) 的硬上限。
    当超过上限时，直接丢弃更新。
    
    设计选择 (D5): 硬上限，不重新缩放 —— 保持 v0 简单
    """
    new_alpha = cell.alpha + delta_alpha
    new_beta = cell.beta + delta_beta
    
    if new_alpha + new_beta > SAMPLE_CAP:
        return cell  # 超过上限，不更新
    
    return BanditCell(alpha=new_alpha, beta=new_beta)
```

**文件位置**: `litellm/router_strategy/adaptive_router/bandit.py:69-80

#### 7.2.6 先验设计的权衡

| 设计决策 | 好处 | 代价 |
|---------|------|------|
| **COLD_START_MASS = 10** | 防止早期噪声样本过度影响；给真实样本需要积累再学习 | 新模型收敛较慢；需要更多真实样本才能显著调整 |
| **质量层级系统 | 利用领域知识编码；高置信度初始预期 | 层级映射可能不准确；需要手动配置 |
| **能力标签加成** | 利用模型已知优势；针对性优化路由 | 标签可能过时；需要持续验证 |
| **硬上限 SAMPLE_CAP=200** | 防止过拟合；模型退化时更容易切换 | 长期学习被截断；无法持续优化 |

---

### 7.3 多实例 Redis 同步的竞争处理

#### 7.3.1 双缓存架构与同步机制

`BaseRoutingStrategy` 实现了内存 + Redis 双层缓存同步机制：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    双缓存同步架构 (usage-based-routing-v2)                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  读路径 (低延迟优先)                                               │
│  ┌──────────────┐                                                 │
│  │  请求到达   │                                                 │
│  └──────┬───────┘                                                 │
│         │                                                          │
│         ▼                                                          │
│  ┌───────────────────────────────┐                                   │
│  │ 1. 本地内存缓存检查         │                                   │
│  │    async_get_cache(key, local_only=True)                       │
│  │    - 微秒级读取                                            │
│  │    - 如果本地已超限 → 直接抛 RateLimitError                  │
│  └───────────────┬───────────────┘                                   │
│                  │                                                  │
│                  ▼ 本地检查通过                                        │
│  ┌───────────────────────────────┐                                   │
│  │ 2. Redis 原子增量 + 检查    │                                   │
│  │    _increment_value_in_current_window()                       │
│  │    - Redis INCR 是原子操作                                    │
│  │    - 返回 Redis 超限 → 抛 RateLimitError                         │
│  └───────────────┬───────────────┘                                   │
│                  │                                                  │
│                  ▼ 检查通过                                            │
│  ┌───────────────────────────────┐                                   │
│  │ 3. 执行请求                  │                                   │
│  └───────────────┬───────────────┘                                   │
│                  │                                                  │
│                  ▼ 请求成功                                            │
│  ┌───────────────────────────────┐                                   │
│  │ 4. 更新本地缓存 + 入队操作    │                                   │
│  │    - router_cache.increment_cache()                               │
│  │    - redis_increment_operation_queue.append(op)                       │
│  └───────────────┬───────────────┘                                   │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  同步路径 (后台定期)                                                │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ periodic_sync_in_memory_spend_with_redis()                │ │
│  │ - 每 0.1 秒执行一次                                       │ │
│  │                                                              │ │
│  │ 步骤 1: _push_in_memory_increments_to_redis()            │ │
│  │   - 压缩同键多次增量为单次操作                            │ │
│  │   - Redis Pipeline 批量执行                                 │ │
│  │                                                              │ │
│  │ 步骤 2: 拉取 Redis 最新值                                │ │
│  │   - 合并 Redis 作为真值                                        │ │
│  │   - 合并公式: merged = redis_val + (本地增量)                 │ │
│  │   - 更新本地缓存                                                │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 7.3.2 增量操作压缩机制

```python
async def _push_in_memory_increments_to_redis(self):
    """
    同键增量压缩机制：
    
    场景:
    - 请求 A 增加 100 tokens
    - 请求 B 增加 50 tokens
    - 请求 C 增加 30 tokens
    
    队列: [{"key": "k1", "val": 100}, {"key": "k1", "val": 50}, {"key": "k1", "val": 30}]
    
    压缩后:
    - {"key": "k1", "val": 180} (单次 Redis 操作)
    """
    if len(self.redis_increment_operation_queue) > 0:
        # 压缩同键操作
        compressed_ops: Dict[str, RedisPipelineIncrementOperation] = {}
        
        for op in self.redis_increment_operation_queue:
            if op["key"] in compressed_ops:
                # 累加同键的值
                compressed_ops[op["key"]]["increment_value"] += op["increment_value"]
            else:
                compressed_ops[op["key"]] = op
        
        # 转换回列表
        compressed_queue = list(compressed_ops.values())
        
        # Redis Pipeline 批量执行
        increment_result = (
            await self.dual_cache.redis_cache.async_increment_pipeline(
                increment_list=compressed_queue,
            )
        )
```

**文件位置**: `litellm/router_strategy/base_routing_strategy.py:116-173

#### 7.3.3 预调用检查的双层保护

`LowestTPMLoggingHandler_v2` 实现了双层速率限制检查：

```python
async def async_pre_call_check(
    self, deployment: Dict, parent_otel_span: Optional[Span]
) -> Optional[Dict]:
    """
    预调用检查 + 更新 RPM 计数
    
    检查顺序:
    1. 本地内存缓存检查 (快速路径)
    2. Redis 原子增量 + 检查 (一致性路径)
    
    设计意图:
    - 本地检查防止绝大多数超限请求，避免不必要的 Redis round-trip
    - Redis 检查确保多实例间的一致性
    """
    dt = get_utc_datetime()
    current_minute = dt.strftime("%H-%M")
    model_id = deployment.get("model_info", {}).get("id")
    deployment_name = deployment.get("litellm_params", {}).get("model")
    
    rpm_key = f"{model_id}:{deployment_name}:rpm:{current_minute}"
    
    # ========== 第一层: 本地内存缓存检查 ==========
    local_result = await self.router_cache.async_get_cache(
        key=rpm_key, local_only=True
    )
    
    if local_result is not None and local_result >= deployment_rpm:
        # 本地已超限，快速失败
        raise litellm.RateLimitError(
            message="Deployment over defined rpm limit={}. current usage={}".format(
                deployment_rpm, local_result
            ),
            ...
        )
    
    # ========== 第二层: Redis 原子增量 + 检查 ==========
    else:
        # 本地检查通过，执行 Redis 原子增量
        result = await self._increment_value_in_current_window(
            key=rpm_key, value=1, ttl=self.routing_args.ttl
        )
        
        if result is not None and result > deployment_rpm:
            # Redis 返回值超限
            raise litellm.RateLimitError(
                message="Deployment over defined rpm limit={}. current usage={}".format(
                    deployment_rpm, result
                ),
                ...
            )
    
    return deployment
```

**文件位置**: `litellm/router_strategy/lowest_tpm_rpm_v2.py:141-225

#### 7.3.4 竞争处理与一致性保证

| 竞争场景 | 处理机制 | 一致性保证 |
|---------|---------|-----------|
| **同实例并发增量 | Redis INCR 原子操作 | 强一致 |
| **跨实例并发增量 | Redis INCR + 定期同步 | 最终一致 |
| **同键多增量** | 操作队列压缩 | 最终一致 |
| **Redis 不可用** | 降级到仅内存缓存 | 仅本实例一致 |

**关键一致性代码 (Redis 原子性依赖):

```python
# _sync_in_memory_spend_with_redis 中的合并逻辑

for key in cache_keys_list:
    redis_val = float(redis_values.get(key, 0) or 0)
    before = float(in_memory_before_dict.get(key, 0) or 0)
    after = float(
        await self.dual_cache.in_memory_cache.async_get_cache(key=key) or 0
    )
    
    delta = after - before  # 本实例自上次同步以来的增量
    
    if after <= redis_val:
        # 本实例本地值 <= Redis 值: 正常情况
        # 合并 = Redis 真值 + 本实例增量
        merged = redis_val + delta
        await self.dual_cache.in_memory_cache.async_set_cache(
            key=key, value=merged
        )
    else:
        # 本实例本地值 > Redis 值: 理论上不应该发生
        # 代码中注释掉的 debug 断言:
        # elif "rpm" in key:
        #     print(f"Redis_val={redis_val} is behind in-memory cache_val={after}")
        #     import os
        #     os._exit(1)
        #     raise Exception(...)
        continue  # 实际处理: 跳过，不更新本地
```

**文件位置**: `litellm/router_strategy/base_routing_strategy.py:236-256

#### 7.3.5 超速率限制的窗口期风险

```
┌─────────────────────────────────────────────────────────────────────┐
│                    窗口期风险分析                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  时间线 (假设同步间隔 0.1 秒 = 100ms)                      │
│                                                                     │
│  T=0ms:                                                         │
│    - 实例 A: 本地 RPM=50, Redis RPM=50                         │
│    - 实例 B: 本地 RPM=50, Redis RPM=50                         │
│    - 限制定额: RPM=100                                            │
│                                                                     │
│  T=10ms:                                                          │
│    - 实例 A: 收到 30 个请求 → 本地 RPM=80                      │
│    - 实例 B: 收到 30 个请求 → 本地 RPM=80                      │
│    - 两者本地检查均通过 (80 < 100)                                │
│    - 两者 Redis INCR 均执行                                        │
│                                                                     │
│  T=20ms:                                                          │
│    - Redis RPM = 50 + 30 + 30 = 110 (超限!)                  │
│    - 但两个实例的本地缓存仍为 80                                    │
│                                                                     │
│  T=50ms:                                                         │
│    - 实例 A: 再收到 15 个请求                                   │
│    - 本地检查: 80 + 15 = 95 < 100 → 通过                    │
│    - Redis INCR 返回 125 > 100 → 抛 RateLimitError           │
│    - 但已经执行了 Redis 检查，防止超限                         │
│                                                                     │
│  T=100ms:                                                         │
│    - 定期同步任务执行                                               │
│    - 实例 A 拉取 Redis=125                                      │
│    - 本地更新为 125                                                 │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  风险分析:                                                          │
│                                                                     │
│  风险点 1: 本地缓存落后 Redis (理论存在但实际有保护)                │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ 本地缓存可能在同步间隔内落后于 Redis 的实际值               │ │
│  │ 但 async_pre_call_check 有双重保护:                        │ │
│  │   1. 本地检查 (快速)                                       │ │
│  │   2. Redis INCR + 检查 (一致性)                            │ │
│  │                                                               │ │
│  │ 真正的风险场景:                                              │ │
│  │ - 如果只有本地检查，没有 Redis 检查                          │ │
│  │ - 但代码中两层都有                                          │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  风险点 2: 同步间隔内的超限                                     │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │ TPM 计数在请求成功后才更新                                    │ │
│  │ 存在窗口: 请求发出 → 成功返回 → 计数更新                   │ │
│  │                                                               │ │
│  │ 但 RPM 计数在 pre_call_check 时就通过 Redis INCR 原子更新   │ │
│  │ 所以 RPM 有强一致性保护                                      │ │
│  │ TPM 有最终一致性，但可能有短暂窗口                           │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

#### 7.3.6 风险缓解措施

| 风险类型 | 缓解措施 | 代码位置 |
|---------|---------|---------|
| **RPM 超限** | 双层检查 + Redis INCR 原子操作 | `lowest_tpm_rpm_v2.py:141-225` |
| **TPM 窗口** | 定期同步 + 增量合并 | `base_routing_strategy.py:195-261` |
| **Redis 不可用** | 降级到内存缓存，不失败请求 | `lowest_tpm_rpm_v2.py:222-225` |
| **高并发竞争** | Redis 单线程模型 + INCR 原子性 | Redis 原生保证 |

**降级处理代码**:

```python
async def async_pre_call_check(
    self, deployment: Dict, parent_otel_span: Optional[Span]
) -> Optional[Dict]:
    try:
        # ... 正常检查逻辑
    except Exception as e:
        if isinstance(e, litellm.RateLimitError):
            raise e
        # 非 RateLimitError 的其他异常（如 Redis 连接失败）
        return deployment  # 不失败请求，降级处理
```

**文件位置**: `litellm/router_strategy/lowest_tpm_rpm_v2.py:222-225

---

## 8. 附录 C：补充关键文件位置

| 补充分析 | 文件路径 |
|---------|---------|
| 后备链终止逻辑 | `litellm/router_utils/fallback_event_handlers.py:85-161` |
| 后备信号回调 | `litellm/router_utils/fallback_event_handlers.py:164-229` |
| 自适应冷启动配置 | `litellm/router_strategy/adaptive_router/config.py` |
| Beta 分布初始化 | `litellm/router_strategy/adaptive_router/bandit.py:45-80` |
| Redis 同步机制 | `litellm/router_strategy/base_routing_strategy.py:95-261` |
| 双层速率检查 | `litellm/router_strategy/lowest_tpm_rpm_v2.py:141-225` |

---

*报告生成时间: 2026-05-01*
*补充分析更新时间: 2026-05-01*
