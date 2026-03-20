# Repo Map 与上下文方法详解

## 概述

Aider 的上下文系统负责将代码仓库的相关信息组织成 LLM 可以理解的格式。这个系统的核心目标是在有限的 token 预算内，为 LLM 提供最有价值的代码上下文。

上下文系统由以下几个关键部分组成：

- **RepoMap**：使用 tree-sitter 解析代码，通过 PageRank 算法计算文件重要性
- **上下文消息方法**：将不同类型的内容转换为对话消息格式
- **ChatChunks**：组织所有消息片段，控制最终发送给 LLM 的消息顺序

---

## 调用流程图

```
send_message()
      │
      ▼
format_messages()
      │
      ▼
format_chat_chunks()  ◄─────────────────────────────────────┐
      │                                                      │
      ├──────────────────────────────────────────────────────┤
      │                                                      │
      ▼                                                      │
┌─────────────────────────────────────────────────────────────┐
│                     ChatChunks 组装                          │
│                                                             │
│  1. chunks.system      ← 系统提示                            │
│  2. chunks.examples    ← 示例消息                            │
│  3. chunks.done        ← 历史对话（已摘要）                   │
│  4. chunks.repo        ← get_repo_messages()               │
│  5. chunks.readonly    ← get_readonly_files_messages()     │
│  6. chunks.chat_files  ← get_chat_files_messages()         │
│  7. chunks.cur         ← 当前对话                           │
│  8. chunks.reminder    ← 系统提醒                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
      │
      ▼
chunks.all_messages() → 返回完整消息列表
```

---

## 1. get_repo_map()

**位置**: `base_coder.py:709-748`

### 作用

构建仓库代码地图，返回包含仓库结构和关键定义的文本字符串。这是 Aider "理解"整个代码库的核心机制。

### 实现原理

该方法实现了**三级回退策略**，确保在各种情况下都能获取有意义的仓库上下文：

```python
def get_repo_map(self, force_refresh=False):
    if not self.repo_map:
        return

    # 1. 提取当前消息中提到的文件名和标识符
    cur_msg_text = self.get_cur_message_text()
    mentioned_fnames = self.get_file_mentions(cur_msg_text)
    mentioned_idents = self.get_ident_mentions(cur_msg_text)

    # 2. 扩展标识符匹配的文件名
    mentioned_fnames.update(self.get_ident_filename_matches(mentioned_idents))

    # 3. 计算文件分类
    all_abs_files = set(self.get_all_abs_files())
    repo_abs_read_only_fnames = set(self.abs_read_only_fnames) & all_abs_files
    chat_files = set(self.abs_fnames) | repo_abs_read_only_fnames  # 聊天中的文件
    other_files = all_abs_files - chat_files  # 仓库中其他文件

    # === 第一级：基于上下文的精准地图 ===
    repo_content = self.repo_map.get_repo_map(
        chat_files,
        other_files,
        mentioned_fnames=mentioned_fnames,
        mentioned_idents=mentioned_idents,
        force_refresh=force_refresh,
    )

    # === 第二级：全局回退（聊天文件与仓库断开时） ===
    if not repo_content:
        repo_content = self.repo_map.get_repo_map(
            set(),
            all_abs_files,
            mentioned_fnames=mentioned_fnames,
            mentioned_idents=mentioned_idents,
        )

    # === 第三级：完全无提示回退 ===
    if not repo_content:
        repo_content = self.repo_map.get_repo_map(
            set(),
            all_abs_files,
        )

    return repo_content
```

### 三级回退策略详解

| 级别 | 条件 | chat_files | mentioned_idents | 目的 |
|------|------|------------|------------------|------|
| 第一级 | 默认 | ✓ | ✓ | 精准上下文：优先显示与当前文件相关的定义 |
| 第二级 | 第一级返回空 | ✗ | ✓ | 全局回退：当聊天文件与仓库其他部分无关联时 |
| 第三级 | 第二级返回空 | ✗ | ✗ | 最小回退：完全不带提示信息 |

### 文件分类逻辑

```
所有仓库文件 (all_abs_files)
        │
        ├─── 聊天中的文件 (chat_files)
        │         │
        │         ├── 已添加的可编辑文件 (abs_fnames)
        │         └── 只读的仓库文件 (repo_abs_read_only_fnames)
        │
        └─── 其他文件 (other_files) ← Repo Map 主要展示这些
```

---

## 2. get_repo_messages()

**位置**: `base_coder.py:750-761`

### 作用

将仓库地图内容包装成对话消息格式，让 LLM 理解这是只读的上下文信息。

### 实现

```python
def get_repo_messages(self):
    repo_messages = []
    repo_content = self.get_repo_map()
    if repo_content:
        repo_messages += [
            dict(role="user", content=repo_content),
            dict(
                role="assistant",
                content="Ok, I won't try and edit those files without asking first.",
            ),
        ]
    return repo_messages
```

