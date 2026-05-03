# LiteLLM Prompt 管理集成与外部仓库同步路径分析报告

> 分析日期: 2026-05-03
> 分析对象: LiteLLM (BerriAI/litellm)

---

## 目录

1. [概述](#1-概述)
2. [外部 Prompt 管理平台对接机制](#2-外部-prompt-管理平台对接机制)
3. [Prompt 版本从远端仓库拉取流程](#3-prompt-版本从远端仓库拉取流程)
4. [Prompt 注入到请求流程的机制](#4-prompt-注入到请求流程的机制)
5. [缓存、刷新和版本切换的协作实现](#5-缓存刷新和版本切换的协作实现)
6. [核心模块索引](#6-核心模块索引)
7. [架构图](#7-架构图)

---

## 1. 概述

LiteLLM 提供了一套统一的 Prompt 管理集成系统，支持与多种外部 Prompt 管理平台和代码仓库对接。该系统实现了：

- **多平台统一接口**: 抽象出 `CustomPromptManagement` 基类，支持多种平台实现
- **版本管理**: 支持多版本 Prompt，支持从 Git 标签/分支拉取
- **缓存机制**: 内存缓存减少重复网络请求
- **自动发现**: 通过注册表自动发现可用的 Prompt 集成
- **请求注入**: 在 LLM 请求流程中自动应用 Prompt 模板

### 1.1 核心设计理念

```
用户请求 → LiteLLM → Prompt Manager (检测 prompt_id)
                ↓
         从远端仓库拉取 Prompt
                ↓
         应用变量渲染模板
                ↓
         合并用户消息
                ↓
         发送给实际 LLM
```

---

## 2. 外部 Prompt 管理平台对接机制

### 2.1 支持的平台列表

| 平台名称 | 集成类型 | 核心实现模块 | 配置方式 |
|---------|---------|-------------|---------|
| **GitLab** | 代码仓库 | `litellm/integrations/gitlab/` | `gitlab_config` |
| **BitBucket** | 代码仓库 | `litellm/integrations/bitbucket/` | `bitbucket_config` |
| **Generic API** | 通用 API | `litellm/integrations/generic_prompt_management/` | `api_base` + `api_key` |
| **Dotprompt** | 本地文件 | `litellm/integrations/dotprompt/` | `prompt_directory` |
| **Langfuse** | Prompt 平台 | `litellm/integrations/langfuse/langfuse_prompt_management.py` | `langfuse_config` |
| **Arize Phoenix** | Prompt 平台 | `litellm/integrations/arize/arize_phoenix_prompt_manager.py` | `arize_phoenix_config` |

### 2.2 统一接口设计

所有 Prompt 管理平台都继承自 `CustomPromptManagement` 基类，该类定义了标准接口：

```python
# 位于: litellm/integrations/custom_prompt_management.py

class CustomPromptManagement(CustomLogger, PromptManagementBase):
    @property
    def integration_name(self) -> str:
        """返回集成名称，如 'dotprompt', 'gitlab'"""
        return "custom-prompt-management"

    def should_run_prompt_management(
        self, prompt_id, prompt_spec, dynamic_callback_params
    ) -> bool:
        """判断是否应该运行此 Prompt 管理"""
        return True

    def get_chat_completion_prompt(
        self, model, messages, non_default_params, 
        prompt_id, prompt_variables, dynamic_callback_params,
        prompt_spec, prompt_label, prompt_version, ...
    ) -> Tuple[str, List[AllMessageValues], dict]:
        """
        核心方法：返回处理后的 (model, messages, params) 三元组
        """
        return model, messages, non_default_params

    def _compile_prompt_helper(
        self, prompt_id, prompt_spec, prompt_variables, ...
    ) -> PromptManagementClient:
        """编译 Prompt 模板"""
        raise NotImplementedError()

    async def async_compile_prompt_helper(
        self, prompt_id, prompt_variables, ...
    ) -> PromptManagementClient:
        """异步编译 Prompt 模板"""
        raise NotImplementedError()
```

### 2.3 注册表发现机制

LiteLLM 通过自动发现机制加载 Prompt 集成：

```python
# 位于: litellm/proxy/prompts/prompt_registry.py

def get_prompt_initializer_from_integrations():
    """
    扫描 integrations 目录，发现具有 prompt_initializer_registry 的模块
    """
    # 1. 获取 integrations 目录路径
    current_dir = Path(__file__).parent.parent.parent
    integrations_dir = os.path.join(current_dir, "integrations")
    
    # 2. 遍历子目录
    for item in os.listdir(integrations_dir):
        item_path = os.path.join(integrations_dir, item)
        
        # 3. 检查是否有 __init__.py
        init_file = os.path.join(item_path, "__init__.py")
        if not os.path.exists(init_file):
            continue
        
        # 4. 动态导入模块
        module_path = f"litellm.integrations.{item}"
        module = importlib.import_module(module_path)
        
        # 5. 检查是否有 prompt_initializer_registry
        if hasattr(module, "prompt_initializer_registry"):
            registry = getattr(module, "prompt_initializer_registry")
            discovered_initializers.update(registry)

    return discovered_initializers

# 全局注册表
prompt_initializer_registry = get_prompt_initializer_from_integrations()
```

### 2.4 各平台注册示例

**GitLab 平台注册** (`litellm/integrations/gitlab/__init__.py`):
```python
prompt_initializer_registry = {
    SupportedPromptIntegrations.GITLAB.value: _gitlab_prompt_initializer,
}

def _gitlab_prompt_initializer(
    litellm_params: PromptLiteLLMParams,
    prompt: PromptSpec,
) -> CustomPromptManagement:
    # 从 litellm_params 提取配置
    gitlab_config: Dict[str, Any] = getattr(litellm_params, "gitlab_config", None)
    git_ref: Optional[str] = getattr(litellm_params, "git_ref", None)
    
    # 创建 GitLabPromptManager 实例
    return GitLabPromptManager(
        gitlab_config=gitlab_config,
        prompt_id=prompt.prompt_id,
        ref=git_ref,
    )
```

**Generic API 注册** (`litellm/integrations/generic_prompt_management/__init__.py`):
```python
prompt_initializer_registry = {
    SupportedPromptIntegrations.GENERIC_PROMPT_MANAGEMENT.value: prompt_initializer,
}

def prompt_initializer(
    litellm_params: PromptLiteLLMParams, 
    prompt_spec: PromptSpec
) -> CustomPromptManagement:
    # 提取 API 配置
    api_base = litellm_params.api_base
    api_key = litellm_params.api_key
    provider_specific_query_params = litellm_params.provider_specific_query_params
    
    return GenericPromptManager(
        api_base=api_base,
        api_key=api_key,
        prompt_id=prompt_id,
        additional_provider_specific_query_params=provider_specific_query_params,
        **litellm_params.model_dump(...),
    )
```

### 2.5 配置参数结构 (`PromptLiteLLMParams`)

```python
# 位于: litellm/types/prompts/init_prompts.py

class PromptLiteLLMParams(BaseModel):
    prompt_id: Optional[str] = None
    prompt_integration: str  # 如 "dotprompt", "gitlab"
    
    # Generic API 专用
    api_base: Optional[str] = None
    api_key: Optional[str] = None
    provider_specific_query_params: Optional[Dict[str, Any]] = None
    
    # 模型参数控制
    ignore_prompt_manager_model: Optional[bool] = False
    ignore_prompt_manager_optional_params: Optional[bool] = False
    
    # Dotprompt 内容（用于数据库存储）
    dotprompt_content: Optional[str] = None
    
    # 额外字段（平台扩展专用）
    model_config = ConfigDict(extra="allow", protected_namespaces=())
```

---

## 3. Prompt 版本从远端仓库拉取流程

### 3.1 GitLab 集成详解

GitLab 是一个典型的代码仓库集成，支持从 GitLab 仓库拉取 `.prompt` 文件。

#### 3.1.1 客户端实现

```python
# 位于: litellm/integrations/gitlab/gitlab_client.py

class GitLabClient:
    def __init__(self, config: Dict[str, Any]):
        self.project: str | int = config.get("project")  # "group/subgroup/repo" 或数字 ID
        self.access_token: str = str(config.get("access_token"))
        self.auth_method = config.get("auth_method", "token")  # 'token' 或 'oauth'
        self.branch = config.get("branch", "main")
        self.tag = config.get("tag")  # tag 优先级高于 branch
        self.base_url = config.get("base_url", "https://gitlab.com/api/v4")
        
        # 有效 ref: 优先使用 tag，否则使用 branch
        self.ref = str(self.tag or self.branch)
        
        # 构建 HTTP 头
        self.headers = {
            "Accept": "application/json",
            "Content-Type": "application/json",
        }
        if self.auth_method == "oauth":
            self.headers["Authorization"] = f"Bearer {self.access_token}"
        else:
            self.headers["Private-Token"] = self.access_token
    
    def get_file_content(
        self, file_path: str, *, ref: Optional[str] = None
    ) -> Optional[str]:
        """
        获取文件内容
        策略: 1) 尝试 RAW 端点 2) 回退到 JSON 端点 (base64 编码)
        """
        # 构建 URL: /projects/{id}/repository/files/{path}/raw?ref={ref}
        raw_url = self._file_raw_url(file_path, ref=ref)
        
        resp = self.http_handler.get(raw_url, headers=self.headers)
        if resp.status_code == 404:
            # 回退到 JSON 端点
            return self._get_file_content_via_json(file_path, ref=ref)
        resp.raise_for_status()
        
        return resp.text

    def list_files(
        self, directory_path: str = "", file_extension: str = ".prompt",
        recursive: bool = False, *, ref: Optional[str] = None
    ) -> List[str]:
        """
        列出目录中的文件
        使用 /projects/{id}/repository/tree API
        """
        url = self._tree_url(directory_path, recursive=recursive, ref=ref)
        resp = self.http_handler.get(url, headers=self.headers)
        data = resp.json() or []
        
        files: List[str] = []
        for item in data:
            if item.get("type") == "blob":
                file_path = item.get("path", "")
                if not file_extension or file_path.endswith(file_extension):
                    files.append(file_path)
        return files
```

#### 3.1.2 Prompt 管理器实现

```python
# 位于: litellm/integrations/gitlab/gitlab_prompt_manager.py

class GitLabPromptManager(CustomPromptManagement):
    def __init__(
        self, gitlab_config: Dict[str, Any],
        prompt_id: Optional[str] = None,
        ref: Optional[str] = None,  # tag/branch/SHA 覆盖
        gitlab_client: Optional[GitLabClient] = None,
    ):
        self.gitlab_config = gitlab_config
        self.prompt_id = prompt_id
        self._prompt_manager: Optional[GitLabTemplateManager] = None
        self._ref_override = ref
        self._injected_gitlab_client = gitlab_client
        
        # 预加载 prompt
        if self.prompt_id:
            self._prompt_manager = GitLabTemplateManager(
                gitlab_config=self.gitlab_config,
                prompt_id=self.prompt_id,
                ref=self._ref_override,
            )
    
    @property
    def prompt_manager(self) -> GitLabTemplateManager:
        """延迟加载 TemplateManager"""
        if self._prompt_manager is None:
            self._prompt_manager = GitLabTemplateManager(
                gitlab_config=self.gitlab_config,
                prompt_id=self.prompt_id,
                ref=self._ref_override,
                gitlab_client=self._injected_gitlab_client,
            )
        return self._prompt_manager

    def _compile_prompt_helper(
        self, prompt_id: Optional[str], prompt_spec: Optional[PromptSpec],
        prompt_variables: Optional[dict], ...
    ) -> PromptManagementClient:
        if prompt_id is None:
            raise ValueError("prompt_id is required")
        
        # 1. 如果 prompt 未加载，从 GitLab 拉取
        decoded_id = decode_prompt_id(prompt_id)
        if decoded_id not in self.prompt_manager.prompts:
            git_ref = getattr(dynamic_callback_params, "extra", {}).get("git_ref")
            self.prompt_manager._load_prompt_from_gitlab(decoded_id, ref=git_ref)
        
        # 2. 获取并渲染 prompt
        rendered_prompt, prompt_metadata = self.get_prompt_template(
            prompt_id, prompt_variables
        )
        
        # 3. 转换为消息格式
        messages = self._parse_prompt_to_messages(rendered_prompt)
        
        # 4. 提取模型和参数
        template_model = prompt_metadata.get("model")
        optional_params = {}
        for param in ["temperature", "max_tokens", "top_p", 
                      "frequency_penalty", "presence_penalty"]:
            if param in prompt_metadata:
                optional_params[param] = prompt_metadata[param]
        
        return PromptManagementClient(
            prompt_id=prompt_id,
            prompt_template=messages,
            prompt_template_model=template_model,
            prompt_template_optional_params=optional_params,
            completed_messages=None,
        )
```

#### 3.1.3 模板管理器

```python
class GitLabTemplateManager:
    def __init__(
        self, gitlab_config: Dict[str, Any],
        prompt_id: Optional[str] = None, ref: Optional[str] = None,
        gitlab_client: Optional[GitLabClient] = None,
    ):
        self.gitlab_config = dict(gitlab_config)
        self.prompt_id = prompt_id
        self.prompts: Dict[str, GitLabPromptTemplate] = {}
        self.gitlab_client = gitlab_client or GitLabClient(self.gitlab_config)
        
        # 支持 prompts_path 配置（仓库中的子目录）
        self.prompts_path: str = (
            self.gitlab_config.get("prompts_path")
            or self.gitlab_config.get("folder")
            or ""
        ).strip("/")
        
        # Jinja2 模板环境
        self.jinja_env = Environment(
            loader=DictLoader({}),
            autoescape=select_autoescape(["html", "xml"]),
            variable_start_string="{{",
            variable_end_string="}}",
            ...
        )
        
        if self.prompt_id:
            self._load_prompt_from_gitlab(self.prompt_id)
    
    def _load_prompt_from_gitlab(
        self, prompt_id: str, *, ref: Optional[str] = None
    ) -> None:
        """从 GitLab 加载单个 .prompt 文件"""
        # 1. 转换 ID 到仓库路径
        file_path = self._id_to_repo_path(prompt_id)
        
        # 2. 从 GitLab API 获取内容
        prompt_content = self.gitlab_client.get_file_content(file_path, ref=ref)
        
        # 3. 解析为 PromptTemplate
        if prompt_content:
            template = self._parse_prompt_file(prompt_content, prompt_id)
            self.prompts[prompt_id] = template
    
    def _parse_prompt_file(self, content: str, prompt_id: str) -> GitLabPromptTemplate:
        """
        解析 .prompt 文件格式 (符合 Dotprompt 规范)
        
        文件格式示例:
        ---
        model: gpt-4
        temperature: 0.7
        ---
        System: 你是一个 {role} 助手
        
        User: 请处理这个请求
        """
        # 1. 分割 frontmatter 和内容
        if content.startswith("---"):
            parts = content.split("---", 2)
            if len(parts) >= 3:
                frontmatter_str = parts[1].strip()
                template_content = parts[2].strip()
            else:
                frontmatter_str = ""
                template_content = content
        else:
            frontmatter_str = ""
            template_content = content
        
        # 2. 解析 YAML frontmatter
        metadata: Dict[str, Any] = {}
        if frontmatter_str:
            import yaml
            metadata = yaml.safe_load(frontmatter_str) or {}
        
        # 3. 创建模板对象
        return GitLabPromptTemplate(
            template_id=prompt_id,
            content=template_content,
            metadata=metadata,
        )
```

### 3.2 Generic API 集成详解

Generic API 允许对接任何实现了 `/beta/litellm_prompt_management` 端点的服务。

```python
# 位于: litellm/integrations/generic_prompt_management/generic_prompt_manager.py

class GenericPromptManager(CustomPromptManagement):
    def __init__(
        self, api_base: str, api_key: Optional[str] = None,
        timeout: int = 30, prompt_id: Optional[str] = None,
        additional_provider_specific_query_params: Optional[Dict[str, Any]] = None,
        **kwargs,
    ):
        super().__init__(**kwargs)
        self.api_base = api_base.rstrip("/")
        self.api_key = api_key
        self.timeout = timeout
        self.prompt_id = prompt_id
        self.additional_provider_specific_query_params = additional_provider_specific_query_params
        self._prompt_cache: Dict[str, PromptManagementClient] = {}  # 内存缓存
    
    def _fetch_prompt_from_api(
        self, prompt_id: Optional[str], prompt_spec: Optional[PromptSpec]
    ) -> Dict[str, Any]:
        """
        从外部 API 获取 Prompt
        端点: GET {api_base}/beta/litellm_prompt_management?prompt_id={id}
        """
        if prompt_id is None and prompt_spec is None:
            raise ValueError("prompt_id or prompt_spec is required")
        
        url = f"{self.api_base}/beta/litellm_prompt_management"
        params = {
            "prompt_id": prompt_id,
            **(self.additional_provider_specific_query_params or {}),
        }
        http_client = _get_httpx_client()
        
        response = http_client.get(
            url, params=params, headers=self._get_headers()
        )
        response.raise_for_status()
        return response.json()
    
    def _parse_api_response(
        self, prompt_id: Optional[str], prompt_spec: Optional[PromptSpec],
        api_response: Dict[str, Any]
    ) -> PromptManagementClient:
        """
        解析 API 响应
        
        期望的响应格式:
        {
            "prompt_id": "string",
            "prompt_template": [
                {"role": "system", "content": "..."},
                {"role": "user", "content": "..."}
            ],
            "prompt_template_model": "gpt-4",        // 可选
            "prompt_template_optional_params": {     // 可选
                "temperature": 0.7,
                "max_tokens": 100
            }
        }
        """
        return PromptManagementClient(
            prompt_id=prompt_id,
            prompt_template=api_response.get("prompt_template", []),
            prompt_template_model=api_response.get("prompt_template_model"),
            prompt_template_optional_params=api_response.get(
                "prompt_template_optional_params"
            ),
            completed_messages=None,
        )
    
    def _common_caching_logic(
        self, prompt_id: Optional[str],
        prompt_label: Optional[str] = None,
        prompt_version: Optional[int] = None,
        prompt_variables: Optional[dict] = None,
    ) -> Optional[PromptManagementClient]:
        """
        统一的缓存检查逻辑
        缓存键: {prompt_id}:{prompt_label}:{prompt_version}
        """
        cache_key = self._get_cache_key(prompt_id, prompt_label, prompt_version)
        if cache_key in self._prompt_cache:
            cached_prompt = self._prompt_cache[cache_key]
            if prompt_variables:
                return self._apply_variables(cached_prompt, prompt_variables)
            return cached_prompt
        return None
    
    def _compile_prompt_helper(
        self, prompt_id: Optional[str], prompt_spec: Optional[PromptSpec],
        prompt_variables: Optional[dict], ...
    ) -> PromptManagementClient:
        """编译 Prompt - 包含缓存检查"""
        # 1. 检查缓存
        cached_prompt = self._common_caching_logic(
            prompt_id=prompt_id,
            prompt_label=prompt_label,
            prompt_version=prompt_version,
            prompt_variables=prompt_variables,
        )
        if cached_prompt:
            return cached_prompt
        
        cache_key = self._get_cache_key(prompt_id, prompt_label, prompt_version)
        
        # 2. 从 API 获取
        api_response = self._fetch_prompt_from_api(prompt_id, prompt_spec)
        
        # 3. 解析响应
        prompt_client = self._parse_api_response(
            prompt_id, prompt_spec, api_response
        )
        
        # 4. 存入缓存
        self._prompt_cache[cache_key] = prompt_client
        
        # 5. 应用变量（如果有）
        if prompt_variables:
            prompt_client = self._apply_variables(prompt_client, prompt_variables)
        
        return prompt_client
    
    def _apply_variables(
        self, prompt_client: PromptManagementClient,
        variables: Dict[str, Any]
    ) -> PromptManagementClient:
        """
        对 Prompt 内容应用变量替换
        支持 {variable} 和 {{variable}} 两种语法
        """
        updated_messages: List[AllMessageValues] = []
        for message in prompt_client["prompt_template"]:
            updated_message = dict(message)
            if "content" in updated_message and isinstance(
                updated_message["content"], str
            ):
                content = updated_message["content"]
                for key, value in variables.items():
                    content = content.replace(f"{{{key}}}", str(value))
                    content = content.replace(
                        f"{{{{{key}}}}}", str(value)
                    )
                updated_message["content"] = content
            updated_messages.append(updated_message)
        
        return PromptManagementClient(
            prompt_id=prompt_client["prompt_id"],
            prompt_template=updated_messages,
            prompt_template_model=prompt_client["prompt_template_model"],
            prompt_template_optional_params=prompt_client[
                "prompt_template_optional_params"
            ],
            completed_messages=None,
        )
```

### 3.3 Dotprompt 格式规范

所有平台都使用兼容 Dotprompt 的格式：

```yaml
---
model: gpt-4o
temperature: 0.7
max_tokens: 1024
input:
  schema:
    name: string
    context: string
output:
  format: json
---

System: 你是一个专业的 {role} 助手。

User: 请根据以下上下文回答问题：
{context}

问题: {question}
```

---

## 4. Prompt 注入到请求流程的机制

### 4.1 整体调用链

```
litellm.completion()
       ↓
main.py / responses/main.py
       ↓
Logging.get_chat_completion_prompt()
       ↓
get_custom_logger_for_prompt_management()  # 查找对应的 Prompt Manager
       ↓
_custom_logger.get_chat_completion_prompt()  # 调用具体实现
       ↓
PromptManagementBase.get_chat_completion_prompt()  # 基类处理
       ↓
compile_prompt() → _compile_prompt_helper()  # 编译 Prompt
       ↓
post_compile_prompt_processing()  # 后处理（合并消息、更新参数）
       ↓
返回 (model, messages, non_default_params) 三元组
```

### 4.2 核心入口点

#### 4.2.1 在主流程中的调用位置

**同步调用** (`litellm/main.py:1331-1340`):
```python
# 使用 Prompt 管理
if (
    prompt_id is not None
    or prompts is not None
    or prompt_spec is not None
):
    model, messages, optional_params = litellm_logging_obj.get_chat_completion_prompt(
        model=model,
        messages=messages,
        non_default_params=non_default_params,
        prompt_variables=prompt_variables,
        prompt_id=prompt_id,
        prompt_spec=prompt_spec,
        prompt_label=prompt_label,
        prompt_version=prompt_version,
    )
```

**异步调用** (`litellm/main.py:495-505`):
```python
# 使用 Prompt 管理
model, messages, _, = await litellm_logging_obj.async_get_chat_completion_prompt(
    model=model,
    messages=messages,
    non_default_params=kwargs,
    prompt_variables=prompt_variables,
    prompt_id=prompt_id,
    prompt_spec=prompt_spec,
    prompt_label=prompt_label,
    prompt_version=prompt_version,
)
```

**Router 中的调用** (`litellm/router.py:2960-2970`):
```python
# 应用 Prompt 管理
model, messages, optional_params = litellm_logging_object.get_chat_completion_prompt(
    model=litellm_model,
    messages=messages,
    non_default_params=get_non_default_completion_params(kwargs=kwargs),
    prompt_variables=getattr(self.model_list[deploy], "prompt_variables", None)
    or prompt_variables,
    prompt_id=getattr(self.model_list[deploy], "prompt_id", None) or prompt_id,
    ...
)
```

**Proxy 中的调用** (`litellm/proxy/utils.py:1166-1180`):
```python
# 应用 Prompt 管理
model, messages, optional_params = await litellm_logging_obj.async_get_chat_completion_prompt(
    model=data.get("model", ""),
    messages=data.get("messages", []),
    non_default_params=get_non_default_completion_params(kwargs=data) or {},
    prompt_variables=data.get("prompt_variables", None),
    prompt_id=data.get("prompt_id", None),
    prompt_spec=data.get("prompt_spec", None),
    prompt_label=data.get("prompt_label", None),
    prompt_version=data.get("prompt_version", None),
)
```

### 4.3 Logging 类中的处理

```python
# 位于: litellm/litellm_core_utils/litellm_logging.py

class Logging(AdditionalLoggingUtils, LLMCachingHandler):
    def get_chat_completion_prompt(
        self, model: str, messages: List[AllMessageValues],
        non_default_params: Dict, prompt_variables: Optional[dict],
        prompt_id: Optional[str] = None, prompt_spec: Optional[PromptSpec] = None,
        prompt_management_logger: Optional[CustomLogger] = None,
        prompt_label: Optional[str] = None, prompt_version: Optional[int] = None,
    ) -> Tuple[str, List[AllMessageValues], dict]:
        # 1. 查找或获取 Prompt Manager
        custom_logger = (
            prompt_management_logger
            or self.get_custom_logger_for_prompt_management(
                model=model,
                non_default_params=non_default_params,
                prompt_id=prompt_id,
                prompt_spec=prompt_spec,
                dynamic_callback_params=self.standard_callback_dynamic_params,
            )
        )
        
        # 2. 如果找到，调用其 get_chat_completion_prompt
        if custom_logger:
            model, messages, non_default_params = custom_logger.get_chat_completion_prompt(
                model=model,
                messages=messages,
                non_default_params=non_default_params or {},
                prompt_id=prompt_id,
                prompt_spec=prompt_spec,
                prompt_variables=prompt_variables,
                dynamic_callback_params=self.standard_callback_dynamic_params,
                prompt_label=prompt_label,
                prompt_version=prompt_version,
            )
        
        self.messages = messages
        return model, messages, non_default_params
    
    async def async_get_chat_completion_prompt(
        self, ...
    ) -> Tuple[str, List[AllMessageValues], dict]:
        # 异步版本，逻辑相同但使用 async/await
        custom_logger = ...
        
        if custom_logger:
            model, messages, non_default_params = await custom_logger.async_get_chat_completion_prompt(
                ...
            )
        
        self.messages = messages
        return model, messages, non_default_params
```

### 4.4 Prompt Manager 自动发现

```python
# 位于: litellm/litellm_core_utils/litellm_logging.py

def get_custom_logger_for_prompt_management(
    self, model: str, non_default_params: Dict,
    tools: Optional[List[Dict]] = None,
    prompt_id: Optional[str] = None,
    prompt_spec: Optional[PromptSpec] = None,
    dynamic_callback_params: Optional[StandardCallbackDynamicParams] = None,
) -> Optional[CustomLogger]:
    """
    获取 Prompt Manager，有三个优先级:
    1. 模型前缀匹配 (如 "dotprompt/gpt-4")
    2. 自动检测 (根据 prompt_id)
    3. 回退到第一个注册的 CustomPromptManagement
    """
    
    # 1. 检查模型前缀 (如 "dotprompt/gpt-4")
    for callback_name in litellm._known_custom_logger_compatible_callbacks:
        if model.startswith(callback_name):
            custom_logger = _init_custom_logger_compatible_class(
                logging_integration=callback_name,
                ...
            )
            if custom_logger is not None:
                self.model_call_details["prompt_integration"] = model.split("/")[0]
                return custom_logger
    
    # 2. 自动检测 (根据 prompt_id)
    if prompt_id and dynamic_callback_params is not None:
        auto_detected_logger = self._auto_detect_prompt_management_logger(
            prompt_id=prompt_id,
            prompt_spec=prompt_spec,
            dynamic_callback_params=dynamic_callback_params,
        )
        if auto_detected_logger is not None:
            return auto_detected_logger
    
    # 3. 回退到第一个注册的 CustomPromptManagement
    prompt_management_loggers = (
        litellm.logging_callback_manager.get_custom_loggers_for_type(
            callback_type=CustomPromptManagement
        )
    )
    if prompt_management_loggers:
        logger = prompt_management_loggers[0]
        self.model_call_details["prompt_integration"] = logger.__class__.__name__
        return logger
    
    # 4. 其他检查 (Anthropic Cache Control, Vector Store 等)
    if anthropic_cache_control_logger := AnthropicCacheControlHook.get_custom_logger_for_anthropic_cache_control_hook(
        non_default_params
    ):
        self.model_call_details["prompt_integration"] = (
            anthropic_cache_control_logger.__class__.__name__
        )
        return anthropic_cache_control_logger
    
    return None

def _auto_detect_prompt_management_logger(
    self, prompt_id: str, prompt_spec: Optional[PromptSpec],
    dynamic_callback_params: StandardCallbackDynamicParams,
) -> Optional[CustomLogger]:
    """
    自动检测哪个 Prompt 管理系统拥有给定的 prompt_id
    
    遍历所有注册的 CustomPromptManagement 实例，调用 should_run_prompt_management
    检查哪个实例可以处理这个 prompt_id
    """
    prompt_management_loggers = (
        litellm.logging_callback_manager.get_custom_loggers_for_type(
            callback_type=CustomPromptManagement
        )
    )
    
    for logger in prompt_management_loggers:
        if isinstance(logger, CustomPromptManagement):
            try:
                if logger.should_run_prompt_management(
                    prompt_id=prompt_id,
                    prompt_spec=prompt_spec,
                    dynamic_callback_params=dynamic_callback_params,
                ):
                    self.model_call_details["prompt_integration"] = (
                        logger.__class__.__name__
                    )
                    return logger
            except Exception:
                # 如果检查失败，继续下一个 logger
                continue
    
    return None
```

### 4.5 PromptManagementBase 基类处理

```python
# 位于: litellm/integrations/prompt_management_base.py

class PromptManagementBase(ABC):
    def get_chat_completion_prompt(
        self, model: str, messages: List[AllMessageValues],
        non_default_params: dict, prompt_id: Optional[str],
        prompt_variables: Optional[dict], ...
    ) -> Tuple[str, List[AllMessageValues], dict]:
        if prompt_id is None:
            raise ValueError("prompt_id is required")
        
        # 1. 检查是否应该运行
        if not self.should_run_prompt_management(
            prompt_id=prompt_id, prompt_spec=prompt_spec,
            dynamic_callback_params=dynamic_callback_params,
        ):
            return model, messages, non_default_params
        
        # 2. 编译 Prompt
        prompt_template = self.compile_prompt(
            prompt_id=prompt_id,
            prompt_variables=prompt_variables,
            client_messages=messages,
            dynamic_callback_params=dynamic_callback_params,
            prompt_label=prompt_label,
            prompt_version=prompt_version,
            prompt_spec=prompt_spec,
        )
        
        # 3. 后处理
        return self.post_compile_prompt_processing(
            prompt_template=prompt_template,
            messages=messages,
            non_default_params=non_default_params,
            model=model,
            ignore_prompt_manager_model=ignore_prompt_manager_model,
            ignore_prompt_manager_optional_params=ignore_prompt_manager_optional_params,
        )
    
    def compile_prompt(
        self, prompt_id: str, prompt_variables: Optional[dict],
        client_messages: List[AllMessageValues], ...
    ) -> PromptManagementClient:
        # 1. 调用具体实现的编译
        compiled_prompt_client = self._compile_prompt_helper(
            prompt_id=prompt_id,
            prompt_spec=prompt_spec,
            prompt_variables=prompt_variables,
            dynamic_callback_params=dynamic_callback_params,
            prompt_label=prompt_label,
            prompt_version=prompt_version,
        )
        
        # 2. 合并模板消息和用户消息
        try:
            messages = compiled_prompt_client["prompt_template"] + client_messages
        except Exception as e:
            raise ValueError(f"Error compiling prompt: {e}. Prompt id={prompt_id}")
        
        compiled_prompt_client["completed_messages"] = messages
        return compiled_prompt_client
    
    def post_compile_prompt_processing(
        self, prompt_template: PromptManagementClient,
        messages: List[AllMessageValues], non_default_params: dict,
        model: str, ...
    ):
        """
        后处理:
        1. 合并消息
        2. 更新模型 (从 prompt_metadata 或保持原样)
        3. 更新参数 (temperature, max_tokens 等)
        """
        completed_messages = prompt_template["completed_messages"] or messages
        
        prompt_template_optional_params = (
            prompt_template["prompt_template_optional_params"] or {}
        )
        
        # 合并参数
        updated_non_default_params = {
            **non_default_params,
            **(
                prompt_template_optional_params
                if not ignore_prompt_manager_optional_params
                else {}
            ),
        }
        
        # 更新模型
        if not ignore_prompt_manager_model:
            model = self._get_model_from_prompt(
                prompt_management_client=prompt_template, model=model
            )
        
        return model, completed_messages, updated_non_default_params
    
    def _get_model_from_prompt(
        self, prompt_management_client: PromptManagementClient, model: str
    ) -> str:
        """
        从 Prompt 元数据获取模型，或从模型名中去除集成前缀
        
        例如:
        - model = "dotprompt/gpt-4" → 变成 "gpt-4"
        - 或者 prompt_template_model = "gpt-4o" → 使用这个值
        """
        if prompt_management_client["prompt_template_model"] is not None:
            return prompt_management_client["prompt_template_model"]
        else:
            return model.replace("{}/".format(self.integration_name), "")
```

### 4.6 消息合并逻辑

```python
def merge_messages(
    self, prompt_template: List[AllMessageValues],
    client_messages: List[AllMessageValues],
) -> List[AllMessageValues]:
    """
    合并 Prompt 模板消息和用户消息
    默认行为: 模板消息在前，用户消息在后
    """
    return prompt_template + client_messages
```

**消息合并示例**:
```
模板消息 (从 Prompt 文件解析):
[
    {"role": "system", "content": "你是一个专业的助手"},
    {"role": "user", "content": "请遵循以下规则: ..."}
]

用户消息:
[
    {"role": "user", "content": "你好，请帮我解决这个问题"}
]

合并后:
[
    {"role": "system", "content": "你是一个专业的助手"},
    {"role": "user", "content": "请遵循以下规则: ..."},
    {"role": "user", "content": "你好，请帮我解决这个问题"}
]
```

---

## 5. 缓存、刷新和版本切换的协作实现

### 5.1 缓存策略概览

| 实现类 | 缓存位置 | 缓存键 | 失效策略 |
|-------|---------|-------|---------|
| `GenericPromptManager` | `_prompt_cache` 字典 | `{prompt_id}:{prompt_label}:{prompt_version}` | `clear_cache()` 手动清除 |
| `GitLabPromptCache` | `_by_file`, `_by_id` 字典 | 文件路径 / Prompt ID | `reload()` 手动重载 |
| `DotpromptManager` | `prompts` 字典 | Prompt ID | `reload_prompts()` 重新扫描目录 |

### 5.2 GenericPromptManager 缓存实现

```python
# 位于: litellm/integrations/generic_prompt_management/generic_prompt_manager.py

class GenericPromptManager(CustomPromptManagement):
    def __init__(self, ...):
        self._prompt_cache: Dict[str, PromptManagementClient] = {}  # 内存缓存
    
    def _get_cache_key(
        self, prompt_id: Optional[str],
        prompt_label: Optional[str] = None,
        prompt_version: Optional[int] = None,
    ) -> str:
        """
        生成缓存键
        格式: {prompt_id}:{prompt_label}:{prompt_version}
        """
        return f"{prompt_id}:{prompt_label}:{prompt_version}"
    
    def _common_caching_logic(
        self, prompt_id: Optional[str],
        prompt_label: Optional[str] = None,
        prompt_version: Optional[int] = None,
        prompt_variables: Optional[dict] = None,
    ) -> Optional[PromptManagementClient]:
        """
        统一的缓存检查逻辑
        如果缓存命中且有变量，需要重新应用变量
        """
        cache_key = self._get_cache_key(prompt_id, prompt_label, prompt_version)
        if cache_key in self._prompt_cache:
            cached_prompt = self._prompt_cache[cache_key]
            # 如果有变量，需要重新应用
            if prompt_variables:
                return self._apply_variables(cached_prompt, prompt_variables)
            return cached_prompt
        return None
    
    def clear_cache(self) -> None:
        """清除所有缓存"""
        self._prompt_cache.clear()
```

### 5.3 GitLabPromptCache 专用缓存类

```python
# 位于: litellm/integrations/gitlab/gitlab_prompt_manager.py

class GitLabPromptCache:
    """
    专门用于缓存 GitLab 仓库中的所有 .prompt 文件
    
    支持:
    - 按文件路径索引: _by_file
    - 按 Prompt ID 索引: _by_id
    - 批量加载: load_all()
    - 刷新: reload()
    """
    
    def __init__(
        self, gitlab_config: Dict[str, Any],
        *, ref: Optional[str] = None,
        gitlab_client: Optional[GitLabClient] = None,
    ):
        # 构建底层的 PromptManager
        self.prompt_manager = GitLabPromptManager(
            gitlab_config=gitlab_config,
            prompt_id=None,
            ref=ref,
            gitlab_client=gitlab_client,
        )
        self.template_manager: GitLabTemplateManager = (
            self.prompt_manager.prompt_manager
        )
        
        # 内存存储
        self._by_file: Dict[str, Dict[str, Any]] = {}  # 文件路径 → JSON
        self._by_id: Dict[str, Dict[str, Any]] = {}    # Prompt ID → JSON
    
    def load_all(self, *, recursive: bool = True) -> Dict[str, Dict[str, Any]]:
        """
        扫描 GitLab 中 prompts_path 下的所有 .prompt 文件，
        加载并解析到内存中
        """
        # 1. 列出所有可用的 Prompt ID
        ids = self.template_manager.list_templates(recursive=recursive)
        
        for pid in ids:
            # 2. 确保模板已加载
            if pid not in self.template_manager.prompts:
                self.template_manager._load_prompt_from_gitlab(pid)
            
            tmpl = self.template_manager.get_template(pid)
            if tmpl is None:
                continue
            
            # 3. 转换为 JSON 格式并存储
            file_path = self.template_manager._id_to_repo_path(pid)
            entry = self._template_to_json(pid, tmpl)
            
            self._by_file[file_path] = entry
            encoded_id = encode_prompt_id(pid)  # "gitlab::greet/hi"
            self._by_id[encoded_id] = entry
        
        return self._by_id
    
    def reload(self, *, recursive: bool = True) -> Dict[str, Dict[str, Any]]:
        """清除缓存并重新加载"""
        self._by_file.clear()
        self._by_id.clear()
        return self.load_all(recursive=recursive)
    
    def list_files(self) -> List[str]:
        """返回缓存的所有文件路径"""
        return list(self._by_file.keys())
    
    def list_ids(self) -> List[str]:
        """返回缓存的所有 Prompt ID"""
        return list(self._by_id.keys())
    
    def get_by_file(self, file_path: str) -> Optional[Dict[str, Any]]:
        """按文件路径获取缓存的 Prompt"""
        return self._by_file.get(file_path)
    
    def get_by_id(self, prompt_id: str) -> Optional[Dict[str, Any]]:
        """
        按 Prompt ID 获取缓存的 Prompt
        支持多种 ID 格式:
        - "gitlab::greet/hi" (编码格式)
        - "greet/hi" (原始格式)
        """
        if prompt_id in self._by_id:
            return self._by_id.get(prompt_id)
        
        # 尝试规范化格式
        decoded = decode_prompt_id(prompt_id)
        encoded = encode_prompt_id(decoded)
        
        return self._by_id.get(encoded) or self._by_id.get(decoded)
    
    def _template_to_json(
        self, prompt_id: str, tmpl: GitLabPromptTemplate
    ) -> Dict[str, Any]:
        """
        将 GitLabPromptTemplate 转换为可序列化的 JSON 格式
        
        输出格式:
        {
            "id": "greet/hi",
            "path": "prompts/chat/greet/hi.prompt",
            "content": "System: 你好...",
            "metadata": {"model": "gpt-4", ...},
            "model": "gpt-4",
            "temperature": 0.7,
            "max_tokens": 1024,
            "optional_params": {...}
        }
        """
        md = dict(tmpl.metadata or {})
        
        return {
            "id": prompt_id,
            "path": self.template_manager._id_to_repo_path(prompt_id),
            "content": tmpl.content,
            "metadata": md,
            "model": tmpl.model,
            "temperature": tmpl.temperature,
            "max_tokens": tmpl.max_tokens,
            "optional_params": dict(tmpl.optional_params or {}),
        }
```

### 5.4 DotpromptManager 刷新机制

```python
# 位于: litellm/integrations/dotprompt/prompt_manager.py

class PromptManager:
    def __init__(
        self, prompt_id: Optional[str] = None,
        prompt_directory: Optional[str] = None,
        prompt_data: Optional[Dict[str, Dict[str, Any]]] = None,
        prompt_file: Optional[str] = None,
    ):
        self.prompt_directory = Path(prompt_directory) if prompt_directory else None
        self.prompts: Dict[str, PromptTemplate] = {}  # 内存存储
        ...
        
        if self.prompt_directory:
            self._load_prompts()
    
    def _load_prompts(self) -> None:
        """从目录加载所有 .prompt 文件"""
        if not self.prompt_directory or not self.prompt_directory.exists():
            raise ValueError(
                f"Prompt directory does not exist: {self.prompt_directory}"
            )
        
        prompt_files = list(self.prompt_directory.glob("*.prompt"))
        
        for prompt_file in prompt_files:
            try:
                prompt_id = prompt_file.stem  # 文件名（不含扩展名）
                template = self._load_prompt_file(prompt_file, prompt_id)
                self.prompts[prompt_id] = template
            except Exception:
                pass
    
    def reload_prompts(self) -> None:
        """重新加载所有 Prompt（清除缓存后重新加载）"""
        self.prompts.clear()
        if self.prompt_directory:
            self._load_prompts()
```

### 5.5 版本切换机制

#### 5.5.1 版本参数传递

**在请求中指定版本**:
```python
import litellm

# 方式1: prompt_version 参数
response = litellm.completion(
    model="gpt-4",
    prompt_id="my_prompt",
    prompt_version=2,  # 使用 v2 版本
    prompt_variables={"name": "World"},
    messages=[{"role": "user", "content": "..."}]
)

# 方式2: 版本化的 prompt_id (仅适用于数据库存储的 prompt)
# prompt_id 格式: "jack_success.v2"
```

#### 5.5.2 GitLab 的 Ref 机制

GitLab 支持灵活的版本引用：

```python
# 方式1: 使用 tag
gitlab_config = {
    "project": "myteam/prompts",
    "access_token": "glpat_...",
    "tag": "v1.2.3",  # 使用特定 tag
}

# 方式2: 使用 branch
gitlab_config = {
    "project": "myteam/prompts",
    "access_token": "glpat_...",
    "branch": "develop",  # 使用特定分支
}

# 方式3: 每个 Prompt 独立的 ref
gitlab_config = {
    "project": "myteam/prompts",
    "access_token": "glpat_...",
}

# 在请求时指定
response = litellm.completion(
    model="gitlab/gpt-4",
    prompt_id="greet/hi",
    git_ref="feature/new-prompt",  # per-call ref 覆盖
    prompt_variables={"name": "World"},
    ...
)
```

#### 5.5.3 代理数据库中的版本管理

```python
# 位于: litellm/proxy/prompts/prompt_endpoints.py

def get_version_number(prompt_id: str) -> int:
    """
    从版本化的 prompt_id 提取版本号
    
    示例:
    "jack_success.v2" → 2
    "jack_success_v2" → 2
    "jack_success" → 1 (默认)
    """
    # 尝试点分隔符 (.v)
    if ".v" in prompt_id:
        version_str = prompt_id.split(".v")[1]
        try:
            return int(version_str)
        except ValueError:
            pass
    
    # 尝试下划线分隔符 (_v)
    if "_v" in prompt_id:
        version_str = prompt_id.split("_v")[1]
        try:
            return int(version_str)
        except ValueError:
            pass
    
    return 1

def get_base_prompt_id(prompt_id: str) -> str:
    """
    提取基础 prompt_id（去除版本后缀）
    
    示例:
    "jack_success.v1" → "jack_success"
    "jack_success_v2" → "jack_success"
    "jack_success" → "jack_success"
    """
    if ".v" in prompt_id:
        return prompt_id.split(".v")[0]
    if "_v" in prompt_id:
        return prompt_id.split("_v")[0]
    return prompt_id

def get_latest_version_prompt_id(
    prompt_id: str, all_prompt_ids: Dict[str, Any]
) -> str:
    """
    从可用的 prompt_ids 中找到最新版本
    
    示例:
    all_ids = {"jack.v1": {}, "jack.v2": {}, "jack.v3": {}}
    → 返回 "jack.v3"
    """
    base_id = get_base_prompt_id(prompt_id=prompt_id)
    
    # 找出所有匹配的版本
    matching_versions = []
    for stored_prompt_id in all_prompt_ids.keys():
        if get_base_prompt_id(prompt_id=stored_prompt_id) == base_id:
            version_num = get_version_number(prompt_id=stored_prompt_id)
            matching_versions.append((version_num, stored_prompt_id))
    
    # 使用最高版本号
    if matching_versions:
        matching_versions.sort(reverse=True)
        return matching_versions[0][1]
    else:
        return prompt_id
```

### 5.6 环境管理

LiteLLM 还支持按环境隔离 Prompt：

```python
# 位于: litellm/types/prompts/init_prompts.py

class PromptInfo(BaseModel):
    prompt_type: Literal["config", "db"]
    environment: Optional[str] = "development"  # 支持多环境

class PromptSpec(BaseModel):
    prompt_id: str
    litellm_params: PromptLiteLLMParams
    prompt_info: PromptInfo
    ...
    environment: Optional[str] = "development"  # 多环境支持
    version: Optional[int] = None  # 版本号
```

**多环境使用示例**:
```python
# 创建不同环境的 Prompt
# development 环境
dev_prompt = PromptSpec(
    prompt_id="my_prompt.v1",
    litellm_params=PromptLiteLLMParams(
        prompt_id="my_prompt",
        prompt_integration="dotprompt",
    ),
    prompt_info=PromptInfo(
        prompt_type="db",
        environment="development"
    ),
    version=1,
)

# production 环境
prod_prompt = PromptSpec(
    prompt_id="my_prompt.v2",
    litellm_params=PromptLiteLLMParams(
        prompt_id="my_prompt",
        prompt_integration="dotprompt",
    ),
    prompt_info=PromptInfo(
        prompt_type="db",
        environment="production"
    ),
    version=2,
)
```

---

## 6. 核心模块索引

### 6.1 核心架构模块

| 文件路径 | 功能描述 |
|---------|---------|
| `litellm/integrations/custom_prompt_management.py` | `CustomPromptManagement` 基类定义 |
| `litellm/integrations/prompt_management_base.py` | `PromptManagementBase` 抽象基类，定义核心接口 |
| `litellm/proxy/prompts/prompt_registry.py` | Prompt 注册表和初始化器发现 |
| `litellm/proxy/prompts/prompt_endpoints.py` | Prompt 管理 API 端点 (CRUD, 版本管理) |
| `litellm/litellm_core_utils/litellm_logging.py` | `Logging` 类，包含 prompt 注入入口点 |
| `litellm/litellm_core_utils/logging_callback_manager.py` | 回调管理器，管理 prompt 管理实例 |

### 6.2 平台集成模块

| 平台 | 文件路径 | 功能描述 |
|-----|---------|---------|
| **Dotprompt** | `litellm/integrations/dotprompt/prompt_manager.py` | 本地 `.prompt` 文件管理器 |
| | `litellm/integrations/dotprompt/dotprompt_manager.py` | LiteLLM 集成的 Dotprompt 管理器 |
| **GitLab** | `litellm/integrations/gitlab/gitlab_client.py` | GitLab API 客户端 |
| | `litellm/integrations/gitlab/gitlab_prompt_manager.py` | GitLab Prompt 管理器 + 缓存 |
| **BitBucket** | `litellm/integrations/bitbucket/bitbucket_client.py` | BitBucket API 客户端 |
| | `litellm/integrations/bitbucket/bitbucket_prompt_manager.py` | BitBucket Prompt 管理器 |
| **Generic API** | `litellm/integrations/generic_prompt_management/generic_prompt_manager.py` | 通用 API 集成 |
| **Langfuse** | `litellm/integrations/langfuse/langfuse_prompt_management.py` | Langfuse 平台集成 |
| **Arize Phoenix** | `litellm/integrations/arize/arize_phoenix_prompt_manager.py` | Arize Phoenix 平台集成 |

### 6.3 类型定义模块

| 文件路径 | 功能描述 |
|---------|---------|
| `litellm/types/prompts/init_prompts.py` | Prompt 相关类型定义 (`PromptSpec`, `PromptLiteLLMParams`, `SupportedPromptIntegrations`) |
| `litellm/types/proxy/prompt_endpoints.py` | Prompt API 端点类型 |

---

## 7. 架构图

### 7.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户应用层                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  litellm.completion(                                                     ││
│  │      model="gpt-4",                                                      ││
│  │      prompt_id="my_prompt",                                              ││
│  │      prompt_version=2,                                                    ││
│  │      prompt_variables={"role": "assistant"},                            ││
│  │      messages=[{"role": "user", "content": "你好"}]                     ││
│  │  )                                                                        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           请求处理层                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  main.py / responses/main.py / router.py / proxy/utils.py              ││
│  │  ┌───────────────────────────────────────────────────────────────────┐ ││
│  │  │  litellm_logging_obj.get_chat_completion_prompt()                │ ││
│  │  │  (或 async 版本)                                                    │ ││
│  │  └───────────────────────────────────────────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                    │                                          │
│                                    ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  Logging.get_custom_logger_for_prompt_management()                      ││
│  │  ┌───────────────────────────────────────────────────────────────────┐ ││
│  │  │  1. 检查模型前缀 (如 "dotprompt/gpt-4")                            │ ││
│  │  │  2. 自动检测 (遍历 CustomPromptManagement 实例)                     │ ││
│  │  │  3. 回退到第一个注册的 Prompt Manager                                │ ││
│  │  └───────────────────────────────────────────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           Prompt Manager 层                                   │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │  CustomPromptManagement.get_chat_completion_prompt()                   ││
│  │  (继承自 PromptManagementBase)                                           ││
│  │  ┌───────────────────────────────────────────────────────────────────┐ ││
│  │  │  1. should_run_prompt_management() - 检查是否应该运行              │ ││
│  │  │  2. compile_prompt() - 编译 Prompt 模板                            │ ││
│  │  │     └─ _compile_prompt_helper() - 平台特定实现                      │ ││
│  │  │  3. post_compile_prompt_processing() - 后处理                      │ ││
│  │  │     └─ 合并消息 + 更新模型 + 更新参数                                │ ││
│  │  └───────────────────────────────────────────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                           数据源层                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   GitLab     │  │  BitBucket   │  │ Generic API  │  │  Dotprompt   │  │
│  │   仓库       │  │   仓库       │  │  外部服务    │  │  本地文件    │  │
│  │              │  │              │  │              │  │              │  │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │  │ Client │  │  │  │ Client │  │  │  │ HTTP   │  │  │  │ 目录   │  │
│  │  │ API    │  │  │  │ API    │  │  │  │ 调用   │  │  │  │ 扫描   │  │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │
│  │      │       │  │      │       │  │      │       │  │      │       │  │
│  │      ▼       │  │      ▼       │  │      ▼       │  │      ▼       │  │
│  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │  │  ┌────────┐  │
│  │  │ 缓存   │  │  │  │ 缓存   │  │  │  │ 缓存   │  │  │  │ 缓存   │  │
│  │  │(内存)  │  │  │  │(内存)  │  │  │  │(内存)  │  │  │  │(内存)  │  │
│  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │  │  └────────┘  │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.2 Prompt 编译流程

```
用户请求
    │
    ▼
┌──────────────────────┐
│  1. 检查缓存          │
│  cache_key =         │
│  {id}:{label}:{ver}  │
└──────────┬───────────┘
           │
    ┌──────┴──────┐
    │             │
  命中          未命中
    │             │
    ▼             ▼
┌─────────┐  ┌──────────────────┐
│ 返回    │  │ 2. 从数据源加载   │
│ 缓存    │  │    - GitLab API  │
│         │  │    - BitBucket   │
│         │  │    - Generic API │
│         │  │    - 本地文件    │
└─────────┘  └────────┬─────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 3. 解析 .prompt  │
              │    - YAML frontmatter
              │    - 模板内容    │
              └────────┬─────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 4. 应用变量      │
              │    Jinja2 渲染   │
              │    或字符串替换  │
              └────────┬─────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 5. 转换为消息    │
              │    "System:" →   │
              │    {"role": ...} │
              └────────┬─────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 6. 合并用户消息  │
              │    模板消息 +    │
              │    用户消息      │
              └────────┬─────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 7. 更新模型/参数 │
              │    - model       │
              │    - temperature │
              │    - max_tokens  │
              └────────┬─────────┘
                      │
                      ▼
              ┌──────────────────┐
              │ 返回三元组       │
              │ (model,          │
              │  messages,       │
              │  params)         │
              └──────────────────┘
```

---

## 8. 使用示例

### 8.1 Dotprompt (本地文件)

```python
import litellm
from litellm.integrations.dotprompt.dotprompt_manager import DotpromptManager

# 方式1: 设置全局 prompt 目录
litellm.global_prompt_directory = "/path/to/prompts"

# 方式2: 显式注册
dotprompt_manager = DotpromptManager(
    prompt_directory="/path/to/prompts"
)
litellm.callbacks = [dotprompt_manager]

# 使用
response = litellm.completion(
    model="dotprompt/gpt-4",
    prompt_id="greeting",
    prompt_variables={"name": "World", "role": "assistant"},
    messages=[{"role": "user", "content": "你好"}]
)
```

### 8.2 GitLab 集成

```python
import litellm

# 配置 GitLab
gitlab_config = {
    "project": "myteam/prompts-repo",
    "access_token": "glpat_xxxxxxxx",
    "branch": "main",  # 或使用 "tag": "v1.2.3"
    "prompts_path": "prompts/chat",  # 可选：仓库中的子目录
}

# 通过配置文件或 API 注册 Prompt
# 或者使用注册表自动发现

# 使用
response = litellm.completion(
    model="gpt-4",
    prompt_id="greet/hi",  # 对应 prompts/chat/greet/hi.prompt
    prompt_variables={"name": "Alice"},
    prompt_version=2,  # 可选：指定版本
)
```

### 8.3 Generic API 集成

```python
import litellm

# 方式1: 通过配置
generic_config = {
    "api_base": "https://your-prompt-service.com",
    "api_key": "sk-xxx",
    "timeout": 30,
}

# 方式2: 使用模型前缀
response = litellm.completion(
    model="generic_prompt/gpt-4",
    prompt_id="my-custom-prompt",
    prompt_variables={"context": "..."},
)
```

---

## 9. 总结

### 9.1 核心设计亮点

1. **统一抽象**: `CustomPromptManagement` + `PromptManagementBase` 提供了高度一致的接口
2. **自动发现**: `prompt_initializer_registry` 实现了插件式的集成发现
3. **多版本支持**: 支持 `prompt_version` + Git ref (tag/branch/SHA) 双重版本控制
4. **灵活缓存**: 内存缓存 + 手动刷新 + 按版本键缓存
5. **透明注入**: 在 `litellm.completion()` 调用链中自动应用，用户无需感知

### 9.2 扩展性

- 新增平台只需:
  1. 继承 `CustomPromptManagement`
  2. 实现 `_compile_prompt_helper` 和 `async_compile_prompt_helper`
  3. 在 `__init__.py` 中注册 `prompt_initializer_registry`

### 9.3 数据流

```
用户代码: completion(prompt_id="my_prompt", prompt_variables={...})
                    │
                    ▼
         ┌─────────────────────┐
         │  1. 查找 Prompt     │
         │     Manager         │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  2. 检查/加载缓存   │
         │     或从远端拉取    │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  3. 解析 .prompt    │
         │     渲染变量        │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  4. 合并消息        │
         │     更新模型/参数   │
         └──────────┬──────────┘
                    │
                    ▼
         ┌─────────────────────┐
         │  5. 发送给实际 LLM  │
         │     平台            │
         └─────────────────────┘
```

---

**报告生成时间