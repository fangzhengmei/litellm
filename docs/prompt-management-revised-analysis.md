# LiteLLM Prompt Management 外部仓库同步复核报告（修订版）

> **复核日期**: 2025年
> **复核重点**: 请求级版本切换真实生效路径、缓存命中行为差异、BitBucket 配置异常影响
> **文档状态**: 已修订，基于代码深度分析

---

## 目录
1. [执行摘要](#1-执行摘要)
2. [关键发现总览](#2-关键发现总览)
3. [版本参数传递链深度分析](#3-版本参数传递链深度分析)
4. [缓存机制设计缺陷分析](#4-缓存机制设计缺陷分析)
5. [BitBucket 配置异常影响分析](#5-bitbucket-配置异常影响分析)
6. [结论与建议](#6-结论与建议)
7. [附录：关键代码位置](#7-附录关键代码位置)

---

## 1. 执行摘要

### 1.1 核心结论（已修订）

经过代码深度复核，发现原分析报告存在以下关键偏差：

| 原结论 | 复核修正结论 |
|-------|-------------|
| GitLab 完全支持四级版本优先级 | **仅限 `pre_call_hook` 路径**；`_compile_prompt_helper`（Proxy 层）路径存在设计缺陷 |
| 版本参数在缓存命中时正确处理 | **缓存设计缺陷**：缓存键不包含 `ref`，缓存命中时版本参数被忽略 |
| 配置异常有优雅降级 | **BitBucket 配置异常立即抛出异常**，导致 Manager 初始化失败，无降级机制 |

### 1.2 重大问题清单

| 问题编号 | 问题描述 | 影响等级 | 文件位置 |
|---------|---------|---------|---------|
| P01 | `_compile_prompt_helper` 中 `git_ref` 从 `dynamic_callback_params.extra` 提取，但 `StandardCallbackDynamicParams` 无 `extra` 字段 | **高** | `litellm/integrations/gitlab/gitlab_prompt_manager.py:484-488` |
| P02 | `get_prompt_template` 缓存逻辑：仅缓存未命中时使用 `ref`，缓存命中时忽略 | **高** | `litellm/integrations/gitlab/gitlab_prompt_manager.py:317-341` |
| P03 | BitBucket 配置缺失时立即抛出 `ValueError`，无降级机制 | **中** | `litellm/integrations/bitbucket/bitbucket_client.py:44-45` |

---

## 2. 关键发现总览

### 2.1 两套调用路径行为差异

LiteLLM Prompt Management 存在**两套独立的调用路径**，行为**不一致**：

```
                    ┌─────────────────────────────────┐
                    │      用户请求                    │
                    │  {prompt_id, prompt_version}    │
                    └───────────────┬─────────────────┘
                                    │
                    ┌───────────────▼─────────────────┐
                    │   入口层: 调用方式决定路径        │
                    └───────────────┬─────────────────┘
                                    │
            ┌───────────────────────┴───────────────────────┐
            │                                                 │
            ▼                                                 ▼
┌───────────────────────────┐              ┌───────────────────────────────┐
│  Path A: pre_call_hook    │              │ Path B: _compile_prompt_helper│
│  (直接调用 litellm.completion) │         │ (Proxy 层调用)                  │
├───────────────────────────┤              ├───────────────────────────────┤
│ ✅ 四级优先级正确生效      │              │ ❌ git_ref 无法提取           │
│ ✅ prompt_version 作为最高级 │            │ ❌ StandardCallbackDynamicParams│
│ ✅ git_ref kwargs 支持     │              │   无 extra 字段                │
└───────────────────────────┘              └───────────────────────────────┘
```

### 2.2 缓存设计缺陷的影响

```
场景：用户请求
- 第一次请求：prompt_id="greeting", ref="main"
- 第二次请求：prompt_id="greeting", ref="v1.0"

实际行为：
┌─────────────────────────────────────────────────────────────┐
│ 第一次请求（缓存未命中）                                       │
│   └──► 检查 "greeting" not in self.prompts                   │
│       └──► 调用 _load_prompt_from_gitlab("greeting", ref="main") │
│           └──► 从 main 分支拉取，缓存 key="greeting"          │
├─────────────────────────────────────────────────────────────┤
│ 第二次请求（缓存命中）                                         │
│   └──► 检查 "greeting" in self.prompts ✅                     │
│       └──► ⚠️ 直接返回缓存，ref="v1.0" 被忽略！               │
│           └──► 返回的仍是 main 分支的内容！                   │
└─────────────────────────────────────────────────────────────┘
```

**问题根源**：缓存键仅为 `prompt_id`，不包含 `ref`。

### 2.3 BitBucket 配置异常的连锁反应

```
配置初始化流程：
┌─────────────────────────────────────────────────────────────┐
│ BitBucketPromptManager.__init__()                            │
│   └──► BitBucketTemplateManager.__init__()                   │
│       └──► BitBucketClient.__init__(config)                  │
│           └──► 检查必需字段:                                   │
│               if not all([workspace, repository, access_token]): │
│                   raise ValueError(...)  ← 立即抛出！         │
└─────────────────────────────────────────────────────────────┘
```

**影响链**：
1. 配置缺失 → `BitBucketClient.__init__` 抛出 `ValueError`
2. `BitBucketTemplateManager` 初始化失败
3. `BitBucketPromptManager` 初始化失败
4. **整个 Prompt Management 功能无法使用**

---

## 3. 版本参数传递链深度分析

### 3.1 整体传递架构

```
用户请求层级
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  Level 0: 用户请求参数                                           │
│  {                                                                │
│    "prompt_id": "my_prompt",                                     │
│    "prompt_version": "v1.2.3",   ← 可选，版本参数               │
│    "prompt_variables": {"name": "World"}                         │
│  }                                                                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Level 1: Proxy 层入口 (_process_prompt_template)               │
│  litellm/proxy/utils.py:1128-1188                               │
│                                                                  │
│  Step 1: 从 data 中提取版本参数                                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ prompt_version = data.pop("prompt_version", None)           ││
│  │ # ⭐ 正确提取，但后续路径决定如何使用                          ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  Step 2: 传递到 Logging 层                                       │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ (model, messages, optional_params) = await                   ││
│  │     litellm_logging_obj.async_get_chat_completion_prompt(   ││
│  │         prompt_version=prompt_version,  ← ⭐ 继续传递        ││
│  │         ...                                                   ││
│  │     )                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  Level 2: 路径分叉点                                              │
│  根据调用方式选择不同路径                                          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            │                                 │
            ▼                                 ▼
┌───────────────────────────┐     ┌───────────────────────────────┐
│  Path A: pre_call_hook    │     │ Path B: _compile_prompt_helper│
│  (直接 litellm.completion) │     │ (Proxy 服务调用)               │
├───────────────────────────┤     ├───────────────────────────────┤
│ ✅ 四级优先级正确生效      │     │ ❌ git_ref 无法提取           │
│ ✅ 版本参数参与决策        │     │ ❌ 类型定义不支持 extra 字段   │
└───────────────────────────┘     └───────────────────────────────┘
```

### 3.2 Path A: pre_call_hook 路径（正确实现）

**文件位置**: `litellm/integrations/gitlab/gitlab_prompt_manager.py:343-393`

#### 3.2.1 四级优先级实现

```python
def pre_call_hook(
    self,
    prompt_version: Optional[str] = None,  # Level 1: 最高优先级
    **kwargs,
):
    # ⭐ 四级优先级实现（正确）
    git_ref = (
        prompt_version                    # Level 1: 显式 prompt_version
        or kwargs.get("git_ref")          # Level 2: 调用时的 git_ref 参数
        or self._ref_override             # Level 3: Manager 级覆盖
    )                                    # Level 4: 配置中的 branch/tag（默认）

    # ⭐ 正确传递 ref 到 get_prompt_template
    rendered_prompt, prompt_metadata = self.get_prompt_template(
        prompt_id, prompt_variables, ref=git_ref
    )
```

#### 3.2.2 优先级图解

```
                    ┌────────────────────────────────────┐
                    │  Level 1: prompt_version (请求级)  │ ← 最高优先级
                    │  pre_call_hook 参数                │
                    └─────────────────┬──────────────────┘
                                      ▼ 若未设置
                    ┌────────────────────────────────────┐
                    │  Level 2: git_ref (kwargs 级)     │
                    │  kwargs.get("git_ref")             │
                    └─────────────────┬──────────────────┘
                                      ▼ 若未设置
                    ┌────────────────────────────────────┐
                    │  Level 3: _ref_override (Manager 级)│
                    │  初始化 Manager 时传入的 ref        │
                    └─────────────────┬──────────────────┘
                                      ▼ 若未设置
                    ┌────────────────────────────────────┐
                    │  Level 4: 配置默认值               │ ← 最低优先级
                    │  gitlab_config 中的 branch/tag     │
                    └────────────────────────────────────┘
```

### 3.3 Path B: _compile_prompt_helper 路径（存在缺陷）

**文件位置**: `litellm/integrations/gitlab/gitlab_prompt_manager.py:469-517`

#### 3.3.1 问题代码分析

```python
def _compile_prompt_helper(
    self,
    prompt_version: Optional[int] = None,  # ⚠️ 接收但未直接使用
    dynamic_callback_params: StandardCallbackDynamicParams,
):
    decoded_id = decode_prompt_id(prompt_id)
    
    if decoded_id not in self.prompt_manager.prompts:
        # ⚠️ 尝试从 dynamic_callback_params.extra 提取 git_ref
        git_ref = (
            getattr(dynamic_callback_params, "extra", {}).get("git_ref")
            if hasattr(dynamic_callback_params, "extra")
            else None
        )
        # ⚠️ 问题：StandardCallbackDynamicParams 没有 extra 字段！
        
        self.prompt_manager._load_prompt_from_gitlab(decoded_id, ref=git_ref)
```

#### 3.3.2 类型定义验证

**文件位置**: `litellm/types/utils.py:2951-2991`

```python
class StandardCallbackDynamicParams(TypedDict, total=False):
    # Langfuse dynamic params
    langfuse_public_key: Optional[str]
    langfuse_secret: Optional[str]
    langfuse_secret_key: Optional[str]
    langfuse_host: Optional[str]
    langfuse_prompt_version: Optional[int]
    
    # GCS dynamic params
    gcs_bucket_name: Optional[str]
    gcs_path_service_account: Optional[str]
    
    # Langsmith dynamic params
    langsmith_api_key: Optional[str]
    langsmith_project: Optional[str]
    langsmith_base_url: Optional[str]
    langsmith_sampling_rate: Optional[float]
    langsmith_tenant_id: Optional[str]
    
    # Humanloop dynamic params
    humanloop_api_key: Optional[str]
    
    # Arize dynamic params
    arize_api_key: Optional[str]
    arize_space_key: Optional[str]
    arize_space_id: Optional[str]
    
    # PostHog dynamic params
    posthog_api_key: Optional[str]
    posthog_api_url: Optional[str]
    
    # Weave (W&B) dynamic params
    wandb_api_key: Optional[str]
    weave_project_id: Optional[str]
    
    # Logging settings
    turn_off_message_logging: Optional[bool]
    litellm_disabled_callbacks: Optional[List[str]]
    
    # ⚠️ 注意：没有 extra 字段！
```

#### 3.3.3 路径对比总结

| 维度 | Path A: pre_call_hook | Path B: _compile_prompt_helper |
|-----|------------------------|---------------------------------|
| **版本参数来源** | `prompt_version` 参数 + `git_ref` kwargs | 尝试从 `dynamic_callback_params.extra.git_ref` |
| **四级优先级** | ✅ 完整实现 | ❌ 未实现 |
| **类型支持** | ✅ 参数直接传递 | ❌ `StandardCallbackDynamicParams` 无 `extra` 字段 |
| **实际效果** | ✅ 版本参数生效 | ❌ `git_ref` 永远是 `None` |
| **适用场景** | 直接调用 `litellm.completion()` | 通过 Proxy 服务调用 |

---

## 4. 缓存机制设计缺陷分析

### 4.1 缓存逻辑代码分析

**文件位置**: `litellm/integrations/gitlab/gitlab_prompt_manager.py:317-341`

```python
def get_prompt_template(
    self,
    prompt_id: str,
    prompt_variables: Optional[Dict[str, Any]] = None,
    *,
    ref: Optional[str] = None,  # ⚠️ 接收 ref 参数
) -> Tuple[str, Dict[str, Any]]:
    
    # ⚠️ 问题 1: 缓存键仅为 prompt_id，不包含 ref
    if prompt_id not in self.prompt_manager.prompts:
        # ⚠️ 只有缓存未命中时才使用 ref
        self.prompt_manager._load_prompt_from_gitlab(prompt_id, ref=ref)
    
    # ⚠️ 问题 2: 缓存命中时，直接返回缓存，ref 被完全忽略！
    template = self.prompt_manager.get_template(prompt_id)
    
    if not template:
        raise ValueError(f"Prompt template '{prompt_id}' not found")
    
    # 渲染逻辑
    rendered_prompt = self.prompt_manager.render_template(
        prompt_id, prompt_variables or {}
    )
    
    return rendered_prompt, metadata
```

### 4.2 问题场景演示

#### 场景：同一 prompt_id，不同版本

```python
# 假设场景：
# - main 分支: greeting.prompt = "Hello, {{ name }}!"
# - v1.0 tag:  greeting.prompt = "Hi {{ name }}, welcome to v1.0!"

# 第一次请求（缓存未命中）
result1 = manager.get_prompt_template(
    "greeting", 
    {"name": "Alice"}, 
    ref="main"
)
# 行为：
# 1. "greeting" not in prompts → 加载
# 2. _load_prompt_from_gitlab("greeting", ref="main")
# 3. 从 main 分支拉取 "Hello, {{ name }}!"
# 4. 缓存 key="greeting"
# 5. 返回 "Hello, Alice!" ✅ 正确

# 第二次请求（缓存命中，但 ref 不同）
result2 = manager.get_prompt_template(
    "greeting", 
    {"name": "Bob"}, 
    ref="v1.0"  # ⚠️ 期望使用 v1.0 tag
)
# 实际行为：
# 1. "greeting" in prompts → ✅ 缓存命中
# 2. ⚠️ 直接返回缓存，ref="v1.0" 被忽略！
# 3. 返回 "Hello, Bob!" ❌ 错误！应该是 "Hi Bob, welcome to v1.0!"
```

### 4.3 缓存设计对比

| 维度 | 当前实现 | 正确实现 |
|-----|---------|---------|
| **缓存键** | `prompt_id` | `(prompt_id, ref)` 或 `prompt_id:ref` |
| **缓存未命中** | 使用 `ref` 加载 | 使用 `ref` 加载 |
| **缓存命中** | 忽略 `ref`，直接返回 | 检查 `ref` 是否匹配，不匹配则重新加载 |
| **多版本支持** | ❌ 不支持 | ✅ 支持 |
| **版本切换** | ❌ 切换无效 | ✅ 切换有效 |

### 4.4 影响范围

| 场景 | 影响 |
|-----|------|
| **同一进程内多次请求** | 第一次请求的版本决定后续所有请求的内容 |
| **显式版本切换** | `ref` 参数被忽略，切换无效 |
| **不同分支测试** | 无法在同一进程内测试不同分支的 Prompt |
| **热更新场景** | 新 tag/branch 的内容无法生效，除非进程重启 |

---

## 5. BitBucket 配置异常影响分析

### 5.1 配置检查代码

**文件位置**: `litellm/integrations/bitbucket/bitbucket_client.py:44-45`

```python
def __init__(self, config: Dict[str, Any]):
    self.workspace = config.get("workspace")
    self.repository = config.get("repository")
    self.access_token = config.get("access_token")
    self.branch = config.get("branch", "main")
    
    # ⚠️ 立即检查，缺失则抛出异常
    if not all([self.workspace, self.repository, self.access_token]):
        raise ValueError("workspace, repository, and access_token are required")
```

### 5.2 初始化链条

```
初始化流程（同步执行，无异步包装）：

BitBucketPromptManager.__init__
    │
    ▼
BitBucketTemplateManager.__init__
    │
    ├──► self.bitbucket_client = BitBucketClient(bitbucket_config)
    │       │
    │       ▼
    │       BitBucketClient.__init__
    │           │
    │           ├──► 读取 config 字段
    │           │
    │           └──► if not all([workspace, repository, access_token]):
    │                    raise ValueError(...)  ← 立即抛出！
    │
    └──► if self.prompt_id:
            self._load_prompt_from_bitbucket(self.prompt_id)
```

### 5.3 异常影响链

```
┌─────────────────────────────────────────────────────────────────┐
│  异常 1: workspace 缺失                                           │
│  config = {"repository": "my-repo", "access_token": "xxx"}      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  BitBucketClient.__init__                                         │
│  └──► raise ValueError("workspace, repository, and access_token  │
│                        are required")                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  BitBucketTemplateManager.__init__                                │
│  └──► 异常向上传播，初始化失败                                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  BitBucketPromptManager.__init__                                 │
│  └──► 异常向上传播，初始化失败                                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  最终结果                                                          │
│  - Prompt Manager 无法初始化                                       │
│  - 整个 Prompt Management 功能不可用                               │
│  - 无降级机制，无优雅失败                                           │
└─────────────────────────────────────────────────────────────────┘
```

### 5.4 配置字段分析

| 字段 | 类型 | 必填 | 默认值 | 缺失时行为 |
|-----|------|-----|-------|-----------|
| `workspace` | str | ✅ | - | 立即抛出 `ValueError` |
| `repository` | str | ✅ | - | 立即抛出 `ValueError` |
| `access_token` | str | ✅ | - | 立即抛出 `ValueError` |
| `branch` | str | ❌ | `main` | 使用默认值 |
| `auth_method` | str | ❌ | `token` | 使用默认值 |
| `username` | str | ❌ | - | Basic Auth 时需要 |

### 5.5 与 GitLab 对比

| 维度 | BitBucket | GitLab |
|-----|-----------|--------|
| **配置检查时机** | 初始化时立即检查 | 初始化时检查 |
| **缺失字段处理** | 立即抛出 `ValueError` | 类似行为 |
| **降级机制** | ❌ 无 | ❌ 无 |
| **配置灵活性** | 仅固定分支 | 支持 `branch` 和 `tag` |

---

## 6. 结论与建议

### 6.1 核心问题总结

#### 问题 1: 两套调用路径行为不一致

**现状**:
- `pre_call_hook` 路径：四级优先级正确实现
- `_compile_prompt_helper` 路径：`git_ref` 从不存在的 `extra` 字段提取，永远为 `None`

**影响**:
- 通过 Proxy 服务调用时，版本参数实际无效
- 用户文档与实际行为不一致
- 测试用例仅覆盖 `pre_call_hook` 路径，未发现此问题

**建议修复**:
```python
# 方案 A: 在 _compile_prompt_helper 中实现四级优先级
def _compile_prompt_helper(
    self,
    prompt_version: Optional[int] = None,
    dynamic_callback_params: StandardCallbackDynamicParams,
):
    # ✅ 实现四级优先级，与 pre_call_hook 一致
    git_ref = (
        str(prompt_version) if prompt_version else None
        or self._ref_override
    )
    # ...
```

#### 问题 2: 缓存设计缺陷

**现状**:
- 缓存键仅为 `prompt_id`，不包含 `ref`
- 缓存命中时，`ref` 参数被完全忽略

**影响**:
- 同一进程内无法切换版本
- 不同分支/标签的 Prompt 无法共存
- 显式版本切换参数无效

**建议修复**:
```python
# 方案 A: 缓存键包含 ref
def get_prompt_template(self, prompt_id, ..., ref=None):
    cache_key = f"{prompt_id}:{ref or 'default'}"
    
    if cache_key not in self.prompt_manager.prompts:
        self.prompt_manager._load_prompt_from_gitlab(prompt_id, ref=ref)
    
    template = self.prompt_manager.get_template(cache_key)
    # ...

# 方案 B: 缓存命中时检查 ref，不匹配则刷新
def get_prompt_template(self, prompt_id, ..., ref=None):
    if prompt_id in self.prompt_manager.prompts:
        # 检查缓存的 ref 是否匹配
        cached_ref = self.prompt_manager.prompts[prompt_id].loaded_ref
        if ref is not None and cached_ref != ref:
            # ref 不匹配，重新加载
            del self.prompt_manager.prompts[prompt_id]
    
    if prompt_id not in self.prompt_manager.prompts:
        self.prompt_manager._load_prompt_from_gitlab(prompt_id, ref=ref)
    # ...
```

#### 问题 3: BitBucket 配置异常无降级

**现状**:
- 配置缺失时立即抛出 `ValueError`
- 无降级机制，无优雅失败

**影响**:
- 配置错误导致整个 Prompt Management 功能失效
- 无法渐进式启用/禁用

**建议修复**:
```python
# 方案 A: 延迟初始化 + 降级机制
class BitBucketPromptManager:
    def __init__(self, bitbucket_config, prompt_id=None):
        self.bitbucket_config = bitbucket_config
        self.prompt_id = prompt_id
        self._prompt_manager = None
        self._init_failed = False
        self._init_error = None
    
    @property
    def prompt_manager(self):
        if self._init_failed:
            # 降级：返回 None 或抛出特定异常
            raise ValueError(f"Prompt manager initialization failed: {self._init_error}")
        
        if self._prompt_manager is None:
            try:
                self._prompt_manager = BitBucketTemplateManager(
                    self.bitbucket_config, self.prompt_id
                )
            except Exception as e:
                self._init_failed = True
                self._init_error = str(e)
                raise
        
        return self._prompt_manager
```

### 6.2 测试覆盖建议

当前测试用例存在以下覆盖缺口：

| 测试场景 | 当前覆盖 | 建议补充 |
|---------|---------|---------|
| `pre_call_hook` 路径版本优先级 | ✅ 有测试 | 保持 |
| `_compile_prompt_helper` 路径 | ❌ 无测试 | 补充 |
| 缓存命中时版本切换 | ❌ 无测试 | 补充 |
| 同一 prompt_id 不同 ref | ❌ 无测试 | 补充 |
| BitBucket 配置异常处理 | ❌ 无测试 | 补充 |

### 6.3 架构建议

1. **统一两套调用路径**
   - 提取版本优先级逻辑到公共方法
   - 确保 `pre_call_hook` 和 `_compile_prompt_helper` 行为一致

2. **重构缓存机制**
   - 缓存键应包含版本标识
   - 或实现缓存版本校验机制

3. **增强配置异常处理**
   - 实现延迟初始化
   - 提供降级机制
   - 增加配置验证的提前检查

---

## 7. 附录：关键代码位置

### 7.1 GitLab 相关

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| `pre_call_hook` 四级优先级 | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | 343-393 |
| `get_prompt_template` 缓存逻辑 | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | 317-341 |
| `_compile_prompt_helper` 版本处理 | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | 469-517 |
| `_load_prompt_from_gitlab` | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | 64-150 |

### 7.2 BitBucket 相关

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| 配置检查（抛出异常） | `litellm/integrations/bitbucket/bitbucket_client.py` | 44-45 |
| `get_file_content`（无 ref 支持） | `litellm/integrations/bitbucket/bitbucket_client.py` | 65-107 |
| `_compile_prompt_helper`（忽略版本） | `litellm/integrations/bitbucket/bitbucket_prompt_manager.py` | 230-259 |

### 7.3 类型定义

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| `StandardCallbackDynamicParams` | `litellm/types/utils.py` | 2951-2991 |

### 7.4 Proxy 层入口

| 功能 | 文件路径 | 行号 |
|-----|---------|-----|
| `_process_prompt_template` | `litellm/proxy/utils.py` | 1128-1188 |

---

## 修订记录

| 版本 | 日期 | 修订内容 |
|-----|------|---------|
| v2.0 | 2025 | 复核修订版，发现并记录三大关键问题 |
| v1.0 | - | 初始分析报告（存在偏差） |

---

**文档状态**: 已完成复核分析，建议提交给开发团队评估修复优先级。