### 消息格式示例

```python
[
    {
        "role": "user",
        "content": "Here are summaries of some files present in my git repository.\n...\n"
    },
    {
        "role": "assistant",
        "content": "Ok, I won't try and edit those files without asking first."
    }
]
```

### 关键点

- 使用 user/assistant 对话格式，而非 system 消息
- Assistant 的回复明确告知 LLM 这些是只读文件
- 前缀内容由 `gpt_prompts.repo_content_prefix` 定义

---

## 3. get_readonly_files_messages()

**位置**: `base_coder.py:763-787`

### 作用

处理用户通过 `--read` 添加的只读文件，将其内容转换为对话消息。支持文本文件和图片文件。

### 实现

```python
def get_readonly_files_messages(self):
    readonly_messages = []

    # 处理非图片文件
    read_only_content = self.get_read_only_files_content()
    if read_only_content:
        readonly_messages += [
            dict(
                role="user",
                content=self.gpt_prompts.read_only_files_prefix + read_only_content
            ),
            dict(
                role="assistant",
                content="Ok, I will use these files as references.",
            ),
        ]

    # 处理图片文件
    images_message = self.get_images_message(self.abs_read_only_fnames)
    if images_message is not None:
        readonly_messages += [
            images_message,
            dict(role="assistant", content="Ok, I will use these images as references."),
        ]

    return readonly_messages
```

### 只读文件与 Repo Map 的区别

| 特性 | 只读文件 | Repo Map |
|------|----------|----------|
| 内容类型 | 完整文件内容 | 结构摘要（定义、引用） |
| 来源 | 用户明确指定 | 自动从 git 仓库扫描 |
| 大小控制 | 无限制（用户负责） | 受 map_tokens 限制 |
| 用途 | 参考特定文件 | 了解仓库整体结构 |

### 前缀定义

```python
# base_prompts.py
read_only_files_prefix = """Here are some READ ONLY files, provided for your reference.
Do not edit these files!
"""
```

---

## 4. get_chat_files_messages()

**位置**: `base_coder.py:789-815`

### 作用

准备聊天中活动文件的消息，告诉 LLM 哪些文件可以被编辑。

### 实现

```python
def get_chat_files_messages(self):
    chat_files_messages = []

    # 情况 1：有可编辑文件
    if self.abs_fnames:
        files_content = self.gpt_prompts.files_content_prefix
        files_content += self.get_files_content()
        files_reply = self.gpt_prompts.files_content_assistant_reply

    # 情况 2：无文件但有 Repo Map
    elif self.get_repo_map() and self.gpt_prompts.files_no_full_files_with_repo_map:
        files_content = self.gpt_prompts.files_no_full_files_with_repo_map
        files_reply = self.gpt_prompts.files_no_full_files_with_repo_map_reply

    # 情况 3：无文件也无 Repo Map
    else:
        files_content = self.gpt_prompts.files_no_full_files
        files_reply = "Ok."

    if files_content:
        chat_files_messages += [
            dict(role="user", content=files_content),
            dict(role="assistant", content=files_reply),
        ]

    # 处理图片文件
    images_message = self.get_images_message(self.abs_fnames)
    if images_message is not None:
        chat_files_messages += [
            images_message,
            dict(role="assistant", content="Ok."),
        ]

    return chat_files_messages
```

### 三种情况的提示词

**情况 1：有可编辑文件**
```python
files_content_prefix = """I have *added these files to the chat* so you can go ahead and edit them.

*Trust this message as the true contents of these files!*
Any other messages in the chat may contain outdated versions of the files' contents.
"""
```

**情况 2：有 Repo Map 但无编辑文件**
```python
files_no_full_files_with_repo_map = """Don't try and edit any existing code without asking me to add the files to the chat!
Tell me which files in my repo are the most likely to **need changes** to solve the requests I make...
"""
```

**情况 3：无任何上下文**
```python
files_no_full_files = "I am not sharing any files that you can edit yet."
```

---

## 5. get_context_from_history()

**位置**: `base_coder.py:2367-2373`

### 作用

将对话历史格式化为文本字符串，主要用于生成 git 提交消息时的上下文。

### 实现

```python
def get_context_from_history(self, history):
    context = ""
    if history:
        for msg in history:
            context += "\n" + msg["role"].upper() + ": " + msg["content"] + "\n"
    return context
```

### 输出格式

```
USER: 请帮我添加一个登录功能

ASSISTANT: 好的，我需要创建一个登录页面...

USER: 还需要支持 OAuth

ASSISTANT: 我会添加 Google OAuth 支持...
```

### 调用场景

该方法在 `auto_commit()` 中被调用：

