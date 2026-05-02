# LiteLLM Proxy 请求处理链路深度分析报告

## 目录
1. [请求处理总览](#1-请求处理总览)
2. [阶段一：密钥鉴权与租户归属判定](#2-阶段一密钥鉴权与租户归属判定)
   - 2.1 [API Key 提取与预处理](#21-api-key-提取与预处理)
   - 2.2 [缓存优先的密钥查找](#22-缓存优先的密钥查找)
   - 2.3 [数据库回退查找](#23-数据库回退查找)
   - 2.4 [租户关联信息加载](#24-租户关联信息加载)
3. [阶段二：对象权限与路由限制检查](#3-阶段二对象权限与路由限制检查)
   - 3.1 [团队/组织状态检查](#31-团队组织状态检查)
   - 3.2 [模型访问权限检查](#32-模型访问权限检查)
   - 3.3 [路由权限检查](#33-路由权限检查)
   - 3.4 [对象权限检查](#34-对象权限检查)
4. [阶段三：预算与速率限制检查](#4-阶段三预算与速率限制检查)
   - 4.1 [common_checks 中的预算检查](#41-common_checks-中的预算检查)
   - 4.2 [预调用钩子中的速率限制](#42-预调用钩子中的速率限制)
   - 4.3 [预调用钩子中的预算限制](#43-预调用钩子中的预算限制)
5. [边界场景拦截机制](#5-边界场景拦截机制)
   - 5.1 [缓存未命中场景](#51-缓存未命中场景)
   - 5.2 [密钥过期场景](#52-密钥过期场景)
   - 5.3 [租户越权场景](#53-租户越权场景)
   - 5.4 [预算超限场景](#54-预算超限场景)
   - 5.5 [速率限制超限场景](#55-速率限制超限场景)
6. [完整执行顺序时序图](#6-完整执行顺序时序图)
7. [关键代码位置索引](#7-关键代码位置索引)

---

## 1. 请求处理总览

LiteLLM Proxy 的请求处理采用 **分层检查、逐级过滤** 的架构，整个请求生命周期可划分为四个主要阶段：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        请求处理总览                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  客户端请求                                                                   │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段一：密钥鉴权与租户归属判定 (user_api_key_auth.py)               │  │
│  │                                                                      │  │
│  │  1. API Key 提取 (Authorization Header)                             │  │
│  │  2. 快速路径：主密钥检查 (secrets.compare_digest)                   │  │
│  │  3. 缓存查找：get_key_object(check_cache_only=True)                 │  │
│  │  4. 数据库回退：get_key_object(check_cache_only=False)              │  │
│  │  5. 租户关联信息加载：Team、Organization、Project、End User         │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│      │                                                                      │
│      ▼ (返回 UserAPIKeyAuth 对象)                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段二：对象权限与路由限制检查 (common_checks in auth_checks.py)   │  │
│  │                                                                      │  │
│  │  1. 团队/项目状态检查 (blocked 字段)                                 │  │
│  │  2. 模型访问权限检查 (can_team_access_model, can_user_call_model)  │  │
│  │  3. 路由权限检查 (_is_api_route_allowed, RouteChecks)               │  │
│  │  4. 对象权限检查 (vector_store_access_check, check_tools_allowlist)│  │
│  │  5. 组织 RBAC 检查 (organization_role_based_access_check)           │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段三：预算与速率限制检查 (双轨机制)                                │  │
│  │                                                                      │  │
│  │  【轨道 A】common_checks 中的预算检查                                │  │
│  │    ├── 团队预算检查 (_team_max_budget_check)                        │  │
│  │    ├── 团队多窗口预算检查 (_team_multi_budget_check)                │  │
│  │    ├── 组织预算检查 (_organization_max_budget_check)                │  │
│  │    ├── 用户预算检查 (个人密钥场景)                                   │  │
│  │    ├── 团队成员预算检查 (_check_team_member_budget)                 │  │
│  │    └── 标签预算检查 (_tag_max_budget_check)                         │  │
│  │                                                                      │  │
│  │  【轨道 B】预调用钩子 (async_pre_call_hook)                         │  │
│  │    ├── DynamicRateLimitHandlerV3 (TPM/RPM 速率限制)               │  │
│  │    ├── MaxParallelRequestsHandlerV3 (并行请求限制)                  │  │
│  │    ├── MaxBudgetLimiter (用户级预算限制)                            │  │
│  │    ├── ModelMaxBudgetLimiter (模型级预算限制)                       │  │
│  │    └── 其他自定义回调 (CustomLogger)                                │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│      │                                                                      │
│      ▼                                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │ 阶段四：实际 LLM 调用与消费更新                                      │  │
│  │                                                                      │  │
│  │  1. route_request (模型路由与实际调用)                               │  │
│  │  2. 后调用钩子：消费更新 (proxy_track_cost_callback)                 │  │
│  │  3. 速率限制计数器更新 (dynamic_rate_limiter_v3)                    │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 阶段一：密钥鉴权与租户归属判定

阶段一的核心函数是 `user_api_key_auth()`，位于 `litellm/proxy/auth/user_api_key_auth.py`。这是每个请求的第一道防线。

### 2.1 API Key 提取与预处理

#### 执行流程

```
请求到达
    │
    ▼
从 Authorization Header 提取 API Key
    │
    ├── 格式：Bearer sk-...
    │
    ▼
快速路径检查
    │
    ├── 是否为 JWT Token?
    │   └── 是 → 走 JWT 认证分支 (check_jwt_token)
    │
    └── 是否为主密钥?
        └── 是 → secrets.compare_digest 时序安全比较
```

#### 关键代码

**API Key 提取** (`user_api_key_auth.py` 约第 600-700 行):
```python
# 从请求头提取
auth_header = request.headers.get("Authorization", "")
if auth_header.startswith("Bearer "):
    api_key = auth_header[len("Bearer "):]
```

**主密钥检查** (`user_api_key_auth.py:1103-1147`):
```python
try:
    is_master_key_valid = secrets.compare_digest(api_key, master_key)
except Exception:
    is_master_key_valid = False

if is_master_key_valid:
    _user_api_key_obj = await _return_user_api_key_auth_obj(
        user_obj=None,
        user_role=LitellmUserRoles.PROXY_ADMIN,  # 最高权限
        api_key=master_key,
        ...
    )
    # 缓存主密钥对象
    asyncio.create_task(
        _cache_key_object(
            hashed_token=hash_token(master_key),
            user_api_key_obj=_user_api_key_obj,
            ...
        )
    )
    return _user_api_key_obj
```

**安全设计要点**:
- 使用 `secrets.compare_digest` 而非普通字符串比较，防止时序攻击
- 主密钥具有 `PROXY_ADMIN` 角色，跳过大多数权限检查
- 主密钥同样会被缓存，避免每次请求都进行比较

### 2.2 缓存优先的密钥查找

#### 执行流程

```
非主密钥路径
    │
    ▼
检查密钥格式 (必须以 sk- 开头)
    │
    └── 否 → 抛出异常: "LiteLLM Virtual Key expected"
    │
    ▼
哈希 API Key
    │
    └── hash_token = hashlib.sha256(token.encode()).hexdigest()
    │
    ▼
缓存查找 (check_cache_only=True)
    │
    ├── 缓存命中
    │   └── 返回 cached UserAPIKeyAuth 对象
    │
    └── 缓存未命中
        └── 继续数据库查找
```

#### 关键代码

**密钥格式验证** (`user_api_key_auth.py:1168-1187`):
```python
if valid_token is None:
    if isinstance(api_key, str):
        _masked_key = (
            "{}****{}".format(api_key[:4], api_key[-4:])
            if len(api_key) > 8
            else "****"
        )
        assert api_key.startswith(
            "sk-"
        ), "LiteLLM Virtual Key expected. Received={}, expected to start with 'sk-'.".format(
            _masked_key
        )
```

**缓存查找** (`user_api_key_auth.py:1010-1024`):
```python
if valid_token is None:
    # Check CACHE
    try:
        with tracer.trace("litellm.proxy.auth.get_key_object_check_cache"):
            valid_token = await get_key_object(
                hashed_token=hash_token(api_key),
                prisma_client=prisma_client,
                user_api_key_cache=user_api_key_cache,
                parent_otel_span=parent_otel_span,
                proxy_logging_obj=proxy_logging_obj,
                check_cache_only=True,  # 只查缓存
            )
    except Exception:
        verbose_logger.debug("api key not found in cache.")
        valid_token = None
```

**`get_key_object` 缓存查找逻辑** (`auth_checks.py:2308-2324`):
```python
# 检查缓存
key = hashed_token

cached_key_obj: Optional[UserAPIKeyAuth] = await user_api_key_cache.async_get_cache(
    key=key
)

if cached_key_obj is not None:
    if isinstance(cached_key_obj, dict):
        return UserAPIKeyAuth(**cached_key_obj)
    elif isinstance(cached_key_obj, UserAPIKeyAuth):
        return cached_key_obj

if check_cache_only:
    raise Exception(
        f"Key doesn't exist in cache + check_cache_only=True. key={key}."
    )
```

### 2.3 数据库回退查找

#### 执行流程

```
缓存未命中
    │
    ▼
数据库查找
    │
    ├── _fetch_key_object_from_db_with_reconnect
    │   └── 查询 LiteLLM_VerificationToken 表
    │
    ├── 找到?
    │   ├── 是
    │   │   ├── 构造 UserAPIKeyAuth 对象
    │   │   ├── 加载对象权限 (object_permission)
    │   │   ├── 缓存回写 (_cache_key_object)
    │   │   └── 返回
    │   │
    │   └── 否
    │       └── 抛出 ProxyException: token_not_found_in_db
    │
    └── 数据库连接失败?
        └── 抛出 ProxyException: no_db_connection
```

#### 关键代码

**数据库查找入口** (`user_api_key_auth.py:1188-1210`):
```python
if valid_token is None:
    # ... 格式验证 ...
    
    if api_key.startswith("sk-"):
        api_key = hash_token(token=api_key)  # 哈希后再查数据库
    
    try:
        with tracer.trace("litellm.proxy.auth.get_key_object_from_db"):
            valid_token = await get_key_object(
                hashed_token=api_key,
                prisma_client=prisma_client,
                user_api_key_cache=user_api_key_cache,
                parent_otel_span=parent_otel_span,
                proxy_logging_obj=proxy_logging_obj,
            )
```

**`get_key_object` 完整逻辑** (`auth_checks.py:2290-2369`):
```python
async def get_key_object(
    hashed_token: str,
    prisma_client: Optional[PrismaClient],
    user_api_key_cache: DualCache,
    ...
    check_cache_only: Optional[bool] = None,
) -> UserAPIKeyAuth:
    
    if prisma_client is None:
        raise Exception("No DB Connected. See - https://docs.litellm.ai/docs/proxy/virtual_keys")
    
    # 阶段 1: 缓存检查
    key = hashed_token
    cached_key_obj: Optional[UserAPIKeyAuth] = await user_api_key_cache.async_get_cache(
        key=key
    )
    
    if cached_key_obj is not None:
        # 缓存命中，直接返回
        if isinstance(cached_key_obj, dict):
            return UserAPIKeyAuth(**cached_key_obj)
        elif isinstance(cached_key_obj, UserAPIKeyAuth):
            return cached_key_obj
    
    if check_cache_only:
        raise Exception(f"Key doesn't exist in cache + check_cache_only=True. key={key}.")
    
    # 阶段 2: 数据库查询
    _valid_token: Optional[BaseModel] = await _fetch_key_object_from_db_with_reconnect(
        hashed_token=hashed_token,
        prisma_client=prisma_client,
        parent_otel_span=parent_otel_span,
        proxy_logging_obj=proxy_logging_obj,
    )
    
    if _valid_token is None:
        raise ProxyException(
            message="Authentication Error, Invalid proxy server token passed. key={}, not found in db.".format(hashed_token),
            type=ProxyErrorTypes.token_not_found_in_db,
            param="key",
            code=status.HTTP_401_UNAUTHORIZED,
        )
    
    # 阶段 3: 构造响应对象
    _response = UserAPIKeyAuth(**_valid_token.model_dump(exclude_none=True))
    
    # 阶段 4: 加载对象权限 (如果有 object_permission_id)
    if _response.object_permission_id and not _response.object_permission:
        try:
            _response.object_permission = await get_object_permission(
                object_permission_id=_response.object_permission_id,
                prisma_client=prisma_client,
                user_api_key_cache=user_api_key_cache,
                ...
            )
        except Exception as e:
            verbose_proxy_logger.debug(f"Failed to load object_permission: {e}")
    
    # 阶段 5: 缓存回写
    await _cache_key_object(
        hashed_token=hashed_token,
        user_api_key_obj=_response,
        user_api_key_cache=user_api_key_cache,
        proxy_logging_obj=proxy_logging_obj,
    )
    
    return _response
```

**缓存回写机制** (`auth_checks.py` 约 2360-2367 行):
```python
await _cache_key_object(
    hashed_token=hashed_token,
    user_api_key_obj=_response,
    user_api_key_cache=user_api_key_cache,
    proxy_logging_obj=proxy_logging_obj,
)
```

缓存使用 `DualCache`，支持：
- **内存缓存**: 本地内存，最快
- **Redis 缓存**: 分布式缓存，支持多实例共享

### 2.4 租户关联信息加载

#### 执行流程

```
获得 UserAPIKeyAuth 对象后
    │
    ▼
关联信息加载 (在 user_api_key_auth 中并行执行)
    │
    ├── 如果有 team_id
    │   └── get_team_object() → 加载团队信息
    │       └── 更新 valid_token.team_* 字段
    │
    ├── 如果有 organization_id (或通过 team 关联)
    │   └── get_org_object() → 加载组织信息
    │
    ├── 如果有 project_id
    │   └── get_project_object() → 加载项目信息
    │
    └── 如果请求中有 end_user_id
        └── get_end_user_object() → 加载终端用户信息
```

#### 关键代码

**团队信息动态更新** (`user_api_key_auth.py:1071-1102`):
```python
if (
    valid_token is not None
    and isinstance(valid_token, UserAPIKeyAuth)
    and valid_token.team_id is not None
):
    # 基于缓存的团队对象更新密钥值
    # 这使得 /team/update 的值对已缓存的 token 也能生效
    try:
        team_obj: LiteLLM_TeamTableCachedObj = await get_team_object(
            team_id=valid_token.team_id,
            prisma_client=prisma_client,
            user_api_key_cache=user_api_key_cache,
            parent_otel_span=parent_otel_span,
            proxy_logging_obj=proxy_logging_obj,
            check_cache_only=True,  # 只查缓存
        )

        if (
            team_obj.last_refreshed_at is not None
            and valid_token.last_refreshed_at is not None
            and team_obj.last_refreshed_at > valid_token.last_refreshed_at
        ):
            # 团队对象比密钥对象新，需要更新密钥中的团队相关字段
            team_obj_dict = team_obj.__dict__
            
            for k, v in team_obj_dict.items():
                field_name = f"team_{k}"
                if field_name in valid_token.__fields__:
                    setattr(valid_token, field_name, v)
    except Exception as e:
        verbose_logger.debug(e)
```

**设计亮点**:
- 团队信息和密钥信息分别缓存
- 通过 `last_refreshed_at` 时间戳判断是否需要更新
- 管理员通过 `/team/update` 更新团队配置后，已缓存的密钥会自动感知

**终端用户信息加载** (`user_api_key_auth.py:951-1004`):
```python
# 检查 END-USER OBJECT
_end_user_object = None
end_user_params = {}

end_user_id = get_end_user_id_from_request_body(
    request_data, _safe_get_request_headers(request)
)
if end_user_id:
    try:
        end_user_params["end_user_id"] = end_user_id
        
        with tracer.trace("litellm.proxy.auth.get_end_user_object"):
            _end_user_object = await get_end_user_object(
                end_user_id=end_user_id,
                prisma_client=prisma_client,
                user_api_key_cache=user_api_key_cache,
                parent_otel_span=parent_otel_span,
                proxy_logging_obj=proxy_logging_obj,
                route=route,
            )
        if _end_user_object is not None:
            end_user_params["allowed_model_region"] = (
                _end_user_object.allowed_model_region
            )
            if _end_user_object.litellm_budget_table is not None:
                _apply_budget_limits_to_end_user_params(
                    end_user_params=end_user_params,
                    budget_info=_end_user_object.litellm_budget_table,
                    end_user_id=end_user_id,
                )
        # ... 默认预算处理 ...
    except Exception as e:
        if isinstance(e, litellm.BudgetExceededError):
            raise e  # 预算超限直接抛出
        verbose_proxy_logger.debug("Unable to find user in db. Error - {}".format(str(e)))
        pass
```

---

## 3. 阶段二：对象权限与路由限制检查

阶段二的核心是 `common_checks()` 函数，位于 `litellm/proxy/auth/auth_checks.py:451-701`。这是一个综合性的检查函数，涵盖状态、权限、路由等多个维度。

### 3.1 团队/组织状态检查

#### 检查顺序

```
common_checks 入口
    │
    ▼
检查 1: 团队是否被阻塞 (team.blocked)
    │
    └── 是 → 抛出 Exception: "Team={team_id} is blocked"
    │
    ▼
检查 1.1: 项目是否被阻塞 (project.blocked)
    │
    └── 是 → 抛出异常
```

#### 关键代码

**团队阻塞检查** (`auth_checks.py:491-495`):
```python
# 1. If team is blocked
if team_object is not None and team_object.blocked is True:
    raise Exception(
        f"Team={team_object.team_id} is blocked. Update via `/team/unblock` if you're an admin."
    )
```

**项目检查入口** (`auth_checks.py:560-569`):
```python
# 1.1 - 2.2 - 3.0.2 - 3.0.3: Project checks (blocked, model access, budget)
with tracer.trace("litellm.proxy.auth.common_checks.run_project_checks"):
    await _run_project_checks(
        project_object=project_object,
        _model=_model,
        llm_router=llm_router,
        skip_budget_checks=skip_budget_checks,
        valid_token=valid_token,
        proxy_logging_obj=proxy_logging_obj,
    )
```

### 3.2 模型访问权限检查

#### 检查顺序

```
状态检查通过后
    │
    ▼
检查 2: 团队模型访问权限 (can_team_access_model)
    │
    ├── 团队有 models 列表?
    │   ├── 是 → 检查请求模型是否在列表中
    │   │
    │   │   不在列表中?
    │   │   └── 尝试访问组 fallback (access_group_ids)
    │   │       └── 仍失败 → ProxyException: team_model_access_denied
    │   │
    │   └── 否 → 不限制
    │
    ▼
检查 2.2: 团队成员模型范围限制 (_check_team_member_model_access)
    │
    └── 成员有 per-member allowed_models?
        └── 是 → 进一步限制成员可访问的模型
    │
    ▼
检查 2.1: 用户模型访问权限 (个人密钥场景)
    │
    └── 用户有 models 列表?
        └── 是 → 检查请求模型
    │
    ▼
检查: 项目模型访问权限 (can_project_access_model)
    │
    └── 项目有 models 列表?
        └── 是 → 检查请求模型
```

#### 关键代码

**团队模型访问检查** (`auth_checks.py:497-513`):
```python
# 2. If team can call model
if _model and team_object:
    with tracer.trace("litellm.proxy.auth.common_checks.can_team_access_model"):
        if not await can_team_access_model(
            model=_model,
            team_object=team_object,
            llm_router=llm_router,
            team_model_aliases=(
                valid_token.team_model_aliases if valid_token else None
            ),
        ):
            raise ProxyException(
                message=f"Team not allowed to access model. Team={team_object.team_id}, Model={_model}. Allowed team models = {team_object.models}",
                type=ProxyErrorTypes.team_model_access_denied,
                param="model",
                code=status.HTTP_401_UNAUTHORIZED,
            )
```

**`can_team_access_model` 实现** (`auth_checks.py:2881-2920`):
```python
async def can_team_access_model(
    model: Union[str, List[str]],
    team_object: Optional[LiteLLM_TeamTable],
    llm_router: Optional[Router],
    team_model_aliases: Optional[Dict[str, str]] = None,
) -> Literal[True]:
    """
    Returns True if the team can access a specific model.
    
    1. First checks native team-level model permissions (current implementation)
    2. If not allowed natively, falls back to access_group_ids on the team
    """
    try:
        return _can_object_call_model(
            model=model,
            llm_router=llm_router,
            models=team_object.models if team_object else [],
            team_model_aliases=team_model_aliases,
            team_id=team_object.team_id if team_object else None,
            object_type="team",
        )
    except ProxyException:
        # Fallback: check team's access_group_ids
        team_access_group_ids = (
            (team_object.access_group_ids or []) if team_object else []
        )
        if team_access_group_ids:
            models_from_groups = await _get_models_from_access_groups(
                access_group_ids=team_access_group_ids,
            )
            if models_from_groups:
                return _can_object_call_model(
                    model=model,
                    llm_router=llm_router,
                    models=models_from_groups,
                    team_model_aliases=team_model_aliases,
                    team_id=team_object.team_id if team_object else None,
                    object_type="team",
                )
        raise  # 重新抛出原始异常
```

**团队成员模型限制** (`auth_checks.py:515-528`):
```python
# 2.2. If team member has per-member model scope, enforce it
if _model and team_object and valid_token and valid_token.user_id:
    with tracer.trace(
        "litellm.proxy.auth.common_checks.check_team_member_model_access"
    ):
        await _check_team_member_model_access(
            model=_model,
            team_object=team_object,
            valid_token=valid_token,
            llm_router=llm_router,
            prisma_client=prisma_client,
            user_api_key_cache=user_api_key_cache,
            proxy_logging_obj=proxy_logging_obj,
        )
```

**`_check_team_member_model_access` 实现** (`auth_checks.py:3332-3381`):
```python
async def _check_team_member_model_access(
    model: Union[str, List[str]],
    team_object: LiteLLM_TeamTable,
    valid_token: UserAPIKeyAuth,
    llm_router: Optional[Router],
    prisma_client: Optional["PrismaClient"],
    user_api_key_cache: DualCache,
    proxy_logging_obj: ProxyLogging,
) -> None:
    """
    Check if a team member's per-member model scope allows access to the requested model.
    
    Only enforced when the member's budget table has a non-empty allowed_models list.
    If allowed_models is empty or absent, the team-level models list applies (no extra restriction).
    """
    if valid_token.user_id is None or team_object.team_id is None:
        return
    
    # 获取团队成员关系
    team_membership = await get_team_membership(
        user_id=valid_token.user_id,
        team_id=team_object.team_id,
        prisma_client=prisma_client,
        user_api_key_cache=user_api_key_cache,
        proxy_logging_obj=proxy_logging_obj,
    )
    
    if (
        team_membership is None
        or team_membership.litellm_budget_table is None
        or not team_membership.litellm_budget_table.allowed_models
    ):
        return  # no per-member restriction — inherit team-level check
    
    member_allowed_models: List[str] = (
        team_membership.litellm_budget_table.allowed_models
    )
    try:
        _can_object_call_model(
            model=model,
            llm_router=llm_router,
            models=member_allowed_models,
            object_type="team",
        )
    except ProxyException:
        raise ProxyException(
            message=f"Team member not allowed to access model. User={valid_token.user_id}, Team={team_object.team_id}, Model={model}. Allowed member models = {member_allowed_models}",
            type=ProxyErrorTypes.team_model_access_denied,
            param="model",
            code=status.HTTP_401_UNAUTHORIZED,
        )
```

### 3.3 路由权限检查

#### 检查顺序

```
模型检查通过后
    │
    ▼
检查: 组织 RBAC 检查 (organization_role_based_access_check)
    │
    └── 检查用户在组织中的角色权限
    │
    ▼
检查: API 路由权限 (_is_api_route_allowed)
    │
    ├── 用户是 Proxy Admin?
    │   └── 是 → 跳过路由检查 (所有路由都允许)
    │
    └── 非 Admin 用户
        └── RouteChecks.non_proxy_admin_allowed_routes_check()
            │
            ├── 检查路由类型: openai_routes / management_routes
            ├── 检查 valid_token.allowed_routes
            └── 检查 user_role 对应的权限
```

#### 关键代码

**路由权限检查入口** (`auth_checks.py:676-682`):
```python
_is_route_allowed = _is_api_route_allowed(
    route=route,
    request=request,
    request_data=request_body,
    valid_token=valid_token,
    user_obj=user_object,
)
```

**`_is_api_route_allowed` 实现** (`auth_checks.py:721-745`):
```python
def _is_api_route_allowed(
    route: str,
    request: Request,
    request_data: dict,
    valid_token: Optional[UserAPIKeyAuth],
    user_obj: Optional[LiteLLM_UserTable] = None,
) -> bool:
    """
    - Route b/w api token check and normal token check
    """
    _user_role = _get_user_role(user_obj=user_obj)
    
    if valid_token is None:
        raise Exception("Invalid proxy server token passed. valid_token=None.")
    
    if not _is_user_proxy_admin(user_obj=user_obj):  # if non-admin
        RouteChecks.non_proxy_admin_allowed_routes_check(
            user_obj=user_obj,
            _user_role=_user_role,
            route=route,
            request=request,
            request_data=request_data,
            valid_token=valid_token,
        )
    return True
```

**`RouteChecks` 类** (`auth/route_checks.py`):
```python
class RouteChecks:
    """
    路由级别权限检查
    
    主要检查:
    1. 用户角色与路由的匹配
    2. 密钥 allowed_routes 配置
    3. 管理路由的特殊权限
    """
    
    @staticmethod
    def non_proxy_admin_allowed_routes_check(
        user_obj: Optional[LiteLLM_UserTable],
        _user_role: Optional[LitellmUserRoles],
        route: str,
        request: Request,
        request_data: dict,
        valid_token: UserAPIKeyAuth,
    ) -> None:
        # 检查路由是否在 LiteLLMRoutes.openai_routes 中
        if route in LiteLLMRoutes.openai_routes.value:
            # OpenAI 兼容路由 - 通常对所有有效密钥开放
            pass
        
        # 检查密钥的 allowed_routes 配置
        allowed_routes = valid_token.allowed_routes or []
        if allowed_routes:
            if route not in allowed_routes and "openai_routes" not in allowed_routes:
                raise ProxyException(
                    message=f"Route not allowed. route={route}, allowed_routes={allowed_routes}",
                    type=ProxyErrorTypes.invalid_route,
                    code=status.HTTP_403_FORBIDDEN,
                )
        
        # 管理路由检查
        if route in LiteLLMRoutes.management_routes.value:
            # 检查用户角色
            if _user_role == LitellmUserRoles.ORG_ADMIN:
                # 组织管理员 - 检查是否操作自己组织的资源
                pass
            elif _user_role == LitellmUserRoles.TEAM_ADMIN:
                # 团队管理员 - 检查是否操作自己团队的资源
                pass
            elif _user_role == LitellmUserRoles.INTERNAL_USER:
                # 普通用户 - 拒绝管理路由访问
                raise ProxyException(
                    message="Not authorized to access management routes",
                    type=ProxyErrorTypes.permission_denied,
                    code=status.HTTP_403_FORBIDDEN,
                )
```

### 3.4 对象权限检查

#### 检查顺序

```
路由检查通过后
    │
    ▼
检查 11: 向量存储访问权限 (vector_store_access_check)
    │
    └── 检查密钥/团队是否有权访问请求的 vector_store_id
    │
    ▼
检查 12: 工具白名单 (check_tools_allowlist)
    │
    └── 检查请求中的 tools 是否在 key/team 的 allowed_tools 中
    │
    └── 或者检查 blocked_tools 是否包含请求的工具
```

#### 关键代码

**向量存储检查** (`auth_checks.py:684-690`):
```python
# 11. [OPTIONAL] Vector store checks - is the object allowed to access the vector store
with tracer.trace("litellm.proxy.auth.common_checks.vector_store_access_check"):
    await vector_store_access_check(
        request_body=request_body,
        team_object=team_object,
        valid_token=valid_token,
    )
```

**工具白名单检查** (`auth_checks.py:692-699`):
```python
# 12. [OPTIONAL] Tool allowlist - key/team allowed_tools (no DB in hot path)
with tracer.trace("litellm.proxy.auth.common_checks.check_tools_allowlist"):
    await check_tools_allowlist(
        request_body=request_body,
        valid_token=valid_token,
        team_object=team_object,
        route=route,
    )
```

---

## 4. 阶段三：预算与速率限制检查

这是最复杂的阶段，采用 **双轨机制**：
1. **轨道 A**: `common_checks` 中的同步预算检查
2. **轨道 B**: `async_pre_call_hook` 中的异步速率和预算限制

### 4.1 common_checks 中的预算检查

#### 检查顺序

```
模型/路由/对象权限检查通过后
    │
    ▼
检查: 是否为免费模型? (_is_model_cost_zero)
    │
    └── 是 → 跳过所有预算检查 (skip_budget_checks=True)
    │
    ▼ (非免费模型)
检查 3: 团队预算检查 (_team_max_budget_check)
    │
    └── team.max_budget 已设置?
        └── 是 → 检查 spend > max_budget
            └── 是 → BudgetExceededError + 触发告警
    │
    ▼
检查 3.1: 团队多窗口预算检查 (_team_multi_budget_check)
    │
    └── team.budget_limits 有多个预算窗口?
        └── 是 → 逐个检查每个窗口
            例如: 每日预算 + 每月预算
    │
    ▼
检查 3.2: 密钥多窗口预算检查 (_virtual_key_multi_budget_check)
    │
    └── 类似团队多窗口检查
    │
    ▼
检查 3.0.5: 团队软预算检查 (_team_soft_budget_check)
    │
    └── 仅触发告警，不阻塞请求
    │
    ▼
检查 3.1: 组织预算检查 (_organization_max_budget_check)
    │
    └── 确定 org_id (token.org_id 或 team.organization_id)
        └── 检查 org 预算
    │
    ▼
检查: 标签预算检查 (_tag_max_budget_check)
    │
    └── 请求 metadata 中有 tags?
        └── 是 → 检查每个 tag 的预算
    │
    ▼
检查 4: 用户个人预算检查 (个人密钥场景)
    │
    └── 无 team_id 且 user.max_budget 已设置?
        └── 是 → 检查用户 spend
    │
    ▼
检查 4.2: 团队成员预算检查 (_check_team_member_budget)
    │
    └── 团队密钥场景，检查成员在团队内的个人预算
```

#### 关键代码

**免费模型判断** (`auth_checks.py:571-572`):
```python
# If this is a free model, skip all budget checks
if not skip_budget_checks:
    # 进入预算检查流程
```

**团队预算检查** (`auth_checks.py:573-598`):
```python
# 3. If team is in budget
with tracer.trace("litellm.proxy.auth.common_checks.team_max_budget_check"):
    await _team_max_budget_check(
        team_object=team_object,
        proxy_logging_obj=proxy_logging_obj,
        valid_token=valid_token,
    )

# 3.1. Multi-window budget check for team
with tracer.trace("litellm.proxy.auth.common_checks.team_multi_budget_check"):
    await _team_multi_budget_check(team_object=team_object)

# 3.2. Multi-window budget check for key
with tracer.trace(
    "litellm.proxy.auth.common_checks.virtual_key_multi_budget_check"
):
    if valid_token is not None:
        await _virtual_key_multi_budget_check(valid_token=valid_token)

# 3.0.5. If team is over soft budget (alert only, doesn't block)
with tracer.trace("litellm.proxy.auth.common_checks.team_soft_budget_check"):
    await _team_soft_budget_check(
        team_object=team_object,
        proxy_logging_obj=proxy_logging_obj,
        valid_token=valid_token,
    )
```

**`_team_max_budget_check` 实现** (`auth_checks.py:3384-3428`):
```python
async def _team_max_budget_check(
    team_object: Optional[LiteLLM_TeamTable],
    valid_token: Optional[UserAPIKeyAuth],
    proxy_logging_obj: ProxyLogging,
):
    """
    Check if the team is over it's max budget.
    
    Raises:
        BudgetExceededError if the team is over it's max budget.
        Triggers a budget alert if the team is over it's max budget.
    """
    if team_object is not None and team_object.max_budget is not None:
        from litellm.proxy.proxy_server import get_current_spend
        
        # Read spend from cross-pod counter (Redis-first) or cached object (fallback)
        spend = await get_current_spend(
            counter_key=f"spend:team:{team_object.team_id}",
            fallback_spend=team_object.spend or 0.0,
        )
        
        if spend > team_object.max_budget:
            if valid_token:
                # 构造告警信息
                call_info = CallInfo(
                    token=valid_token.token,
                    spend=spend,
                    max_budget=team_object.max_budget,
                    user_id=valid_token.user_id,
                    team_id=valid_token.team_id,
                    team_alias=valid_token.team_alias,
                    organization_id=valid_token.org_id,
                    event_group=Litellm_EntityType.TEAM,
                )
                # 异步触发告警 (不阻塞主流程)
                asyncio.create_task(
                    proxy_logging_obj.budget_alerts(
                        type="team_budget",
                        user_info=call_info,
                    )
                )
            
            # 抛出异常阻塞请求
            raise litellm.BudgetExceededError(
                current_cost=spend,
                max_budget=team_object.max_budget,
                message=f"Budget has been exceeded! Team={team_object.team_id} Current cost: {spend}, Max budget: {team_object.max_budget}",
            )
```

**`get_current_spend` 跨 Pod 计数器** (`proxy_server.py` 中):
```python
async def get_current_spend(
    counter_key: str,
    fallback_spend: float,
) -> float:
    """
    获取当前消费金额
    
    优先级:
    1. Redis 中的实时计数器 (跨 Pod 共享)
    2. 缓存/数据库中的 fallback 值
    
    Redis 计数器的键格式:
    - spend:team:{team_id}
    - spend:org:{org_id}
    - spend:user:{user_id}
    - spend:key:{token_hash}
    - spend:team_member:{user_id}:{team_id}
    """
    from litellm.proxy.proxy_server import user_api_key_cache
    
    try:
        # 尝试从 Redis 获取实时计数器
        redis_spend = await user_api_key_cache.async_get_cache(
            key=counter_key,
        )
        if redis_spend is not None:
            return float(redis_spend)
    except Exception:
        pass
    
    # Fallback 到数据库中的值
    return fallback_spend
```

**组织预算检查** (`auth_checks.py:3689-3781`):
```python
async def _organization_max_budget_check(
    valid_token: Optional[UserAPIKeyAuth],
    team_object: Optional[LiteLLM_TeamTable],
    prisma_client: Optional[PrismaClient],
    user_api_key_cache: DualCache,
    proxy_logging_obj: ProxyLogging,
):
    """
    Check if the organization is over its max budget.
    
    确定 org_id 的优先级:
    1. First, tries to use valid_token.org_id (if key has organization_id set)
    2. Falls back to team_object.organization_id (if key doesn't have org_id but team does)
    """
    if valid_token is None or prisma_client is None:
        return
    
    # Determine organization_id
    org_id: Optional[str] = None
    if valid_token.org_id is not None:
        org_id = valid_token.org_id
    elif team_object is not None and team_object.organization_id is not None:
        org_id = team_object.organization_id
    
    if org_id is None:
        return  # No organization, skip check
    
    # Get organization object with budget table
    try:
        org_table = await get_org_object(
            org_id=org_id,
            prisma_client=prisma_client,
            user_api_key_cache=user_api_key_cache,
            proxy_logging_obj=proxy_logging_obj,
            include_budget_table=True,
        )
    except Exception:
        return
    
    if org_table is None:
        return
    
    # Get max_budget from organization's budget table
    org_max_budget: Optional[float] = None
    if org_table.litellm_budget_table is not None:
        org_max_budget = org_table.litellm_budget_table.max_budget
    
    if org_max_budget is None or org_max_budget <= 0:
        return
    
    # Read spend from cross-pod counter
    org_spend = await get_current_spend(
        counter_key=f"spend:org:{org_id}",
        fallback_spend=org_table.spend or 0.0,
    )
    
    if org_spend >= org_max_budget:
        # Trigger budget alert
        call_info = CallInfo(
            token=valid_token.token,
            spend=org_spend,
            max_budget=org_max_budget,
            user_id=valid_token.user_id,
            team_id=valid_token.team_id,
            team_alias=valid_token.team_alias,
            organization_id=org_id,
            event_group=Litellm_EntityType.ORGANIZATION,
        )
        asyncio.create_task(
            proxy_logging_obj.budget_alerts(
                type="organization_budget",
                user_info=call_info,
            )
        )
        
        raise litellm.BudgetExceededError(
            current_cost=org_spend,
            max_budget=org_max_budget,
            message=f"Budget has been exceeded! Organization={org_id} Current cost: {org_spend}, Max budget: {org_max_budget}",
        )
```

**团队成员预算检查** (`auth_checks.py:3266-3329`):
```python
async def _check_team_member_budget(
    team_object: Optional[LiteLLM_TeamTable],
    user_object: Optional[LiteLLM_UserTable],
    valid_token: Optional[UserAPIKeyAuth],
    prisma_client: Optional[PrismaClient],
    user_api_key_cache: DualCache,
    proxy_logging_obj: ProxyLogging,
):
    """Check if team member is over their max budget within the team."""
    if (
        team_object is not None
        and team_object.team_id is not None
        and user_object is not None
        and valid_token is not None
        and valid_token.user_id is not None
    ):
        team_membership = await get_team_membership(
            user_id=valid_token.user_id,
            team_id=team_object.team_id,
            prisma_client=prisma_client,
            user_api_key_cache=user_api_key_cache,
            proxy_logging_obj=proxy_logging_obj,
        )
        
        # Per-member override wins; otherwise fall back to team-level default
        team_member_budget: Optional[float] = None
        if (
            team_membership is not None
            and team_membership.litellm_budget_table is not None
        ):
            team_member_budget = team_membership.litellm_budget_table.max_budget
        else:
            default_budget_id = (team_object.metadata or {}).get(
                "team_member_budget_id"
            )
            if isinstance(default_budget_id, str):
                default_budget = await get_team_member_default_budget(
                    budget_id=default_budget_id,
                    prisma_client=prisma_client,
                    user_api_key_cache=user_api_key_cache,
                )
                if default_budget is not None:
                    team_member_budget = default_budget.max_budget
        
        if team_member_budget is not None:
            team_member_spend = (
                team_membership.spend if team_membership is not None else 0.0
            ) or 0.0
            
            # Read from cross-pod counter (Redis-first) if available
            team_member_spend = await get_current_spend(
                counter_key=f"spend:team_member:{valid_token.user_id}:{team_object.team_id}",
                fallback_spend=team_member_spend,
            )
            
            if team_member_spend >= team_member_budget:
                raise litellm.BudgetExceededError(
                    current_cost=team_member_spend,
                    max_budget=team_member_budget,
                    message=f"Budget has been exceeded! User={valid_token.user_id} in Team={team_object.team_id} Current cost: {team_member_spend}, Max budget: {team_member_budget}",
                )
```

### 4.2 预调用钩子中的速率限制

#### 执行机制

预调用钩子通过 `litellm.callbacks` 列表注册，在 `proxy_logging_obj.pre_call_hook()` 中按顺序执行：

```
common_checks 通过后
    │
    ▼
proxy_logging_obj.pre_call_hook() 被调用
    │
    ├── 1. 执行 Guardrail Pipelines (如有)
    │
    └── 2. 遍历 litellm.callbacks 列表
            │
            ├── CustomGuardrail 类型 → 执行防护检查
            │
            └── CustomLogger 类型 (有 async_pre_call_hook)
                    │
                    ├── DynamicRateLimitHandlerV3
                    │   └── TPM/RPM 速率限制检查
                    │
                    ├── MaxParallelRequestsHandlerV3
                    │   └── 并行请求数限制检查
                    │
                    └── ... 其他自定义回调
```

#### 关键代码

**预调用钩子入口** (`utils.py:1394-1445`):
```python
for callback in litellm.callbacks:
    start_time = time.time()
    _callback = None
    if isinstance(callback, str):
        _callback = litellm.litellm_core_utils.litellm_logging.get_custom_logger_compatible_class(
            cast(_custom_logger_compatible_callbacks_literal, callback)
        )
    else:
        _callback = callback
    
    if (
        _callback is not None
        and isinstance(_callback, CustomGuardrail)
        and data is not None
    ):
        # Guardrail 处理
        result = await self._process_guardrail_callback(
            callback=_callback,
            data=data,
            user_api_key_dict=user_api_key_dict,
            call_type=call_type,
            event_type=GuardrailEventHooks.pre_call,
        )
        if result is None:
            continue
        data = result
    
    elif (
        _callback is not None
        and isinstance(_callback, CustomLogger)
        and "async_pre_call_hook" in vars(_callback.__class__)
        and _callback.__class__.async_pre_call_hook
        != CustomLogger.async_pre_call_hook  # 不是默认实现 (说明有自定义)
    ):
        # 调用自定义预调用钩子
        response = await _callback.async_pre_call_hook(
            user_api_key_dict=user_api_key_dict,
            cache=self.call_details["user_api_key_cache"],
            data=data,
            call_type=call_type,
        )
        if response is not None:
            data = await self.process_pre_call_hook_response(
                response=response, data=data, call_type=call_type
            )
```

**`DynamicRateLimitHandlerV3` 实现** (`hooks/dynamic_rate_limiter_v3.py`):
```python
class _PROXY_DynamicRateLimitHandlerV3(CustomLogger):
    """
    Saturation-aware priority-based rate limiter using v3 infrastructure.
    
    Key features:
    1. Model capacity ALWAYS enforced at 100% (prevents over-allocation)
    2. Priority usage tracked from first request (accurate accounting)
    3. Priority limits only enforced when saturated >= threshold
    4. Three-phase checking prevents partial counter increments
    5. Reuses v3 limiter's Redis-based tracking (multi-instance safe)
    """
    
    async def async_pre_call_hook(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        cache: DualCache,
        data: dict,
        call_type: str,
    ):
        """
        三阶段检查流程:
        
        Phase 1: Read-only check of ALL limits (no increments)
            ↓
        Phase 2: Decide enforcement based on saturation
            ↓ 允许
        Phase 3: Increment counters only if request allowed
        
        当未饱和时: 优先级可以借用未使用的容量 (宽松)
        当饱和时: 严格执行基于优先级的限制 (公平)
        """
        # 获取模型信息
        model = data.get("model")
        model_group_info = self._get_model_group_info(model)
        
        # 构建速率限制描述符
        descriptors = self._build_rate_limit_descriptors(
            user_api_key_dict=user_api_key_dict,
            model_group_info=model_group_info,
        )
        
        # Phase 1: 只读检查所有限制
        check_results = await self._check_all_limits(descriptors)
        
        # Phase 2: 根据饱和度决定执行策略
        saturation = self._calculate_saturation(check_results)
        enforcement_strategy = self._decide_enforcement_strategy(saturation)
        
        # Phase 3: 原子递增计数器 (如果允许)
        if enforcement_strategy == "allow":
            await self._increment_counters_atomically(
                descriptors=descriptors,
                tokens_estimate=self._estimate_tokens(data),
            )
        else:
            # 超限，抛出 429
            raise HTTPException(
                status_code=429,
                detail={
                    "error": "Rate limit exceeded",
                    "type": "rate_limit_error",
                    "retry_after": self._calculate_retry_after(check_results),
                }
            )
```

### 4.3 预调用钩子中的预算限制

#### `MaxBudgetLimiter` 实现

注意：这与 `common_checks` 中的预算检查是**双重检查**：

```python
# hooks/max_budget_limiter.py

class _PROXY_MaxBudgetLimiter(CustomLogger):
    # Class variables or attributes
    def __init__(self):
        pass
    
    async def async_pre_call_hook(
        self,
        user_api_key_dict: UserAPIKeyAuth,
        cache: DualCache,
        data: dict,
        call_type: str,
    ):
        try:
            verbose_proxy_logger.debug("Inside Max Budget Limiter Pre-Call Hook")
            max_budget = user_api_key_dict.user_max_budget
            user_id = user_api_key_dict.user_id
            
            if max_budget is None or user_id is None:
                return
            
            # 个人预算只适用于非团队请求
            # 这与 common_checks 中的明确团队密钥豁免保持一致
            if user_api_key_dict.team_id is not None:
                return
            
            from litellm.proxy.proxy_server import get_current_spend
            
            curr_spend = await get_current_spend(
                counter_key=f"spend:user:{user_id}",
                fallback_spend=user_api_key_dict.user_spend or 0.0,
            )
            
            verbose_proxy_logger.debug(
                "MaxBudgetLimiter: user_id=%s, spend=%.6f, max=%.6f",
                user_id,
                curr_spend,
                max_budget,
            )
            
            # CHECK IF REQUEST ALLOWED
            if curr_spend >= max_budget:
                raise HTTPException(status_code=429, detail="Max budget limit reached.")
        except HTTPException as e:
            raise e
        except Exception as e:
            verbose_logger.exception(
                "litellm.proxy.hooks.max_budget_limiter.py::async_pre_call_hook(): Exception occured - {}".format(
                    str(e)
                )
            )
```

**设计说明**:
- `MaxBudgetLimiter` 只检查**个人密钥**场景（`team_id is None`）
- `common_checks` 中的预算检查更全面（团队、组织、标签等）
- 这是**双重保险**机制

---

## 5. 边界场景拦截机制

### 5.1 缓存未命中场景

#### 场景描述

当密钥不在缓存中时的处理流程。

#### 拦截流程

```
请求携带 API Key
    │
    ▼
get_key_object(check_cache_only=True)
    │
    ├── 缓存命中
    │   └── 直接返回 UserAPIKeyAuth
    │
    └── 缓存未命中 → 抛出 Exception
            │
            ▼
捕获异常，valid_token = None
    │
    ▼
继续执行，check_cache_only=False (默认)
    │
    ▼
数据库查询 _fetch_key_object_from_db_with_reconnect
    │
    ├── 找到
    │   ├── 构造 UserAPIKeyAuth
    │   ├── 缓存回写
    │   └── 返回
    │
    └── 未找到
        └── 抛出 ProxyException (token_not_found_in_db)
            │
            └── 返回 401 Unauthorized
```

#### 关键代码

**缓存未命中处理** (`user_api_key_auth.py:1010-1024`):
```python
if valid_token is None:
    # Check CACHE
    try:
        with tracer.trace("litellm.proxy.auth.get_key_object_check_cache"):
            valid_token = await get_key_object(
                hashed_token=hash_token(api_key),
                prisma_client=prisma_client,
                user_api_key_cache=user_api_key_cache,
                parent_otel_span=parent_otel_span,
                proxy_logging_obj=proxy_logging_obj,
                check_cache_only=True,  # 只查缓存
            )
    except Exception:
        verbose_logger.debug("api key not found in cache.")
        valid_token = None  # 继续走数据库路径
```

**数据库未找到时的异常** (`auth_checks.py:2334-2342`):
```python
if _valid_token is None:
    raise ProxyException(
        message="Authentication Error, Invalid proxy server token passed. key={}, not found in db. Create key via `/key/generate` call.".format(
            hashed_token
        ),
        type=ProxyErrorTypes.token_not_found_in_db,
        param="key",
        code=status.HTTP_401_UNAUTHORIZED,
    )
```

**异常类型定义** (`_types.py`):
```python
class ProxyErrorTypes(str, Enum):
    token_not_found_in_db = "token_not_found_in_db"
    expired_key = "expired_key"
    no_db_connection = "no_db_connection"
    team_model_access_denied = "team_model_access_denied"
    key_model_access_denied = "key_model_access_denied"
    permission_denied = "permission_denied"
    invalid_route = "invalid_route"
    # ...
```

### 5.2 密钥过期场景

#### 场景描述

密钥的 `expires` 字段已过当前时间。

#### 拦截流程

```
获取 UserAPIKeyAuth 对象后
    │
    ▼
检查 expires 字段
    │
    ├── expires 为 None
    │   └── 永不过期 → 通过
    │
    └── expires 有值
            │
            ▼
时区处理 (确保 UTC 比较)
    │
    ▼
比较 expiry_time < current_time
    │
    ├── 否 → 未过期 → 通过
    │
    └── 是 → 已过期
            │
            ├── 1. 删除缓存中的密钥对象
            │   └── _delete_cache_key_object(hashed_token)
            │
            └── 2. 抛出 ProxyException (expired_key)
                    │
                    └── 返回 400 Bad Request
```

#### 关键代码

**过期检查** (`user_api_key_auth.py:1037-1059`):
```python
if valid_token.expires is not None:
    current_time = datetime.now(timezone.utc)
    
    # 处理 expires 字段的类型 (可能是 datetime 或 ISO 字符串)
    if isinstance(valid_token.expires, datetime):
        expiry_time = valid_token.expires
    else:
        expiry_time = datetime.fromisoformat(valid_token.expires)
    
    # 时区规范化
    if (
        expiry_time.tzinfo is None
        or expiry_time.tzinfo.utcoffset(expiry_time) is None
    ):
        expiry_time = expiry_time.replace(tzinfo=timezone.utc)
    
    # 过期判断
    if expiry_time < current_time:
        # 从缓存中删除过期密钥
        await _delete_cache_key_object(
            hashed_token=hash_token(api_key),
            user_api_key_cache=user_api_key_cache,
            proxy_logging_obj=proxy_logging_obj,
        )
        # 抛出异常
        raise ProxyException(
            message=f"Authentication Error - Expired Key. Key Expiry time {expiry_time} and current time {current_time}",
            type=ProxyErrorTypes.expired_key,
            code=400,
            param=abbreviate_api_key(api_key=api_key),
        )
```

**设计亮点**:
- 过期后立即从缓存删除，防止后续请求仍命中缓存
- 使用 `abbreviate_api_key` 脱敏日志中的密钥

### 5.3 租户越权场景

#### 场景类型

| 场景类型 | 触发条件 | 拦截位置 |
|---------|---------|---------|
| 模型越权 | 请求的模型不在允许列表中 | `can_team_access_model`, `can_user_call_model` |
| 路由越权 | 访问未授权的管理路由 | `RouteChecks.non_proxy_admin_allowed_routes_check` |
| 团队越权 | 访问其他团队的资源 | 组织 RBAC + 数据级检查 |
| 对象越权 | 访问无权限的向量存储/工具 | `vector_store_access_check`, `check_tools_allowlist` |

#### 模型越权拦截流程

```
请求模型: gpt-4
    │
    ▼
团队 models 列表: ["gpt-3.5-turbo", "claude-3-sonnet"]
    │
    ▼
gpt-4 不在列表中
    │
    ▼
尝试 access_group_ids fallback
    │
    └── 仍失败
            │
            ▼
抛出 ProxyException
    │
    └── type: team_model_access_denied
        code: 401 Unauthorized
```

#### 关键代码

**模型越权异常** (`auth_checks.py:508-513`):
```python
raise ProxyException(
    message=f"Team not allowed to access model. Team={team_object.team_id}, Model={_model}. Allowed team models = {team_object.models}",
    type=ProxyErrorTypes.team_model_access_denied,
    param="model",
    code=status.HTTP_401_UNAUTHORIZED,
)
```

**团队成员模型越权** (`auth_checks.py:3375-3381`):
```python
raise ProxyException(
    message=f"Team member not allowed to access model. User={valid_token.user_id}, Team={team_object.team_id}, Model={model}. Allowed member models = {member_allowed_models}",
    type=ProxyErrorTypes.team_model_access_denied,
    param="model",
    code=status.HTTP_401_UNAUTHORIZED,
)
```

**路由越权检查** (`route_checks.py` 中):
```python
# 检查密钥的 allowed_routes 配置
allowed_routes = valid_token.allowed_routes or []
if allowed_routes:
    if route not in allowed_routes and "openai_routes" not in allowed_routes:
        raise ProxyException(
            message=f"Route not allowed. route={route}, allowed_routes={allowed_routes}",
            type=ProxyErrorTypes.invalid_route,
            code=status.HTTP_403_FORBIDDEN,
        )
```

### 5.4 预算超限场景

#### 拦截层级

预算检查在多个层级进行，任一超限都会拦截请求：

```
请求
    │
    ▼
common_checks 预算检查
    │
    ├── 组织预算超限?
    │   └── BudgetExceededError
    │
    ├── 团队预算超限?
    │   └── BudgetExceededError
    │
    ├── 团队多窗口预算超限?
    │   └── BudgetExceededError
    │
    ├── 标签预算超限?
    │   └── BudgetExceededError
    │
    ├── 团队成员预算超限?
    │   └── BudgetExceededError
    │
    └── 用户个人预算超限?
        └── BudgetExceededError
    │
    ▼
预调用钩子预算检查
    │
    └── MaxBudgetLimiter (个人密钥场景)
            │
            └── 超限 → HTTP 429
```

#### 关键代码

**预算超限异常** (`auth_checks.py:3424-3428`):
```python
raise litellm.BudgetExceededError(
    current_cost=spend,
    max_budget=team_object.max_budget,
    message=f"Budget has been exceeded! Team={team_object.team_id} Current cost: {spend}, Max budget: {team_object.max_budget}",
)
```

**告警触发** (`auth_checks.py:3417-3422`):
```python
asyncio.create_task(
    proxy_logging_obj.budget_alerts(
        type="team_budget",
        user_info=call_info,
    )
)
```

**软预算（仅告警，不拦截）** (`auth_checks.py:3464-3535`):
```python
async def _team_soft_budget_check(
    team_object: Optional[LiteLLM_TeamTable],
    valid_token: Optional[UserAPIKeyAuth],
    proxy_logging_obj: ProxyLogging,
):
    """
    Triggers a budget alert if the team is over it's soft budget.
    """
    if (
        team_object is not None
        and team_object.soft_budget is not None
        and team_object.spend is not None
        and team_object.spend >= team_object.soft_budget
    ):
        # 仅记录日志和发送告警，不抛出异常
        verbose_proxy_logger.debug(
            "Crossed Soft Budget for team %s, spend %s, soft_budget %s",
            team_object.team_id,
            team_object.spend,
            team_object.soft_budget,
        )
        
        # 发送邮件告警 (如果配置了 alert_emails)
        if valid_token and alert_emails:
            call_info = CallInfo(...)
            asyncio.create_task(
                proxy_logging_obj.budget_alerts(
                    type="team_soft_budget",
                    user_info=call_info,
                )
            )
```

### 5.5 速率限制超限场景

#### 拦截流程

```
预调用钩子阶段
    │
    ▼
DynamicRateLimitHandlerV3.async_pre_call_hook()
    │
    ▼
三阶段检查
    │
    ├── Phase 1: 只读检查所有限制
    │   ├── 检查密钥 TPM/RPM 限制
    │   ├── 检查团队 TPM/RPM 限制
    │   ├── 检查模型容量限制
    │   └── 检查优先级预留
    │
    ├── Phase 2: 饱和度判断
    │   └── 决定是否强制执行优先级限制
    │
    └── Phase 3: 原子递增或拒绝
            │
            ├── 允许 → 递增所有计数器
            │
            └── 拒绝 → 抛出 HTTP 429
```

#### 关键代码

**速率限制超限响应** (`dynamic_rate_limiter_v3.py` 中):
```python
else:
    # 超限，抛出 429
    raise HTTPException(
        status_code=429,
        detail={
            "error": "Rate limit exceeded",
            "type": "rate_limit_error",
            "retry_after": self._calculate_retry_after(check_results),
        }
    )
```

**并行请求限制** (`parallel_request_limiter_v3.py` 中):
```python
# 当 max_parallel_requests 超限
raise HTTPException(
    status_code=429,
    detail={
        "error": "Max parallel requests exceeded",
        "type": "parallel_request_limit_error",
    }
)
```

---

## 6. 完整执行顺序时序图

### 6.1 整体时序图

```
┌──────────┐    ┌──────────────────┐    ┌──────────────────┐    ┌──────────────┐
│  客户端   │    │  proxy_server.py │    │ user_api_key_auth│    │ auth_checks  │
└─────┬────┘    └────────┬─────────┘    └────────┬─────────┘    └──────┬───────┘
      │                  │                       │                      │
      │  POST /v1/chat/completions              │                      │
      │─────────────────>│                       │                      │
      │                  │                       │                      │
      │                  │  Depends(user_api_key_auth)               │
      │                  │──────────────────────>│                      │
      │                  │                       │                      │
      │                  │                       │  1. 提取 API Key    │
      │                  │                       │  2. 主密钥检查      │
      │                  │                       │  3. 缓存查找        │
      │                  │                       │     get_key_object(check_cache_only=True)
      │                  │                       │──────────────────────>│ (缓存检查)
      │                  │                       │                      │
      │                  │                       │     缓存未命中       │
      │                  │                       │     get_key_object(check_cache_only=False)
      │                  │                       │──────────────────────>│ (数据库查询)
      │                  │                       │                      │
      │                  │                       │     缓存回写         │
      │                  │                       │<──────────────────────│
      │                  │                       │                      │
      │                  │                       │  4. 加载关联信息     │
      │                  │                       │     - 团队信息        │
      │                  │                       │     - 组织信息        │
      │                  │                       │     - 终端用户信息    │
      │                  │                       │                      │
      │                  │<──────────────────────│                      │
      │                  │   返回 UserAPIKeyAuth │                      │
      │                  │                       │                      │
      │                  │  common_checks()      │                      │
      │                  │──────────────────────>│                      │
      │                  │                       │                      │
      │                  │                       │  检查 1: 团队阻塞     │
      │                  │                       │  检查 2: 模型权限     │
      │                  │                       │  检查 3: 预算检查     │
      │                  │                       │     - 团队预算        │
      │                  │                       │     - 组织预算        │
      │                  │                       │     - 标签预算        │
      │                  │                       │  检查 4: 路由权限     │
      │                  │                       │  检查 5: 对象权限     │
      │                  │                       │                      │
      │                  │<──────────────────────│                      │
      │                  │   返回 True           │                      │
      │                  │                       │                      │
      │                  │  pre_call_hook()      │                      │
      │                  │──────────────────────>│                      │
      │                  │                       │                      │
      │                  │                       │  遍历 litellm.callbacks
      │                  │                       │  - Guardrails         │
      │                  │                       │  - DynamicRateLimitHandlerV3
      │                  │                       │  - MaxBudgetLimiter  │
      │                  │                       │                      │
      │                  │<──────────────────────│                      │
      │                  │   返回处理后的 data   │                      │
      │                  │                       │                      │
      │                  │  route_request()      │                      │
      │                  │──────────────────────>│                      │
      │                  │                       │                      │
      │                  │                       │  实际 LLM API 调用    │
      │                  │                       │                      │
      │                  │<──────────────────────│                      │
      │                  │                       │                      │
      │                  │  post_call_hook()     │                      │
      │                  │──────────────────────>│                      │
      │                  │                       │                      │
      │                  │                       │  - 消费更新           │
      │                  │                       │  - 速率计数器更新     │
      │                  │                       │                      │
      │<─────────────────│                       │                      │
      │   HTTP 响应       │                       │                      │
      │                  │                       │                      │
```

### 6.2 关键决策点和异常拦截点

| 阶段 | 检查点 | 拦截条件 | 异常类型 | HTTP 状态码 |
|------|--------|----------|----------|-------------|
| **阶段一** | 主密钥检查 | 密钥不匹配 | - | 继续数据库查找 |
| **阶段一** | 缓存查找 | 缓存未命中 | Exception | 继续数据库查找 |
| **阶段一** | 数据库查找 | 密钥不存在 | `ProxyException` (token_not_found_in_db) | 401 Unauthorized |
| **阶段一** | 密钥过期 | `expires < now` | `ProxyException` (expired_key) | 400 Bad Request |
| **阶段一** | 密钥格式 | 不以 `sk-` 开头 | `AssertionError` | 400 Bad Request |
| **阶段二** | 团队阻塞 | `team.blocked = True` | `Exception` | 403 Forbidden |
| **阶段二** | 模型越权 | 模型不在允许列表 | `ProxyException` (team_model_access_denied) | 401 Unauthorized |
| **阶段二** | 路由越权 | 路由不在 `allowed_routes` | `ProxyException` (invalid_route) | 403 Forbidden |
| **阶段二** | 管理路由越权 | 非 Admin 访问管理路由 | `ProxyException` (permission_denied) | 403 Forbidden |
| **阶段二** | 组织预算 | `org.spend >= org.max_budget` | `BudgetExceededError` | 402 Payment Required |
| **阶段二** | 团队预算 | `team.spend >= team.max_budget` | `BudgetExceededError` | 402 Payment Required |
| **阶段二** | 团队成员预算 | `member.spend >= member.budget` | `BudgetExceededError` | 402 Payment Required |
| **阶段二** | 标签预算 | `tag.spend >= tag.max_budget` | `BudgetExceededError` | 402 Payment Required |
| **阶段三** | TPM 限制 | Token 数超限 | `HTTPException` | 429 Too Many Requests |
| **阶段三** | RPM 限制 | 请求数超限 | `HTTPException` | 429 Too Many Requests |
| **阶段三** | 并行请求限制 | 并行数超限 | `HTTPException` | 429 Too Many Requests |
| **阶段三** | 用户预算 (钩子) | `user.spend >= max_budget` | `HTTPException` | 429 Too Many Requests |

---

## 7. 关键代码位置索引

### 7.1 鉴权与密钥查找

| 功能 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| API Key 鉴权入口 | `litellm/proxy/auth/user_api_key_auth.py` | `user_api_key_auth()` |
| 主密钥检查 | `litellm/proxy/auth/user_api_key_auth.py:1103-1147` | `secrets.compare_digest` |
| 密钥对象获取 | `litellm/proxy/auth/auth_checks.py:2290-2369` | `get_key_object()` |
| 密钥哈希 | `litellm/proxy/utils.py:4775-4781` | `hash_token()` |
| 缓存密钥对象 | `litellm/proxy/auth/auth_checks.py` | `_cache_key_object()` |
| 删除缓存密钥 | `litellm/proxy/auth/user_api_key_auth.py:1049-1053` | `_delete_cache_key_object()` |
| 加载对象权限 | `litellm/proxy/auth/auth_checks.py:2373-2417` | `get_object_permission()` |

### 7.2 团队与组织信息加载

| 功能 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 获取团队对象 | `litellm/proxy/auth/auth_checks.py` | `get_team_object()` |
| 获取组织对象 | `litellm/proxy/auth/auth_checks.py:2485-...` | `get_org_object()` |
| 获取项目对象 | `litellm/proxy/auth/auth_checks.py` | `get_project_object()` |
| 获取终端用户对象 | `litellm/proxy/auth/auth_checks.py` | `get_end_user_object()` |
| 获取团队成员关系 | `litellm/proxy/auth/auth_checks.py` | `get_team_membership()` |

### 7.3 权限与路由检查

| 功能 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 通用检查入口 | `litellm/proxy/auth/auth_checks.py:451-701` | `common_checks()` |
| 团队模型访问 | `litellm/proxy/auth/auth_checks.py:2881-2920` | `can_team_access_model()` |
| 用户模型访问 | `litellm/proxy/auth/auth_checks.py:2941-2962` | `can_user_call_model()` |
| 项目模型访问 | `litellm/proxy/auth/auth_checks.py:2923-2938` | `can_project_access_model()` |
| 团队成员模型访问 | `litellm/proxy/auth/auth_checks.py:3332-3381` | `_check_team_member_model_access()` |
| 路由权限检查 | `litellm/proxy/auth/auth_checks.py:721-745` | `_is_api_route_allowed()` |
| 路由检查类 | `litellm/proxy/auth/route_checks.py` | `RouteChecks` 类 |
| 组织 RBAC 检查 | `litellm/proxy/auth/auth_checks_organization.py` | `organization_role_based_access_check()` |
| 向量存储访问 | `litellm/proxy/auth/auth_checks.py:684-690` | `vector_store_access_check()` |
| 工具白名单 | `litellm/proxy/auth/auth_checks.py:692-699` | `check_tools_allowlist()` |

### 7.4 预算检查

| 功能 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 获取当前消费 | `litellm/proxy/proxy_server.py` | `get_current_spend()` |
| 团队预算检查 | `litellm/proxy/auth/auth_checks.py:3384-3428` | `_team_max_budget_check()` |
| 团队多窗口预算 | `litellm/proxy/auth/auth_checks.py:3431-3461` | `_team_multi_budget_check()` |
| 团队软预算检查 | `litellm/proxy/auth/auth_checks.py:3464-...` | `_team_soft_budget_check()` |
| 组织预算检查 | `litellm/proxy/auth/auth_checks.py:3689-3781` | `_organization_max_budget_check()` |
| 团队成员预算检查 | `litellm/proxy/auth/auth_checks.py:3266-3329` | `_check_team_member_budget()` |
| 标签预算检查 | `litellm/proxy/auth/auth_checks.py:3784-3833` | `_tag_max_budget_check()` |
| 密钥多窗口预算 | `litellm/proxy/auth/auth_checks.py:2990-...` | `_virtual_key_max_budget_check()` |
| 预算告警 | `litellm/proxy/utils.py` | `proxy_logging_obj.budget_alerts()` |

### 7.5 速率限制与预调用钩子

| 功能 | 文件路径 | 关键函数/类 |
|------|----------|-------------|
| 预调用钩子入口 | `litellm/proxy/utils.py:1394-1445` | `proxy_logging_obj.pre_call_hook()` |
| 动态速率限制器 V3 | `litellm/proxy/hooks/dynamic_rate_limiter_v3.py` | `_PROXY_DynamicRateLimitHandlerV3` |
| 并行请求限制器 V3 | `litellm/proxy/hooks/parallel_request_limiter_v3.py` | `_PROXY_MaxParallelRequestsHandlerV3` |
| 批量速率限制器 | `litellm/proxy/hooks/batch_rate_limiter.py` | 相关实现 |
| 预算限制器 | `litellm/proxy/hooks/max_budget_limiter.py` | `_PROXY_MaxBudgetLimiter` |
| 模型预算限制器 | `litellm/proxy/hooks/model_max_budget_limiter.py` | 相关实现 |
| 会话预算限制器 | `litellm/proxy/hooks/max_budget_per_session_limiter.py` | 相关实现 |
| Guardrail 处理 | `litellm/proxy/utils.py` | `_process_guardrail_callback()` |
| 消费更新回调 | `litellm/proxy/hooks/proxy_track_cost_callback.py` | 相关实现 |

### 7.6 异常与错误类型

| 功能 | 文件路径 | 关键定义 |
|------|----------|----------|
| 代理异常类型 | `litellm/proxy/_types.py` | `ProxyErrorTypes` 枚举 |
| 代理异常类 | `litellm/proxy/_types.py` | `ProxyException` 类 |
| 预算超限异常 | `litellm/__init__.py` | `BudgetExceededError` |
| 用户角色定义 | `litellm/proxy/_types.py` | `LitellmUserRoles` 枚举 |
| 实体类型定义 | `litellm/proxy/_types.py` | `Litellm_EntityType` 枚举 |

---

## 总结

LiteLLM Proxy 的请求处理链路采用 **分层防御、逐级拦截** 的架构设计：

### 核心设计原则

1. **缓存优先**
   - 密钥对象、团队信息、组织信息都优先从缓存获取
   - 缓存未命中时才查询数据库，然后回写缓存
   - 过期密钥会主动从缓存删除

2. **双重保险**
   - 预算检查在 `common_checks` 和 `pre_call_hook` 中都有实现
   - `common_checks` 更全面（团队、组织、标签等）
   - `pre_call_hook` 作为补充（个人密钥场景）

3. **快速失败**
   - 越早的检查越简单、越快速
   - 异常状态（过期、阻塞、越权）尽早拦截
   - 数据库查询作为最后的回退

4. **异步解耦**
   - 告警触发使用 `asyncio.create_task()` 不阻塞主流程
   - 缓存回写也是异步操作
   - 消费更新在 post_call_hook 中异步处理

### 关键拦截点总结

```
请求进入
    │
    ├─[阶段一] 鉴权层 ──────────────────────────────────────────────
    │   │
    │   ├─ 主密钥检查 (secrets.compare_digest)
    │   │       └─ 失败 → 继续缓存查找
    │   │
    │   ├─ 缓存查找 (get_key_object, check_cache_only=True)
    │   │       └─ 未命中 → 继续数据库查找
    │   │
    │   ├─ 数据库查找 (get_key_object, check_cache_only=False)
    │   │       ├─ 未找到 → 401 Unauthorized (token_not_found_in_db)
    │   │       └─ 找到 → 缓存回写
    │   │
    │   ├─ 密钥格式检查 (必须以 sk- 开头)
    │   │       └─ 失败 → AssertionError
    │   │
    │   └─ 密钥过期检查 (expires < now)
    │           └─ 过期 → 400 Bad Request (expired_key) + 删除缓存
    │
    ├─[阶段二] 权限层 (common_checks) ─────────────────────────────
    │   │
    │   ├─ 团队阻塞检查 (team.blocked)
    │   │       └─ 阻塞 → Exception
    │   │
    │   ├─ 模型访问检查 (can_team_access_model 等)
    │   │       └─ 越权 → 401 Unauthorized (team_model_access_denied)
    │   │
    │   ├─ 预算检查 (组织 → 团队 → 成员 → 标签)
    │   │       └─ 超限 → BudgetExceededError + 异步告警
    │   │
    │   ├─ 路由权限检查 (_is_api_route_allowed)
    │   │       └─ 越权 → 403 Forbidden (invalid_route/permission_denied)
    │   │
    │   └─ 对象权限检查 (向量存储、工具等)
    │           └─ 越权 → 相应异常
    │
    ├─[阶段三] 配额层 (pre_call_hook) ─────────────────────────────
    │   │
    │   ├─ Guardrails 检查
    │   │       └─ 拦截 → 相应异常
    │   │
    │   ├─ 速率限制检查 (DynamicRateLimitHandlerV3)
    │   │       └─ 超限 → 429 Too Many Requests
    │   │
    │   ├─ 并行请求限制 (MaxParallelRequestsHandlerV3)
    │   │       └─ 超限 → 429 Too Many Requests
    │   │
    │   └─ 预算检查 (MaxBudgetLimiter) [个人密钥场景]
    │           └─ 超限 → 429 Too Many Requests
    │
    └─[阶段四] 执行层 ──────────────────────────────────────────────
        │
        ├─ 实际 LLM 调用 (route_request)
        │
        └─ 后调用钩子 (post_call_hook)
                ├─ 消费金额更新
                ├─ 速率计数器更新
                └─ 日志记录
```

这种分层设计确保了：
- **安全性**：每一层都有独立的检查机制
- **性能**：缓存优先、快速失败减少不必要的数据库查询
- **可观测性**：每个检查点都有对应的 tracer span 和日志
- **可扩展性**：通过 `litellm.callbacks` 可以轻松添加自定义钩子
