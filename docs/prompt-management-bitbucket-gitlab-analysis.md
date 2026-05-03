# LiteLLM Prompt Management 外部仓库同步分析报告

## 目录
1. [概述](#1-概述)
2. [BitBucket 完整实现分析](#2-bitbucket-完整实现分析)
3. [GitLab 完整实现分析](#3-gitlab-完整实现分析)
4. [BitBucket vs GitLab 关键差异对比](#4-bitbucket-vs-gitlab-关键差异对比)
5. [完整调用链路](#5-完整调用链路)
6. [结论与建议](#6-结论与建议)

---

## 1. 概述

LiteLLM 的 Prompt Management 系统支持多种外部 Git 仓库作为 Prompt 模板的数据源，包括 BitBucket 和 GitLab。本文档深入分析这两种方案的实现细节、设计差异以及完整的调用链路。

### 核心概念

| 概念 | 说明 |
|-----|------|
| `prompt_id` | Prompt 模板的唯一标识符 |
| `prompt_version` | 版本参数（可用于 Git 引用） |
| `git_ref` | Git 引用（tag/branch/SHA，GitLab 专有） |
| `prompts_path` | 目录 scoping 配置（GitLab 专有） |

---

## 2. BitBucket 完整实现分析

### 2.1 架构概览

BitBucket 方案由三个核心组件构成：

```
┌─────────────────────────────────────────────────────────────┐
│                   BitBucketPromptManager                      │
│  ┌─────────────────┐    ┌──────────────────────────────┐   │
│  │ 初始化与配置管理  │───▶│  BitBucketTemplateManager   │   │
│  └─────────────────┘    │  ┌────────────────────────┐ │   │
│                         │  │ BitBucketClient        │ │   │
│                         │  │ - API 调用              │ │   │
│                         │  │ - 认证处理              │ │   │
│                         │  └────────────────────────┘ │   │
│                         │  ┌────────────────────────┐ │   │
│                         │  │ 内存缓存 (self.prompts) │ │   │
│                         │  └────────────────────────┘ │   │
│                         └──────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 BitBucketClient 实现细节

**文件位置**: `litellm/integrations/bitbucket/bitbucket_client.py`

#### 2.2.1 初始化与认证

```python
class BitBucketClient:
    def __init__(self, config: Dict[str, Any]):
        # 必需字段
        self.workspace = config.get("workspace")
        self.repository = config.get("repository")
        self.access_token = config.get("access_token")
        self.branch = config.get("branch", "main")  # 默认分支
        
        # 认证方式
        self.auth_method = config.get("auth_method", "token")
        self.username = config.get("username")
        
        # 认证头设置
        if self.auth_method == "basic" and self.username:
            # Basic Auth: username + app password
            credentials = f"{self.username}:{self.access_token}"
            encoded = base64.b64encode(credentials.encode()).decode()
            self.headers["Authorization"] = f"Basic {encoded}"
        else:
            # Token Auth (默认): Bearer token
            self.headers["Authorization"] = f"Bearer {self.access_token}"
```

**配置字段对比**:

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|-----|------|-----|-------|------|
| `workspace` | str | ✅ | - | BitBucket 工作区名称 |
| `repository` | str | ✅ | - | 仓库名称 |
| `access_token` | str | ✅ | - | 访问令牌或应用密码 |
| `branch` | str | ❌ | `main` | 拉取文件的分支 |
| `auth_method` | str | ❌ | `token` | 认证方式：`token` 或 `basic` |
| `username` | str | ❌ | - | Basic Auth 时的用户名 |

#### 2.2.2 文件拉取机制

```python
def get_file_content(self, file_path: str) -> Optional[str]:
    # API 端点: /repositories/{workspace}/{repo}/src/{branch}/{path}
    url = f"{self.base_url}/repositories/{self.workspace}/{self.repository}/src/{self.branch}/{file_path}"
    
    try:
        response = self.http_handler.get(url, headers=self.headers)
        response.raise_for_status()
        
        # 内容解码
        if response.headers.get("content-type", "").startswith("text/"):
            return response.text
        else:
            # 二进制或 Base64 编码内容
            try:
                return base64.b64decode(response.content).decode("utf-8")
            except Exception:
                return response.text
                
    except Exception as e:
        # 错误处理
        if hasattr(e, "response"):
            if e.response.status_code == 404:
                return None  # 文件不存在
            elif e.response.status_code == 403:
                raise Exception("Access denied")
            elif e.response.status_code == 401:
                raise Exception("Authentication failed")
```

**关键限制**: `get_file_content` 方法**硬编码使用 `self.branch`**，**不支持** 按请求传递的 `git_ref` 动态切换分支或标签。

#### 2.2.3 目录列表

```python
def list_files(self, directory_path: str = "", file_extension: str = ".prompt") -> List[str]:
    url = f"{self.base_url}/repositories/{self.workspace}/{self.repository}/src/{self.branch}/{directory_path}"
    # ... 过滤特定扩展名的文件
```

同样，目录列表也只使用配置中的默认 `branch`。

### 2.3 BitBucketTemplateManager 实现

**文件位置**: `litellm/integrations/bitbucket/bitbucket_prompt_manager.py:54`

#### 2.3.1 初始化

```python
class BitBucketTemplateManager:
    def __init__(self, bitbucket_config: Dict[str, Any], prompt_id: Optional[str] = None):
        self.bitbucket_config = bitbucket_config
        self.prompt_id = prompt_id
        self.prompts: Dict[str, BitBucketPromptTemplate] = {}  # 内存缓存
        self.bitbucket_client = BitBucketClient(bitbucket_config)
        
        # Jinja2 环境（Handlebars 风格分隔符）
        self.jinja_env = Environment(
            variable_start_string="{{",
            variable_end_string="}}",
            block_start_string="{%",
            block_end_string="%}",
        )
        
        # 初始化时加载指定的 prompt
        if self.prompt_id:
            self._load_prompt_from_bitbucket(self.prompt_id)
```

#### 2.3.2 缓存机制

**缓存结构**: 简单的内存字典 `self.prompts: Dict[str, BitBucketPromptTemplate]`

| 特性 | 实现 |
|-----|------|
| 缓存键 | `prompt_id` (字符串) |
| 缓存值 | `BitBucketPromptTemplate` 对象 |
| 缓存类型 | 进程内内存缓存 |
| TTL | 无（永久有效，除非手动刷新） |
| 并发安全 | 未实现 |

**缓存命中逻辑**:

```python
def render_template(self, template_id: str, variables: ...) -> str:
    if template_id not in self.prompts:
        raise ValueError(f"Template '{template_id}' not found")
    # 直接使用已缓存的模板
    template = self.prompts[template_id]
    jinja_template = self.jinja_env.from_string(template.content)
    return jinja_template.render(**(variables or {}))
```

#### 2.3.3 刷新机制

```python
# 在 BitBucketPromptManager 中
def reload_prompts(self) -> None:
    if self.prompt_id:
        self._prompt_manager = None  # 重置管理器实例
        self.prompt_manager  # 访问属性触发重新初始化
```

**刷新策略**: 通过**重置整个管理器实例**实现，而非增量更新。

### 2.4 BitBucketPromptManager 核心逻辑

**文件位置**: `litellm/integrations/bitbucket/bitbucket_prompt_manager.py:185`

#### 2.4.1 初始化器注册

```python
# litellm/integrations/bitbucket/__init__.py

def prompt_initializer(litellm_params: PromptLiteLLMParams, prompt_spec: PromptSpec) -> CustomPromptManagement:
    bitbucket_config = getattr(litellm_params, "bitbucket_config", None)
    prompt_id = getattr(litellm_params, "prompt_id", None)
    
    if not bitbucket_config:
        raise ValueError("bitbucket_config is required")
    
    return BitBucketPromptManager(
        bitbucket_config=bitbucket_config,
        prompt_id=prompt_id,
    )

prompt_initializer_registry = {
    SupportedPromptIntegrations.BITBUCKET.value: prompt_initializer,
}
```

**关键观察**: `prompt_initializer` **不接受** `git_ref` 或类似的版本参数。

#### 2.4.2 编译 Prompt 逻辑

```python
def _compile_prompt_helper(
    self,
    prompt_id: Optional[str],
    prompt_spec: Optional[PromptSpec],
    prompt_variables: Optional[dict],
    dynamic_callback_params: StandardCallbackDynamicParams,
    prompt_label: Optional[str] = None,
    prompt_version: Optional[int] = None,  # ⚠️ 接收但未使用
) -> PromptManagementClient:
    
    if prompt_id not in self.prompt_manager.prompts:
        # 加载时只使用默认 branch，忽略 prompt_version
        self.prompt_manager._load_prompt_from_bitbucket(prompt_id)
    
    # 渲染模板
    rendered_prompt, prompt_metadata = self.get_prompt_template(
        prompt_id, prompt_variables
    )
    
    # 解析为消息
    messages = self._parse_prompt_to_messages(rendered_prompt)
    
    return PromptManagementClient(
        prompt_id=prompt_id,
        prompt_template=messages,
        prompt_template_model=prompt_metadata.get("model"),
        prompt_template_optional_params=optional_params,
        completed_messages=None,
    )
```

**⚠️ 关键发现**: `prompt_version` 参数被接收但**完全未使用**。BitBucket 方案**不支持**按请求级别的版本切换。

#### 2.4.3 Pre-Call Hook

```python
def pre_call_hook(
    self,
    prompt_id: Optional[str] = None,
    prompt_variables: Optional[Dict[str, Any]] = None,
    prompt_version: Optional[str] = None,  # ⚠️ 未使用
    **kwargs,
) -> Tuple[List[AllMessageValues], Optional[Dict[str, Any]]]:
    
    if not prompt_id:
        return messages, litellm_params
    
    # 直接获取模板，不处理 prompt_version
    rendered_prompt, prompt_metadata = self.get_prompt_template(
        prompt_id, prompt_variables
    )
    # ...
```

同样，`pre_call_hook` 中的 `prompt_version` 参数被忽略。

---

## 3. GitLab 完整实现分析

### 3.1 架构概览

GitLab 方案相比 BitBucket 更加完善，增加了版本控制、目录 scoping、专用缓存等高级特性。

```
┌─────────────────────────────────────────────────────────────────┐
│                    GitLabPromptManager                            │
│  ┌─────────────────┐    ┌──────────────────────────────────┐   │
│  │ 初始化与配置管理  │───▶│    GitLabTemplateManager         │   │
│  │ - git_ref 支持   │    │  ┌────────────────────────────┐ │   │
│  │ - _ref_override  │    │  │ GitLabClient              │ │   │
│  └─────────────────┘    │  │ - ref 动态切换            │ │   │
│                         │  │ - 双端点 fallback         │ │   │
│                         │  └────────────────────────────┘ │   │
│                         │  ┌────────────────────────────┐ │   │
│                         │  │ prompts_path 目录 scoping  │ │   │
│                         │  │ - _id_to_repo_path()       │ │   │
│                         │  │ - _repo_path_to_id()       │ │   │
│                         │  └────────────────────────────┘ │   │
│                         │  ┌────────────────────────────┐ │   │
│                         │  │ ID 编码机制                │ │   │
│                         │  │ - encode_prompt_id()      │ │   │
│                         │  │ - decode_prompt_id()      │ │   │
│                         │  └────────────────────────────┘ │   │
│                         └──────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              GitLabPromptCache (专用缓存类)               │  │
│  │  - load_all() 批量加载                                    │  │
│  │  - reload() 刷新                                          │  │
│  │  - get_by_id() / get_by_file() 双索引查询                │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 GitLabClient 实现细节

**文件位置**: `litellm/integrations/gitlab/gitlab_client.py`

#### 3.2.1 初始化与认证

```python
class GitLabClient:
    def __init__(self, config: Dict[str, Any]):
        self.project = config.get("project")  # 支持字符串路径或数字 ID
        self.access_token = config.get("access_token")
        self.branch = config.get("branch", "main")
        self.tag = config.get("tag")  # ⭐ Tag 支持
        
        # ref 优先级: tag > branch
        self.ref = self.tag or self.branch
        
        # 认证方式
        self.auth_method = config.get("auth_method", "token")
        
        if self.auth_method == "oauth":
            self.headers["Authorization"] = f"Bearer {self.access_token}"
        else:
            self.headers["Private-Token"] = self.access_token
```

**配置字段对比**:

| 字段 | 类型 | 必填 | 默认值 | 说明 |
|-----|------|-----|-------|------|
| `project` | str/int | ✅ | - | GitLab 项目（`group/repo` 或数字 ID） |
| `access_token` | str | ✅ | - | 访问令牌 |
| `branch` | str | ❌ | `main` | 默认分支 |
| `tag` | str | ❌ | - | Tag（优先级高于 branch） |
| `auth_method` | str | ❌ | `token` | `token` (Private-Token) 或 `oauth` (Bearer) |

#### 3.2.2 动态 Ref 切换

```python
def set_ref(self, ref: str) -> None:
    """动态设置 Git 引用（tag/branch/SHA）"""
    self.ref = ref
```

这是 GitLab 方案的核心特性之一：**支持按请求动态切换引用**。

#### 3.2.3 文件拉取（支持 Ref 参数）

```python
def get_file_content(self, file_path: str, ref: Optional[str] = None) -> Optional[str]:
    # 使用传入的 ref 或默认 ref
    use_ref = ref or self.ref
    
    # 端点 1: RAW 端点（优先）
    raw_url = f"{self.base_url}/projects/{self.project_encoded}/repository/files/{file_path_encoded}/raw?ref={use_ref}"
    
    try:
        response = self.http_handler.get(raw_url, headers=self.headers)
        if response.status_code == 200:
            return response.text
    except Exception:
        pass  # RAW 失败，尝试 JSON 端点
    
    # 端点 2: JSON 端点（fallback）
    json_url = f"{self.base_url}/projects/{self.project_encoded}/repository/files/{file_path_encoded}?ref={use_ref}"
    
    response = self.http_handler.get(json_url, headers=self.headers)
    data = response.json()
    
    # 解码 Base64 内容
    content = data.get("content", "")
    if data.get("encoding") == "base64":
        content = base64.b64decode(content).decode("utf-8")
    
    return content
```

**关键差异**:
1. ✅ 支持 `ref` 参数动态切换
2. ✅ 双端点 fallback（RAW → JSON）
3. ✅ Tag 优先级高于 Branch

### 3.3 GitLabTemplateManager 实现

**文件位置**: `litellm/integrations/gitlab/gitlab_prompt_manager.py:64`

#### 3.3.1 初始化

```python
class GitLabTemplateManager:
    def __init__(
        self,
        gitlab_config: Dict[str, Any],
        prompt_id: Optional[str] = None,
        ref: Optional[str] = None,  # ⭐ 支持初始化时的 ref
        gitlab_client: Optional[GitLabClient] = None,
    ):
        self.gitlab_config = dict(gitlab_config)
        self.prompt_id = prompt_id
        self.prompts: Dict[str, GitLabPromptTemplate] = {}
        self.gitlab_client = gitlab_client or GitLabClient(self.gitlab_config)
        
        # 应用初始化时的 ref
        if ref:
            self.gitlab_client.set_ref(ref)
        
        # ⭐ 目录 scoping 配置
        self.prompts_path: str = (
            self.gitlab_config.get("prompts_path")
            or self.gitlab_config.get("folder")
            or ""
        ).strip("/")
```

#### 3.3.2 目录 Scoping 机制

这是 GitLab 方案的另一核心特性，支持将 Prompt 文件限定在特定目录下。

```python
def _id_to_repo_path(self, prompt_id: str) -> str:
    """将 prompt_id 映射到仓库路径（考虑 prompts_path）"""
    prompt_id = decode_prompt_id(prompt_id)
    if self.prompts_path:
        return f"{self.prompts_path}/{prompt_id}.prompt"
    return f"{prompt_id}.prompt"

def _repo_path_to_id(self, repo_path: str) -> str:
    """将仓库路径映射回 prompt_id（去掉 prompts_path 前缀和扩展名）"""
    path = repo_path.strip("/")
    if self.prompts_path and path.startswith(self.prompts_path.strip("/") + "/"):
        path = path[len(self.prompts_path.strip("/")) + 1 :]
    if path.endswith(".prompt"):
        path = path[: -len(".prompt")]
    return encode_prompt_id(path)
```

**示例**:

| 配置 | prompt_id | 实际仓库路径 |
|-----|-----------|-------------|
| `prompts_path=""` | `greeting` | `greeting.prompt` |
| `prompts_path="prompts"` | `greeting` | `prompts/greeting.prompt` |
| `prompts_path="prompts/chat"` | `conversation/welcome` | `prompts/chat/conversation/welcome.prompt` |

#### 3.3.3 ID 编码机制

GitLab 支持路径式的 prompt_id（如 `invoice/extract`），通过编码避免与分隔符冲突。

```python
GITLAB_PREFIX = "gitlab::"

def encode_prompt_id(raw_id: str) -> str:
    """Convert GitLab path IDs like 'invoice/extract' → 'gitlab::invoice::extract'"""
    if raw_id.startswith(GITLAB_PREFIX):
        return raw_id  # 已编码
    return f"{GITLAB_PREFIX}{raw_id.replace('/', '::')}"

def decode_prompt_id(encoded_id: str) -> str:
    """Convert 'gitlab::invoice::extract' → 'invoice/extract'"""
    if not encoded_id.startswith(GITLAB_PREFIX):
        return encoded_id
    return encoded_id[len(GITLAB_PREFIX) :].replace("::", "/")
```

#### 3.3.4 按 Ref 加载 Prompt

```python
def _load_prompt_from_gitlab(
    self, prompt_id: str, *, ref: Optional[str] = None  # ⭐ 支持 ref 参数
) -> None:
    file_path = self._id_to_repo_path(prompt_id)
    # 传递 ref 到客户端
    prompt_content = self.gitlab_client.get_file_content(file_path, ref=ref)
    
    if prompt_content:
        template = self._parse_prompt_file(prompt_content, prompt_id)
        self.prompts[prompt_id] = template
```

### 3.4 GitLabPromptManager 核心逻辑

**文件位置**: `litellm/integrations/gitlab/gitlab_prompt_manager.py:269`

#### 3.4.1 四级版本优先级

这是 GitLab 方案最核心的设计：**四级版本优先级机制**。

```python
def __init__(
    self,
    gitlab_config: Dict[str, Any],
    prompt_id: Optional[str] = None,
    ref: Optional[str] = None,  # Level 3: Manager 级 ref_override
    gitlab_client: Optional[GitLabClient] = None,
):
    # ...
    self._ref_override = ref  # 保存 Manager 级覆盖
```

**Pre-Call Hook 中的优先级处理**:

```python
def pre_call_hook(
    self,
    prompt_version: Optional[str] = None,   # Level 1: 最高优先级
    **kwargs,
):
    # ⭐ 四级优先级: prompt_version > git_ref kwarg > _ref_override > config default
    git_ref = (
        prompt_version                    # Level 1: 显式 prompt_version
        or kwargs.get("git_ref")          # Level 2: 调用时的 git_ref 参数
        or self._ref_override             # Level 3: Manager 级覆盖
    )                                    # Level 4: 配置中的 branch/tag（默认）
    
    # 使用确定的 git_ref 拉取 prompt
    rendered_prompt, prompt_metadata = self.get_prompt_template(
        prompt_id, prompt_variables, ref=git_ref
    )
```

**优先级图解**:

```
                    ┌────────────────────────────────────┐
                    │  Level 1: prompt_version (请求级)  │ ← 最高优先级
                    │  (用户请求中显式传递的版本参数)      │
                    └─────────────────┬──────────────────┘
                                      ▼ 若未设置
                    ┌────────────────────────────────────┐
                    │  Level 2: git_ref (调用级)         │
                    │  (kwargs.get("git_ref"))           │
                    └─────────────────┬──────────────────┘
                                      ▼ 若未设置
                    ┌────────────────────────────────────┐
                    │  Level 3: _ref_override (Manager 级)│
                    │  (初始化 Manager 时传入的 ref)      │
                    └─────────────────┬──────────────────┘
                                      ▼ 若未设置
                    ┌────────────────────────────────────┐
                    │  Level 4: 配置默认值               │ ← 最低优先级
                    │  (gitlab_config 中的 branch/tag)   │
                    └────────────────────────────────────┘
```

#### 3.4.2 编译 Prompt 逻辑（带版本支持）

```python
def _compile_prompt_helper(
    self,
    prompt_id: Optional[str],
    prompt_version: Optional[int] = None,
    dynamic_callback_params: StandardCallbackDynamicParams,
    **kwargs,
) -> PromptManagementClient:
    
    decoded_id = decode_prompt_id(prompt_id)
    
    if decoded_id not in self.prompt_manager.prompts:
        # 从 dynamic_callback_params 中提取 git_ref
        git_ref = (
            getattr(dynamic_callback_params, "extra", {}).get("git_ref")
            if hasattr(dynamic_callback_params, "extra")
            else None
        )
        # 使用 git_ref 加载
        self.prompt_manager._load_prompt_from_gitlab(decoded_id, ref=git_ref)
    
    # ... 渲染逻辑
```

### 3.5 GitLabPromptCache 专用缓存类

**文件位置**: `litellm/integrations/gitlab/gitlab_prompt_manager.py:608`

GitLab 提供了**专用的缓存类**，支持批量加载、双索引查询、显式刷新等高级功能。

```python
class GitLabPromptCache:
    """
    将 GitLab 仓库中的所有 .prompt 文件缓存到内存。
    
    - 双索引存储:
      - _by_file: 仓库文件路径 → JSON 对象
      - _by_id: 编码后的 prompt_id → JSON 对象
    
    - 公共 API:
      - load_all(): 批量扫描并加载所有 .prompt 文件
      - reload(): 清空并重新加载
      - get_by_id(): 按 prompt_id 查询
      - get_by_file(): 按仓库路径查询
    """
    
    def __init__(self, gitlab_config: Dict[str, Any], *, ref: Optional[str] = None, ...):
        # 内部构建 PromptManager
        self.prompt_manager = GitLabPromptManager(
            gitlab_config=gitlab_config,
            prompt_id=None,
            ref=ref,
            gitlab_client=gitlab_client,
        )
        self.template_manager: GitLabTemplateManager = self.prompt_manager.prompt_manager
        
        # 双索引存储
        self._by_file: Dict[str, Dict[str, Any]] = {}
        self._by_id: Dict[str, Dict[str, Any]] = {}
    
    def load_all(self, *, recursive: bool = True) -> Dict[str, Dict[str, Any]]:
        """批量加载所有 .prompt 文件"""
        # 1. 列出所有模板 ID
        ids = self.template_manager.list_templates(recursive=recursive)
        
        for pid in ids:
            # 2. 确保模板已加载
            if pid not in self.template_manager.prompts:
                self.template_manager._load_prompt_from_gitlab(pid)
            
            tmpl = self.template_manager.get_template(pid)
            
            # 3. 构建缓存条目
            file_path = self.template_manager._id_to_repo_path(pid)
            entry = self._template_to_json(pid, tmpl)
            
            # 4. 双索引存储
            self._by_file[file_path] = entry
            self._by_id[encode_prompt_id(pid)] = entry
        
        return self._by_id
    
    def reload(self, *, recursive: bool = True) -> Dict[str, Dict[str, Any]]:
        """刷新缓存"""
        self._by_file.clear()
        self._by_id.clear()
        return self.load_all(recursive=recursive)
    
    def get_by_id(self, prompt_id: str) -> Optional[Dict[str, Any]]:
        """按 prompt_id 查询（自动处理编码/解码）"""
        if prompt_id in self._by_id:
            return self._by_id[prompt_id]
        
        # 尝试标准化形式
        decoded = decode_prompt_id(prompt_id)
        encoded = encode_prompt_id(decoded)
        return self._by_id.get(encoded) or self._by_id.get(decoded)
```

**缓存条目结构**:

```python
{
    "id": "greet/hi",                    # prompt_id（解码后）
    "path": "prompts/chat/greet/hi.prompt", # 仓库完整路径
    "content": "Hello {{ name }}!",      # 模板内容（无前 matter）
    "metadata": {                         # 解析后的 YAML frontmatter
        "model": "gpt-4",
        "temperature": 0.7
    },
    "model": "gpt-4",                     # 提取的 model
    "temperature": 0.7,                   # 提取的参数
    "max_tokens": 1000,
    "optional_params": {...}              # 其他可选参数
}
```

---

## 4. BitBucket vs GitLab 关键差异对比

### 4.1 功能特性矩阵

| 特性 | BitBucket | GitLab | 说明 |
|-----|-----------|--------|------|
| **版本控制** | ❌ 仅支持固定分支 | ✅ 四级优先级 | GitLab 支持 tag/branch/SHA 动态切换 |
| **目录 Scoping** | ❌ 不支持 | ✅ `prompts_path` | GitLab 可限定 Prompt 搜索目录 |
| **专用缓存类** | ❌ 无 | ✅ `GitLabPromptCache` | GitLab 有批量加载、双索引查询 |
| **ID 编码机制** | ❌ 无 | ✅ `encode/decode_prompt_id` | GitLab 支持路径式 ID（如 `a/b/c`） |
| **按 Ref 拉取** | ❌ 不支持 | ✅ `get_file_content(..., ref=...)` | GitLab 每次请求可指定不同 ref |
| **Tag 支持** | ❌ 仅分支 | ✅ `tag` 配置（优先级高于 branch） | GitLab 支持按 Tag 拉取 |
| **双端点 Fallback** | ❌ 单端点 | ✅ RAW → JSON | GitLab 更健壮的文件获取 |
| **初始化器参数** | 仅 `bitbucket_config` | `gitlab_config` + `git_ref` | GitLab 支持 Manager 级 ref 覆盖 |

### 4.2 代码层面详细对比

#### 4.2.1 初始化器对比

**BitBucket** (`litellm/integrations/bitbucket/__init__.py`):

```python
def prompt_initializer(litellm_params, prompt_spec):
    bitbucket_config = getattr(litellm_params, "bitbucket_config", None)
    prompt_id = getattr(litellm_params, "prompt_id", None)
    # ❌ 没有 git_ref 或类似参数
    return BitBucketPromptManager(
        bitbucket_config=bitbucket_config,
        prompt_id=prompt_id,
    )
```

**GitLab** (`litellm/integrations/gitlab/__init__.py`):

```python
def _gitlab_prompt_initializer(litellm_params, prompt):
    gitlab_config: Dict[str, Any] = getattr(litellm_params, "gitlab_config", None) or {}
    git_ref: Optional[str] = getattr(litellm_params, "git_ref", None)  # ✅ 支持
    
    return GitLabPromptManager(
        gitlab_config=gitlab_config,
        prompt_id=prompt.prompt_id,
        ref=git_ref,  # ✅ 传递到 Manager
    )
```

#### 4.2.2 客户端文件拉取对比

**BitBucket Client**:

```python
def get_file_content(self, file_path: str) -> Optional[str]:
    # ❌ 硬编码使用 self.branch，不支持 ref 参数
    url = f"{self.base_url}/repositories/{self.workspace}/{self.repository}/src/{self.branch}/{file_path}"
    # ...
```

**GitLab Client**:

```python
def get_file_content(self, file_path: str, ref: Optional[str] = None) -> Optional[str]:
    # ✅ 支持 ref 参数动态切换
    use_ref = ref or self.ref
    raw_url = f"{self.base_url}/projects/{self.project_encoded}/repository/files/{file_path_encoded}/raw?ref={use_ref}"
    # ...
```

#### 4.2.3 Pre-Call Hook 版本处理对比

**BitBucket**:

```python
def pre_call_hook(self, prompt_version: Optional[str] = None, **kwargs):
    # ⚠️ prompt_version 被接收但完全忽略
    rendered_prompt, prompt_metadata = self.get_prompt_template(
        prompt_id, prompt_variables  # 没有传递 ref
    )
```

**GitLab**:

```python
def pre_call_hook(self, prompt_version: Optional[str] = None, **kwargs):
    # ✅ 四级优先级处理
    git_ref = prompt_version or kwargs.get("git_ref") or self._ref_override
    
    rendered_prompt, prompt_metadata = self.get_prompt_template(
        prompt_id, prompt_variables, ref=git_ref  # 传递 ref
    )
```

### 4.3 配置对比

#### BitBucket 配置

```python
bitbucket_config = {
    "workspace": "my-workspace",      # 必需
    "repository": "my-repo",           # 必需
    "access_token": "atc_xxx",         # 必需
    "branch": "main",                  # 可选，默认 main
    "auth_method": "token",            # 可选，token 或 basic
    "username": "user@example.com",    # basic auth 时需要
}
```

#### GitLab 配置

```python
gitlab_config = {
    "project": "group/subgroup/repo",  # 必需（或数字 ID）
    "access_token": "glpat_xxx",        # 必需
    "branch": "main",                   # 可选，默认 main
    "tag": "v1.2.3",                    # 可选，优先级高于 branch
    "prompts_path": "prompts/chat",     # 可选，目录 scoping
    "auth_method": "token",              # 可选，token 或 oauth
}
```

### 4.4 缓存策略对比

| 维度 | BitBucket | GitLab |
|-----|-----------|--------|
| **缓存实现** | 简单 `Dict[str, Template]` | 专用 `GitLabPromptCache` 类 |
| **加载方式** | 按需单文件加载 | 支持 `load_all()` 批量加载 |
| **索引方式** | 仅按 prompt_id | 双索引（by_id + by_file） |
| **刷新方式** | 重置整个 Manager 实例 | `reload()` 增量可控 |
| **查询优化** | 无 | `get_by_id()` 自动处理编码/解码 |

---

## 5. 完整调用链路

### 5.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           用户请求                                         │
│  {                                                                        │
│    "model": "gpt-4",                                                     │
│    "prompt_id": "my_prompt",                                             │
│    "prompt_version": "v1.2.3",      # GitLab 支持，BitBucket 忽略       │
│    "prompt_variables": {"name": "World"},                                │
│    "messages": [...]                                                      │
│  }                                                                        │
└─────────────────────────────┬───────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Proxy 层: _process_prompt_template                    │
│  litellm/proxy/utils.py:1128-1188                                        │
│                                                                           │
│  Step 1: 构建版本化 lookup_id                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ if prompt_version is None:                                           │ │
│  │     lookup_prompt_id = get_latest_version_prompt_id(prompt_id=...) │ │
│  │ else:                                                                 │ │
│  │     lookup_prompt_id = construct_versioned_prompt_id(               │ │
│  │         prompt_id, version=prompt_version                            │ │
│  │     )                                                                 │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  Step 2: 从注册表获取 Prompt Manager                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ custom_logger = IN_MEMORY_PROMPT_REGISTRY.get_prompt_callback_by_id(│ │
│  │     lookup_prompt_id                                                  │ │
│  │ )                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  Step 3: 调用 Logging 层进行 Prompt 编译                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ (model, messages, optional_params) = await litellm_logging_obj.     │ │
│  │     async_get_chat_completion_prompt(                                │ │
│  │         prompt_id=litellm_prompt_id,                                 │ │
│  │         prompt_spec=prompt_spec,                                      │ │
│  │         prompt_management_logger=custom_logger,                      │ │
│  │         prompt_variables=data.pop("prompt_variables", None),         │ │
│  │         prompt_label=data.pop("prompt_label", None),                 │ │
│  │         prompt_version=data.pop("prompt_version", None),  # ⭐ 传递 │ │
│  │     )                                                                 │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  Step 4: 更新请求数据（完成注入）                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │ data.update(optional_params)                                          │ │
│  │ data["model"] = model                                                 │ │
│  │ data["messages"] = messages                                           │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────┬───────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Logging 层: async_get_chat_completion_prompt         │
│  职责: 协调 Prompt Manager 完成模板编译和参数提取                          │
└─────────────────────────────┬───────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Prompt Manager 层（BitBucket vs GitLab 差异点）             │
│                                                                           │
│  ┌──────────────────────────┐    ┌──────────────────────────────────┐  │
│  │   BitBucketPromptManager │    │      GitLabPromptManager         │  │
│  │                          │    │                                  │  │
│  │ ❌ 忽略 prompt_version   │    │ ✅ 四级优先级处理:               │  │
│  │    参数                   │    │    prompt_version               │  │
│  │                          │    │    → git_ref (kwargs)           │  │
│  │ ❌ 仅使用配置中的 branch  │    │    → _ref_override              │  │
│  │                          │    │    → 配置默认 branch/tag        │  │
│  │ ❌ 无目录 scoping        │    │                                  │  │
│  │                          │    │ ✅ 目录 scoping: prompts_path   │  │
│  │ ❌ 无 ID 编码            │    │                                  │  │
│  │                          │    │ ✅ ID 编码: encode/decode       │  │
│  └──────────────────────────┘    └──────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────────────┘
                              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Client 层（API 调用）                              │
│                                                                           │
│  ┌──────────────────────────┐    ┌──────────────────────────────────┐  │
│  │     BitBucketClient      │    │         GitLabClient             │  │
│  │                          │    │                                  │  │
│  │ 端点:                     │    │ 端点（双 fallback）:             │  │
│  │ /repositories/{workspace}│    │ 1. /projects/{id}/repository/   │  │
│  │   /{repo}/src/{branch}/  │    │      files/{path}/raw?ref={ref} │  │
│  │   {path}                  │    │ 2. /projects/{id}/repository/   │  │
│  │                          │    │      files/{path}?ref={ref}     │  │
│  │ ❌ 无 ref 参数           │    │                                  │  │
│  │                          │    │ ✅ ref 参数支持动态切换          │  │
│  └──────────────────────────┘    └──────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 关键代码位置

| 阶段 | 文件 | 行号 | 说明 |
|-----|------|-----|------|
| Proxy 入口 | `litellm/proxy/utils.py` | 1128-1188 | `_process_prompt_template` |
| 注册表查找 | `litellm/prompt_management.py` | - | `IN_MEMORY_PROMPT_REGISTRY` |
| Logging 层编译 | `litellm/litellm_core_utils/litellm_logging.py` | - | `async_get_chat_completion_prompt` |
| BitBucket Manager | `litellm/integrations/bitbucket/bitbucket_prompt_manager.py` | 185 | `BitBucketPromptManager` |
| GitLab Manager | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | 269 | `GitLabPromptManager` |
| GitLab 缓存 | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | 608 | `GitLabPromptCache` |

---

## 6. 结论与建议

### 6.1 核心发现

1. **BitBucket 方案是简化实现**
   - 仅支持固定分支拉取，不支持动态版本切换
   - 无目录 scoping，Prompt 文件必须在仓库根目录
   - 缓存机制简单，无专用缓存类
   - 适合简单、不需要版本管理的场景

2. **GitLab 方案是完整实现**
   - 四级版本优先级机制，支持灵活的版本控制
   - 目录 scoping 支持复杂的仓库结构
   - 专用缓存类支持批量加载和高效查询
   - ID 编码机制支持路径式的 Prompt 组织
   - 适合需要严格版本管理、复杂目录结构的企业级场景

3. **版本参数的命运差异**
   - **Proxy 层**: 两个方案都接收 `prompt_version` 参数
   - **BitBucket**: 参数被接收但完全忽略
   - **GitLab**: 参数作为最高优先级参与版本决策

### 6.2 架构建议

#### 如果需要增强 BitBucket 方案

1. **添加 `git_ref` 支持**
   - 修改 `BitBucketClient.get_file_content()` 添加 `ref` 参数
   - 修改 `BitBucketTemplateManager._load_prompt_from_bitbucket()` 传递 `ref`
   - 在 `BitBucketPromptManager.pre_call_hook()` 中实现版本优先级

2. **添加目录 scoping**
   - 参考 GitLab 的 `prompts_path` 实现
   - 添加 `_id_to_repo_path()` 和 `_repo_path_to_id()` 方法

3. **考虑专用缓存类**
   - 对于大型仓库，批量加载和双索引查询会显著提升性能

#### 如果选择 GitLab 方案

1. **合理使用版本优先级**
   - 开发环境：使用 `branch` 配置默认分支
   - 测试环境：使用 `_ref_override` 或 `git_ref` 指定测试分支
   - 生产环境：使用 `tag` 或 `prompt_version` 锁定特定版本

2. **利用目录 scoping 组织 Prompt**
   ```python
   # 推荐的目录结构
   gitlab_config = {
       "prompts_path": "prompts",
       # 实际文件:
       # prompts/chat/welcome.prompt
       # prompts/chat/goodbye.prompt
       # prompts/analysis/sentiment.prompt
   }
   ```

3. **使用 `GitLabPromptCache` 优化性能**
   - 启动时 `load_all()` 预热缓存
   - 定时 `reload()` 刷新（或通过 Webhook 触发）
   - 使用 `get_by_id()` 快速查询

### 6.3 测试用例参考

从测试文件中可以看到两个方案的测试覆盖差异：

**GitLab 额外测试的特性**:

```python
# 目录 scoping 测试
def test_gitlab_prompt_manager_prompts_path_resolution_and_version():
    """prompts_path + explicit prompt_version should produce correct repo path and ref."""
    # ...
    mock_client.get_file_content.assert_any_call(
        "prompts/chat/folder/sub/my_prompt.prompt", ref="commit-sha-999"
    )

# 四级优先级测试
def test_gitlab_prompt_manager_version_precedence():
    """
    prompt_version > git_ref kwarg > manager _ref_override.
    """
    # Level 1: prompt_version wins
    mock_client.get_file_content.assert_any_call("pA.prompt", ref="sha-111")
    
    # Level 2: git_ref kwarg
    mock_client.get_file_content.assert_any_call("pB.prompt", ref="hotfix/ref-2")
    
    # Level 3: manager _ref_override
    mock_client.get_file_content.assert_any_call("pC.prompt", ref="manager-default")

# ID 编码测试
def test_encode_decode_prompt_id_roundtrip():
    raw = "invoice/extract"
    encoded = encode_prompt_id(raw)
    assert encoded == "gitlab::invoice::extract"
    assert decode_prompt_id(encoded) == raw
```

---

## 附录

### A. 相关文件清单

| 文件路径 | 说明 |
|---------|------|
| `litellm/integrations/bitbucket/bitbucket_client.py` | BitBucket API 客户端 |
| `litellm/integrations/bitbucket/bitbucket_prompt_manager.py` | BitBucket Prompt 管理器 |
| `litellm/integrations/bitbucket/__init__.py` | BitBucket 初始化器注册 |
| `litellm/integrations/gitlab/gitlab_client.py` | GitLab API 客户端 |
| `litellm/integrations/gitlab/gitlab_prompt_manager.py` | GitLab Prompt 管理器 + 缓存 |
| `litellm/integrations/gitlab/__init__.py` | GitLab 初始化器注册 |
| `litellm/proxy/utils.py` | Proxy 层 Prompt 处理入口 |
| `litellm/integrations/custom_prompt_management.py` | 所有 Prompt Manager 基类 |

### B. 版本参数传递链

```
用户请求
    │
    ▼
┌───────────────────┐
│ data 字典包含:    │
│ - prompt_id       │
│ - prompt_version  │ ← 用户传入的版本参数
│ - prompt_variables│
│ - prompt_label    │
└─────────┬─────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│ _process_prompt_template (proxy/utils.py)           │
│                                                      │
│ # Step 1: 构建 lookup_id (如果使用内部注册表)        │
│ # Step 2: 从注册表获取 Manager                       │
│                                                      │
│ # Step 3: 传递参数到 Logging 层                      │
│ (model, messages, optional_params) = await          │
│     litellm_logging_obj.async_get_chat_completion_  │
│     prompt(                                          │
│         prompt_version=data.pop("prompt_version"),  │ ← ⭐ 从 data 中提取
│         ...                                          │
│     )                                                │
└─────────┬───────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│ async_get_chat_completion_prompt (litellm_logging) │
│                                                      │
│ # 调用 Manager 的 async_compile_prompt_helper       │
│ return await custom_logger.async_compile_prompt_    │
│     helper(                                          │
│         prompt_version=prompt_version,              │ ← ⭐ 继续传递
│         ...                                          │
│     )                                                │
└─────────┬───────────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────────┐
│ 分叉: BitBucket vs GitLab                           │
│                                                      │
│  BitBucketPromptManager._compile_prompt_helper:     │
│  ┌─────────────────────────────────────────────┐    │
│  │ prompt_version 参数被接收但**完全未使用**    │    │
│  │ 仅使用配置中的默认 branch 拉取文件            │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
│  GitLabPromptManager._compile_prompt_helper:        │
│  ┌─────────────────────────────────────────────┐    │
│  │ ✅ 从 dynamic_callback_params.extra 中      │    │
│  │    提取 git_ref                              │    │
│  │ ✅ 使用 git_ref 调用 _load_prompt_from_     │    │
│  │    gitlab(prompt_id, ref=git_ref)          │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
```

### C. API 端点对比

| 操作 | BitBucket 端点 | GitLab 端点 |
|-----|---------------|-------------|
| **获取文件** | `GET /repositories/{workspace}/{repo}/src/{branch}/{path}` | `GET /projects/{id}/repository/files/{path}/raw?ref={ref}` (优先)<br>`GET /projects/{id}/repository/files/{path}?ref={ref}` (fallback) |
| **列出文件** | `GET /repositories/{workspace}/{repo}/src/{branch}/{dir}` | `GET /projects/{id}/repository/tree?path={path}&ref={ref}` |
| **获取仓库信息** | `GET /repositories/{workspace}/{repo}` | `GET /projects/{id}` |
| **获取分支列表** | `GET /repositories/{workspace}/{repo}/refs/branches` | `GET /projects/{id}/repository/branches` |

---

**报告生成时间**: 2026-05-03  
**分析版本**: LiteLLM 当前代码库 (commit: 分析时状态)  
**分析范围**: BitBucket + GitLab Prompt Management 集成