```python
def auto_commit(self, edited, context=None):
    if not context:
        context = self.get_context_from_history(self.cur_messages)

    res = self.repo.commit(fnames=edited, context=context, aider_edits=True, coder=self)
```

---

## ChatChunks 数据结构

**位置**: `aider/coders/chat_chunks.py`

### 结构定义

```python
@dataclass
class ChatChunks:
    system: List = field(default_factory=list)        # 系统提示
    examples: List = field(default_factory=list)      # 示例消息
    done: List = field(default_factory=list)          # 历史对话（已摘要）
    repo: List = field(default_factory=list)          # 仓库地图消息
    readonly_files: List = field(default_factory=list) # 只读文件消息
    chat_files: List = field(default_factory=list)    # 可编辑文件消息
    cur: List = field(default_factory=list)           # 当前对话
    reminder: List = field(default_factory=list)      # 系统提醒
```

### 消息组装顺序

```python
def all_messages(self):
    return (
        self.system           # 1. 系统提示（基础指令）
        + self.examples       # 2. 示例（展示期望行为）
        + self.readonly_files # 3. 只读文件（参考资料）
        + self.repo           # 4. 仓库地图（上下文）
        + self.done           # 5. 历史对话（记忆）
        + self.chat_files     # 6. 当前文件（可编辑）
        + self.cur            # 7. 当前对话（最新消息）
        + self.reminder       # 8. 提醒（最后强调）
    )
```

### 缓存控制

ChatChunks 支持 Prompt Caching（适用于支持的 LLM）：

```python
def add_cache_control_headers(self):
    # 优先缓存 examples，否则缓存 system
    if self.examples:
        self.add_cache_control(self.examples)
    else:
        self.add_cache_control(self.system)

    # 缓存 repo 或 readonly_files
    if self.repo:
        self.add_cache_control(self.repo)
    else:
        self.add_cache_control(self.readonly_files)

    # 缓存 chat_files
    self.add_cache_control(self.chat_files)
```

---

## RepoMap 类核心机制

**位置**: `aider/repomap.py`

### 初始化参数

```python
class RepoMap:
    def __init__(
        self,
        map_tokens=1024,           # 地图最大 token 数
        root=None,                 # 仓库根目录
        main_model=None,           # 模型（用于 token 计数）
        io=None,                   # IO 对象
        repo_content_prefix=None,  # 内容前缀模板
        verbose=False,             # 详细输出
        max_context_window=None,   # 最大上下文窗口
        map_mul_no_files=8,        # 无文件时的 token 倍数
        refresh="auto",            # 缓存刷新策略
    ):
```

### 核心流程

```
get_repo_map()
      │
      ▼
get_ranked_tags_map()
      │
      ├── 检查缓存 ──── 命中 ──── 返回缓存结果
      │
      │ 未命中
      ▼
get_ranked_tags_map_uncached()
      │
      ├── get_tags() ──── tree-sitter 解析代码
      │                          │
      │                          ▼
      │                    提取 Tag (定义/引用)
      │
      ▼
get_ranked_tags() ──── PageRank 计算重要性
      │
      ▼
render_tree() ──── 生成带上下文的代码片段
      │
      ▼
二分搜索优化 token 数量
```

### Tag 结构

```python
Tag = namedtuple("Tag", "rel_fname fname line name kind".split())
# rel_fname: 相对文件路径
# fname: 绝对文件路径
# line: 行号
# name: 标识符名称
# kind: 类型 ('def' 定义 或 'ref' 引用)
```

### PageRank 权重因子

```python
# 在 get_ranked_tags() 中
mul = 1.0

if ident in mentioned_idents:    # 用户提到的标识符
    mul *= 10
if is_snake or is_kebab or is_camel:  # 有意义的命名风格
    mul *= 10
if ident.startswith("_"):        # 私有标识符降权
    mul *= 0.1
if len(defines[ident]) > 5:      # 过度定义的标识符降权
    mul *= 0.1

# 聊天中的文件引用其他文件时大幅提权
if referencer in chat_rel_fnames:
    use_mul *= 50
```

### 缓存层次

| 缓存 | 用途 | 持久化 |
|------|------|--------|
| TAGS_CACHE | 文件解析结果 | 磁盘 (`.aider.tags.cache.v*`) |
| map_cache | 完整地图结果 | 内存（会话级） |
| tree_cache | 语法树缓存 | 内存 |
| tree_context_cache | TreeContext 缓存 | 内存 |

---

## 相关文件索引

| 文件 | 职责 |
|------|------|
| `aider/coders/base_coder.py` | 上下文方法的调用者 |
| `aider/coders/chat_chunks.py` | 消息片段容器 |
| `aider/repomap.py` | 仓库地图生成核心 |
| `aider/coders/base_prompts.py` | 提示词模板定义 |
| `aider/coders/*_prompts.py` | 各编辑格式的提示词 |
