# RepoMap 方法详解

本文档详细解析 `aider/repomap.py` 中每个方法的作用、实现原理和调用关系。

---

## 1. 概述与核心概念

### 1.1 Tag 命名元组

```python
Tag = namedtuple("Tag", "rel_fname fname line name kind".split())
```

Tag 是整个 RepoMap 系统的核心数据结构，用于表示代码中的一个标识符：

| 字段 | 类型 | 说明 |
|------|------|------|
| `rel_fname` | str | 相对于仓库根目录的文件路径 |
| `fname` | str | 文件的绝对路径 |
| `line` | int | 标识符所在行号（0-indexed） |
| `name` | str | 标识符名称（函数名、类名、变量名等） |
| `kind` | str | 类型：`'def'`（定义）或 `'ref'`（引用） |

**使用示例**：
```python
# 表示 main.py 第 10 行定义了一个名为 "process_data" 的函数
Tag(rel_fname="main.py", fname="/home/user/project/main.py", line=10, name="process_data", kind="def")

# 表示 utils.py 第 25 行引用了 "process_data"
Tag(rel_fname="utils.py", fname="/home/user/project/utils.py", line=25, name="process_data", kind="ref")
```

### 1.2 缓存机制

RepoMap 实现了多层缓存以提高性能：

```
┌─────────────────────────────────────────────────────────────────┐
│                        缓存层次结构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐     持久化到磁盘                            │
│  │   TAGS_CACHE    │     .aider.tags.cache.v{CACHE_VERSION}     │
│  │  (diskcache)    │     缓存文件解析结果                         │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐     内存缓存                                │
│  │    map_cache    │     缓存完整地图结果                         │
│  │     (dict)      │     key: (chat_files, other_files, tokens)  │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐     内存缓存                                │
│  │   tree_cache    │     缓存渲染后的代码片段                     │
│  │     (dict)      │                                            │
│  └────────┬────────┘                                            │
│           │                                                     │
│           ▼                                                     │
│  ┌─────────────────┐     内存缓存                                │
│  │tree_context_cache│    缓存 TreeContext 对象                   │
│  │     (dict)      │     避免重复解析同一文件                     │
│  └─────────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**缓存版本控制**：
```python
CACHE_VERSION = 3
if USING_TSL_PACK:
    CACHE_VERSION = 4  # tree-sitter-language-pack 使用不同版本
```

---

## 2. 初始化与配置

### 2.1 `__init__()` - 初始化参数详解

**位置**: `repomap.py:47-87`

```python
def __init__(
    self,
    map_tokens=1024,           # 地图最大 token 数
    root=None,                 # 仓库根目录
    main_model=None,           # 模型对象（用于 token 计数）
    io=None,                   # IO 对象（用于文件读取和输出）
    repo_content_prefix=None,  # 内容前缀模板
    verbose=False,             # 详细输出模式
    max_context_window=None,   # 最大上下文窗口大小
    map_mul_no_files=8,        # 无聊天文件时的 token 倍数
    refresh="auto",            # 缓存刷新策略
):
```

**参数详解**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `map_tokens` | 1024 | Repo Map 的 token 预算上限 |
| `root` | `os.getcwd()` | 仓库根目录，用于计算相对路径 |
| `main_model` | None | 模型对象，用于精确计算 token 数量 |
| `io` | None | InputOutput 对象，用于读取文件和输出信息 |
| `repo_content_prefix` | None | 地图内容的前缀模板，支持 `{other}` 占位符 |
| `verbose` | False | 是否输出详细调试信息 |
| `max_context_window` | None | LLM 的最大上下文窗口，用于动态调整地图大小 |
| `map_mul_no_files` | 8 | 当聊天中没有文件时，地图 token 数乘以此倍数 |
| `refresh` | "auto" | 缓存刷新策略：`auto`/`manual`/`always`/`files` |

**初始化流程**：
```python
def __init__(self, ...):
    self.io = io
    self.verbose = verbose
    self.refresh = refresh

    # 1. 设置根目录
    if not root:
        root = os.getcwd()
    self.root = root

    # 2. 加载持久化缓存
    self.load_tags_cache()
    self.cache_threshold = 0.95

    # 3. 存储 token 相关配置
    self.max_map_tokens = map_tokens
    self.map_mul_no_files = map_mul_no_files
    self.max_context_window = max_context_window

    # 4. 初始化内存缓存
    self.tree_cache = {}
    self.tree_context_cache = {}
    self.map_cache = {}
    self.map_processing_time = 0
    self.last_map = None
```

### 2.2 `load_tags_cache()` - 缓存加载

**位置**: `repomap.py:217-222`

```python
def load_tags_cache(self):
    path = Path(self.root) / self.TAGS_CACHE_DIR
    try:
        self.TAGS_CACHE = Cache(path)
    except SQLITE_ERRORS as e:
        self.tags_cache_error(e)
```

**实现原理**：

1. 在仓库根目录下创建 `.aider.tags.cache.v{VERSION}` 目录
2. 使用 `diskcache.Cache` 创建基于 SQLite 的持久化缓存
3. 如果创建失败（权限问题、磁盘空间等），调用 `tags_cache_error()` 回退到内存缓存

**缓存结构**：
```python
# 缓存键值结构
cache_key = fname  # 文件绝对路径作为键
cache_value = {
    "mtime": file_mtime,  # 文件最后修改时间
    "data": [Tag, ...]    # 解析出的标签列表
}
```

### 2.3 `save_tags_cache()` - 缓存保存

**位置**: `repomap.py:224-225`

```python
def save_tags_cache(self):
    pass
```

**说明**：这是一个空方法。`diskcache.Cache` 会自动同步数据，无需显式保存。保留此方法是为了未来可能的扩展。

### 2.4 `tags_cache_error()` - 缓存错误处理

**位置**: `repomap.py:177-215`

```python
def tags_cache_error(self, original_error=None):
    """Handle SQLite errors by trying to recreate cache, falling back to dict if needed"""

    if self.verbose and original_error:
        self.io.tool_warning(f"Tags cache error: {str(original_error)}")

    # 如果已经是 dict，无需处理
    if isinstance(getattr(self, "TAGS_CACHE", None), dict):
        return

    path = Path(self.root) / self.TAGS_CACHE_DIR

    # 尝试重建缓存
    try:
        if path.exists():
            shutil.rmtree(path)

        new_cache = Cache(path)

        # 测试缓存是否可用
        test_key = "test"
        new_cache[test_key] = "test"
        _ = new_cache[test_key]
        del new_cache[test_key]

        self.TAGS_CACHE = new_cache
        return

    except SQLITE_ERRORS as e:
        # 回退到内存缓存
        self.io.tool_warning(
            f"Unable to use tags cache at {path}, falling back to memory cache"
        )

    self.TAGS_CACHE = dict()
```

**错误处理策略**：

```
SQLite 错误
     │
     ▼
检查当前缓存类型 ──── 已经是 dict ──── 返回（无需处理）
     │
     │ 不是 dict
     ▼
尝试删除并重建缓存
     │
     ├──── 成功 ──── 使用新的 diskcache
     │
     └──── 失败 ──── 回退到 dict（纯内存缓存）
```

---

## 3. 核心流程方法

### 3.1 `get_repo_map()` - 入口方法

**位置**: `repomap.py:103-167`

这是 RepoMap 的主入口，被 `base_coder.py` 的 `get_repo_map()` 调用。

```python
def get_repo_map(
    self,
    chat_files,           # 聊天中的文件集合
    other_files,          # 仓库中其他文件
    mentioned_fnames=None, # 用户提到的文件名
    mentioned_idents=None, # 用户提到的标识符
    force_refresh=False,   # 是否强制刷新缓存
):
```

**核心逻辑**：

```python
def get_repo_map(self, chat_files, other_files, ...):
    # 1. 边界检查
    if self.max_map_tokens <= 0:
        return
    if not other_files:
        return

    # 2. 动态调整 token 预算
    max_map_tokens = self.max_map_tokens
    padding = 4096

    if max_map_tokens and self.max_context_window:
        target = min(
            int(max_map_tokens * self.map_mul_no_files),
            self.max_context_window - padding,
        )
    else:
        target = 0

    # 当没有聊天文件时，扩大地图范围
    if not chat_files and self.max_context_window and target > 0:
        max_map_tokens = target

    # 3. 生成地图
    try:
        files_listing = self.get_ranked_tags_map(
            chat_files, other_files, max_map_tokens,
            mentioned_fnames, mentioned_idents, force_refresh,
        )
    except RecursionError:
        # 仓库太大时禁用 Repo Map
        self.io.tool_error("Disabling repo map, git repo too large?")
        self.max_map_tokens = 0
        return

    # 4. 添加前缀并返回
    if self.repo_content_prefix:
        repo_content = self.repo_content_prefix.format(other=other)
    else:
        repo_content = ""

    repo_content += files_listing
    return repo_content
```

**动态 Token 调整逻辑**：

```
                    有聊天文件
                        │
          ┌─────────────┴─────────────┐
          │                           │
          ▼                           ▼
    使用默认 map_tokens        扩大地图范围
    (通常 1024 tokens)        map_tokens * map_mul_no_files
                               但不超过 max_context_window - 4096
```

### 3.2 `get_ranked_tags_map()` - 带缓存的地图生成

**位置**: `repomap.py:576-627`

```python
def get_ranked_tags_map(
    self,
    chat_fnames,
    other_fnames=None,
    max_map_tokens=None,
    mentioned_fnames=None,
    mentioned_idents=None,
    force_refresh=False,
):
```

**缓存策略**：

```python
def get_ranked_tags_map(self, ...):
    # 1. 构建缓存键
    cache_key = [
        tuple(sorted(chat_fnames)) if chat_fnames else None,
        tuple(sorted(other_fnames)) if other_fnames else None,
        max_map_tokens,
    ]

    if self.refresh == "auto":
        cache_key += [
            tuple(sorted(mentioned_fnames)) if mentioned_fnames else None,
            tuple(sorted(mentioned_idents)) if mentioned_idents else None,
        ]
    cache_key = tuple(cache_key)

    # 2. 根据刷新策略决定是否使用缓存
    use_cache = False
    if not force_refresh:
        if self.refresh == "manual" and self.last_map:
            return self.last_map

        if self.refresh == "always":
            use_cache = False
        elif self.refresh == "files":
            use_cache = True
        elif self.refresh == "auto":
            use_cache = self.map_processing_time > 1.0

        if use_cache and cache_key in self.map_cache:
            return self.map_cache[cache_key]

    # 3. 生成新地图
    start_time = time.time()
    result = self.get_ranked_tags_map_uncached(...)
    end_time = time.time()

    # 4. 更新缓存
    self.map_processing_time = end_time - start_time
    self.map_cache[cache_key] = result
    self.last_map = result

    return result
```

**刷新策略详解**：

| 策略 | 行为 |
|------|------|
| `auto` | 处理时间 > 1 秒时使用缓存，mentioned 信息参与缓存键 |
| `manual` | 只返回 last_map，不重新生成 |
| `always` | 永不使用缓存，总是重新生成 |
| `files` | 只基于文件列表缓存，忽略 mentioned 信息 |

### 3.3 `get_ranked_tags_map_uncached()` - 实际地图生成

**位置**: `repomap.py:629-706`

```python
def get_ranked_tags_map_uncached(
    self,
    chat_fnames,
    other_fnames=None,
    max_map_tokens=None,
    mentioned_fnames=None,
    mentioned_idents=None,
):
```

**核心算法：二分搜索优化 Token 数量**

```python
def get_ranked_tags_map_uncached(self, ...):
    # 1. 获取排序后的标签
    spin = Spinner(UPDATING_REPO_MAP_MESSAGE)
    ranked_tags = self.get_ranked_tags(
        chat_fnames, other_fnames, mentioned_fnames, mentioned_idents,
        progress=spin.step,
    )

    # 2. 添加特殊文件（配置文件等）
    other_rel_fnames = sorted(set(self.get_rel_fname(fname) for fname in other_fnames))
    special_fnames = filter_important_files(other_rel_fnames)
    ranked_tags = special_fnames + ranked_tags

    # 3. 二分搜索找到最优 token 数量
    num_tags = len(ranked_tags)
    lower_bound = 0
    upper_bound = num_tags
    best_tree = None
    best_tree_tokens = 0

    middle = min(int(max_map_tokens // 25), num_tags)

    while lower_bound <= upper_bound:
        # 生成当前中间值的树
        tree = self.to_tree(ranked_tags[:middle], chat_rel_fnames)
        num_tokens = self.token_count(tree)

        # 计算误差
        pct_err = abs(num_tokens - max_map_tokens) / max_map_tokens
        ok_err = 0.15  # 15% 误差可接受

        # 更新最佳结果
        if (num_tokens <= max_map_tokens and num_tokens > best_tree_tokens) or pct_err < ok_err:
            best_tree = tree
            best_tree_tokens = num_tokens

            if pct_err < ok_err:
                break  # 找到足够好的结果

        # 调整搜索范围
        if num_tokens < max_map_tokens:
            lower_bound = middle + 1
        else:
            upper_bound = middle - 1

        middle = int((lower_bound + upper_bound) // 2)

    spin.end()
    return best_tree
```

**二分搜索可视化**：

```
目标：找到最大的 middle，使得 token_count(ranked_tags[:middle]) <= max_map_tokens

初始状态：
lower_bound = 0, upper_bound = num_tags, middle = max_map_tokens // 25

迭代过程：
┌─────────────────────────────────────────────────────────────┐
│  lower_bound          middle          upper_bound           │
│       │                 │                 │                 │
│       ▼                 ▼                 ▼                 │
│  ─────┼─────────────────┼─────────────────┼─────────────    │
│       │                 │                 │                 │
│       │   tokens < max  │                 │                 │
│       │   → lower = mid + 1               │                 │
│       │                 │                 │                 │
│       │   tokens > max  │                 │                 │
│       │   → upper = mid - 1               │                 │
└─────────────────────────────────────────────────────────────┘

终止条件：lower_bound > upper_bound 或误差 < 15%
```

### 3.4 `get_ranked_tags()` - PageRank 排序核心

**位置**: `repomap.py:365-574`

这是 RepoMap 最核心的方法，使用 PageRank 算法计算文件和定义的重要性。

```python
def get_ranked_tags(
    self, chat_fnames, other_fnames, mentioned_fnames, mentioned_idents, progress=None
):
    import networkx as nx

    defines = defaultdict(set)      # 标识符 -> 定义它的文件集合
    references = defaultdict(list)  # 标识符 -> 引用它的文件列表
    definitions = defaultdict(set)  # (文件, 标识符) -> Tag 集合
    personalization = dict()        # PageRank 个性化向量
```

**步骤 1：收集定义和引用**

```python
fnames = set(chat_fnames).union(set(other_fnames))
personalize = 100 / len(fnames)  # 基础个性化分数

for fname in fnames:
    rel_fname = self.get_rel_fname(fname)
    current_pers = 0.0

    # 计算个性化分数
    if fname in chat_fnames:
        current_pers += personalize
        chat_rel_fnames.add(rel_fname)

    if rel_fname in mentioned_fnames:
        current_pers = max(current_pers, personalize)

    # 检查路径是否匹配提到的标识符
    path_components = set(Path(rel_fname).parts)
    matched_idents = path_components.intersection(mentioned_idents)
    if matched_idents:
        current_pers += personalize

    if current_pers > 0:
        personalization[rel_fname] = current_pers

    # 解析文件获取标签
    tags = list(self.get_tags(fname, rel_fname))
    for tag in tags:
        if tag.kind == "def":
            defines[tag.name].add(rel_fname)
            definitions[(rel_fname, tag.name)].add(tag)
        elif tag.kind == "ref":
            references[tag.name].append(rel_fname)
```

**步骤 2：构建引用图**

```python
G = nx.MultiDiGraph()

# 为没有引用的定义添加自环边
for ident in defines.keys():
    if ident not in references:
        for definer in defines[ident]:
            G.add_edge(definer, definer, weight=0.1, ident=ident)

# 为每个引用关系添加边
idents = set(defines.keys()).intersection(set(references.keys()))

for ident in idents:
    definers = defines[ident]
    mul = 1.0

    # === 权重因子计算 ===
    is_snake = ("_" in ident) and any(c.isalpha() for c in ident)
    is_kebab = ("-" in ident) and any(c.isalpha() for c in ident)
    is_camel = any(c.isupper() for c in ident) and any(c.islower() for c in ident)

    if ident in mentioned_idents:
        mul *= 10  # 用户提到的标识符提权

    if (is_snake or is_kebab or is_camel) and len(ident) >= 8:
        mul *= 10  # 有意义的命名风格提权

    if ident.startswith("_"):
        mul *= 0.1  # 私有标识符降权

    if len(defines[ident]) > 5:
        mul *= 0.1  # 过度定义的标识符降权

    # 添加边
    for referencer, num_refs in Counter(references[ident]).items():
        for definer in definers:
            use_mul = mul
            if referencer in chat_rel_fnames:
                use_mul *= 50  # 聊天文件引用大幅提权

            num_refs = math.sqrt(num_refs)  # 平滑高频引用

            G.add_edge(referencer, definer, weight=use_mul * num_refs, ident=ident)
```

**步骤 3：运行 PageRank**

```python
if personalization:
    pers_args = dict(personalization=personalization, dangling=personalization)
else:
    pers_args = dict()

try:
    ranked = nx.pagerank(G, weight="weight", **pers_args)
except ZeroDivisionError:
    ranked = nx.pagerank(G, weight="weight")
```

**步骤 4：分配排名分数到定义**

```python
ranked_definitions = defaultdict(float)

for src in G.nodes:
    src_rank = ranked[src]
    total_weight = sum(data["weight"] for _src, _dst, data in G.out_edges(src, data=True))

    for _src, dst, data in G.out_edges(src, data=True):
        # 按权重比例分配源节点的排名分数
        data["rank"] = src_rank * data["weight"] / total_weight
        ident = data["ident"]
        ranked_definitions[(dst, ident)] += data["rank"]
```

**步骤 5：生成排序后的标签列表**

```python
ranked_tags = []
ranked_definitions = sorted(
    ranked_definitions.items(), reverse=True, key=lambda x: (x[1], x[0])
)

for (fname, ident), rank in ranked_definitions:
    if fname in chat_rel_fnames:
        continue  # 跳过聊天中的文件
    ranked_tags += list(definitions.get((fname, ident), []))

# 添加没有标签的文件
for fname in top_rank:
    if fname not in fnames_already_included:
        ranked_tags.append((fname,))

return ranked_tags
```

**PageRank 权重因子汇总**：

| 条件 | 乘数 | 说明 |
|------|------|------|
| 用户提到的标识符 | ×10 | 提高相关性 |
| snake_case/kebab-case/camelCase 且长度 ≥ 8 | ×10 | 有意义的命名 |
| 以 `_` 开头 | ×0.1 | 私有标识符降权 |
| 定义次数 > 5 | ×0.1 | 通用名称降权 |
| 聊天文件中的引用 | ×50 | 当前上下文高度相关 |
| 引用次数 | √n | 平滑高频引用 |

---

## 4. 代码解析方法

### 4.1 `get_tags()` - 获取文件标签（带缓存）

**位置**: `repomap.py:233-264`

```python
def get_tags(self, fname, rel_fname):
    # 1. 获取文件修改时间
    file_mtime = self.get_mtime(fname)
    if file_mtime is None:
        return []

    # 2. 检查缓存
    cache_key = fname
    try:
        val = self.TAGS_CACHE.get(cache_key)
    except SQLITE_ERRORS as e:
        self.tags_cache_error(e)
        val = self.TAGS_CACHE.get(cache_key)

    # 3. 缓存命中且未修改
    if val is not None and val.get("mtime") == file_mtime:
        try:
            return self.TAGS_CACHE[cache_key]["data"]
        except SQLITE_ERRORS as e:
            self.tags_cache_error(e)
            return self.TAGS_CACHE[cache_key]["data"]

    # 4. 缓存未命中，解析文件
    data = list(self.get_tags_raw(fname, rel_fname))

    # 5. 更新缓存
    try:
        self.TAGS_CACHE[cache_key] = {"mtime": file_mtime, "data": data}
        self.save_tags_cache()
    except SQLITE_ERRORS as e:
        self.tags_cache_error(e)
        self.TAGS_CACHE[cache_key] = {"mtime": file_mtime, "data": data}

    return data
```

**缓存验证逻辑**：

```
get_tags(fname, rel_fname)
         │
         ▼
   获取文件 mtime
         │
         ▼
   检查缓存 ──── 缓存存在且 mtime 匹配 ──── 返回缓存数据
         │
         │ 缓存不存在或 mtime 不匹配
         ▼
   调用 get_tags_raw() 解析
         │
         ▼
   更新缓存
         │
         ▼
   返回解析结果
```

### 4.2 `get_tags_raw()` - 原始解析实现

**位置**: `repomap.py:279-363`

```python
def get_tags_raw(self, fname, rel_fname):
    # 1. 确定语言
    lang = filename_to_lang(fname)
    if not lang:
        return

    # 2. 获取解析器和语言对象
    try:
        language = get_language(lang)
        parser = get_parser(lang)
    except Exception as err:
        print(f"Skipping file {fname}: {err}")
        return

    # 3. 加载 tree-sitter 查询文件
    query_scm = get_scm_fname(lang)
    if not query_scm.exists():
        return
    query_scm = query_scm.read_text()

    # 4. 解析代码
    code = self.io.read_text(fname)
    if not code:
        return
    tree = parser.parse(bytes(code, "utf-8"))

    # 5. 执行查询
    captures = self._run_captures(Query(language, query_scm), tree.root_node)

    # 6. 处理捕获结果
    captures_by_tag = defaultdict(list)
    matches = []
    for tag, nodes in captures.items():
        for node in nodes:
            captures_by_tag[tag].append(node)
        matches.append((node, tag))

    # 7. 生成 Tag 对象
    saw = set()
    for node, tag in all_nodes:
        if tag.startswith("name.definition."):
            kind = "def"
        elif tag.startswith("name.reference."):
            kind = "ref"
        else:
            continue

        saw.add(kind)

        yield Tag(
            rel_fname=rel_fname,
            fname=fname,
            name=node.text.decode("utf-8"),
            kind=kind,
            line=node.start_point[0],
        )

    # 8. 如果只有定义没有引用，使用 Pygments 补充引用
    if "ref" not in saw and "def" in saw:
        lexer = guess_lexer_for_filename(fname, code)
        tokens = list(lexer.get_tokens(code))
        tokens = [token[1] for token in tokens if token[0] in Token.Name]

        for token in tokens:
            yield Tag(
                rel_fname=rel_fname,
                fname=fname,
                name=token,
                kind="ref",
                line=-1,
            )
```

**解析流程**：

```
get_tags_raw(fname, rel_fname)
         │
         ▼
┌─────────────────────────────────────┐
│  1. filename_to_lang(fname)         │
│     根据文件扩展名确定语言            │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  2. get_language(lang)              │
│     get_parser(lang)                │
│     获取 tree-sitter 解析器          │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  3. get_scm_fname(lang)             │
│     加载语言特定的查询文件            │
│     如 python-tags.scm              │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  4. parser.parse(code)              │
│     解析代码生成语法树                │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  5. Query(language, query_scm)      │
│     执行 tree-sitter 查询            │
│     提取定义和引用节点                │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  6. 生成 Tag 对象                    │
│     kind: "def" 或 "ref"            │
└────────────────┬────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────┐
│  7. 必要时用 Pygments 补充引用        │
│     某些语言的查询文件只定义了定义     │
└─────────────────────────────────────┘
```

### 4.3 `_run_captures()` - tree-sitter 查询执行

**位置**: `repomap.py:266-277`

```python
def _run_captures(self, query: Query, node):
    # tree-sitter 0.23.2 的 python bindings 有 captures 直接在 Query 对象上
    # 但 0.24.0 移到了单独的 QueryCursor 类。支持两者。
    if hasattr(query, "captures"):
        # 旧 API
        return query.captures(node)

    # 新 API
    from tree_sitter import QueryCursor

    cursor = QueryCursor(query)
    return cursor.captures(node)
```

**兼容性说明**：

| tree-sitter 版本 | API |
|------------------|-----|
| 0.23.2 | `query.captures(node)` |
| 0.24.0+ | `QueryCursor(query).captures(node)` |

---

## 5. 渲染方法

### 5.1 `to_tree()` - 将标签转换为树形输出

**位置**: `repomap.py:748-784`

```python
def to_tree(self, tags, chat_rel_fnames):
    if not tags:
        return ""

    cur_fname = None
    cur_abs_fname = None
    lois = None  # lines of interest
    output = ""

    # 添加一个空标签以触发最后一个文件的处理
    dummy_tag = (None,)
    for tag in sorted(tags) + [dummy_tag]:
        this_rel_fname = tag[0]

        # 跳过聊天中的文件
        if this_rel_fname in chat_rel_fnames:
            continue

        # 文件名变化时，输出前一个文件的内容
        if this_rel_fname != cur_fname:
            if lois is not None:
                # 有标签的文件：渲染代码片段
                output += "\n"
                output += cur_fname + ":\n"
                output += self.render_tree(cur_abs_fname, cur_fname, lois)
                lois = None
            elif cur_fname:
                # 无标签的文件：只输出文件名
                output += "\n" + cur_fname + "\n"

            if type(tag) is Tag:
                lois = []
                cur_abs_fname = tag.fname
            cur_fname = this_rel_fname

        if lois is not None:
            lois.append(tag.line)

    # 截断过长的行（防止压缩的 JS 等文件）
    output = "\n".join([line[:100] for line in output.splitlines()]) + "\n"

    return output
```

**输出格式示例**：

```
config.py:
│class Config:
│    def __init__(self):
│        self.settings = {}
│
utils.py:
│def process_data(data):
│    return data.strip()
│
main.py
```

### 5.2 `render_tree()` - 渲染单个文件的代码片段

**位置**: `repomap.py:710-746`

```python
def render_tree(self, abs_fname, rel_fname, lois):
    # 1. 获取修改时间用于缓存键
    mtime = self.get_mtime(abs_fname)
    key = (rel_fname, tuple(sorted(lois)), mtime)

    # 2. 检查缓存
    if key in self.tree_cache:
        return self.tree_cache[key]

    # 3. 获取或创建 TreeContext
    if (
        rel_fname not in self.tree_context_cache
        or self.tree_context_cache[rel_fname]["mtime"] != mtime
    ):
        code = self.io.read_text(abs_fname) or ""
        if not code.endswith("\n"):
            code += "\n"

        context = TreeContext(
            rel_fname,
            code,
            color=False,
            line_number=False,
            child_context=False,
            last_line=False,
            margin=0,
            mark_lois=False,
            loi_pad=0,
            show_top_of_file_parent_scope=False,
        )
        self.tree_context_cache[rel_fname] = {"context": context, "mtime": mtime}

    # 4. 设置感兴趣的行并渲染
    context = self.tree_context_cache[rel_fname]["context"]
    context.lines_of_interest = set()
    context.add_lines_of_interest(lois)
    context.add_context()  # 添加上下文行（如函数签名）
    res = context.format()

    # 5. 缓存结果
    self.tree_cache[key] = res
    return res
```

**TreeContext 的作用**：

`TreeContext` 来自 `grep_ast` 包，它能够：
1. 解析代码的语法结构
2. 根据感兴趣的行（lois）提取相关代码片段
3. 自动添加必要的上下文（如函数定义、类定义）

**示例**：

如果 `lois = [25]`（第 25 行是一个函数调用），`TreeContext` 会自动包含：
- 函数定义所在的行
- 相关的类定义
- 必要的导入语句

### 5.3 `token_count()` - Token 计数估算

**位置**: `repomap.py:89-101`

```python
def token_count(self, text):
    len_text = len(text)

    # 短文本直接计算
    if len_text < 200:
        return self.main_model.token_count(text)

    # 长文本使用采样估算
    lines = text.splitlines(keepends=True)
    num_lines = len(lines)
    step = num_lines // 100 or 1
    lines = lines[::step]  # 每 step 行取一行
    sample_text = "".join(lines)
    sample_tokens = self.main_model.token_count(sample_text)

    # 按比例估算总 token 数
    est_tokens = sample_tokens / len(sample_text) * len_text
    return est_tokens
```

**采样策略**：

```
原始文本（1000 行）
       │
       ▼
采样（每 10 行取 1 行）→ 100 行样本
       │
       ▼
计算样本 token 数
       │
       ▼
估算：sample_tokens / sample_length * total_length
```

---

## 6. 辅助方法

### 6.1 `get_rel_fname()` - 相对路径转换

**位置**: `repomap.py:169-175`

```python
def get_rel_fname(self, fname):
    try:
        return os.path.relpath(fname, self.root)
    except ValueError:
        # Windows 跨盘符问题
        # Issue #1288: ValueError: path is on mount 'C:', start on mount 'D:'
        return fname
```

### 6.2 `get_mtime()` - 获取修改时间

**位置**: `repomap.py:227-231`

```python
def get_mtime(self, fname):
    try:
        return os.path.getmtime(fname)
    except FileNotFoundError:
        self.io.tool_warning(f"File not found error: {fname}")
```

---

## 7. 工具函数

### 7.1 `get_scm_fname()` - 获取 tree-sitter 查询文件

**位置**: `repomap.py:805-829`

```python
def get_scm_fname(lang):
    # 1. 尝试 tree-sitter-language-pack 路径
    if USING_TSL_PACK:
        subdir = "tree-sitter-language-pack"
        try:
            path = resources.files(__package__).joinpath(
                "queries", subdir, f"{lang}-tags.scm",
            )
            if path.exists():
                return path
        except KeyError:
            pass

    # 2. 回退到 tree-sitter-languages 路径
    subdir = "tree-sitter-languages"
    try:
        return resources.files(__package__).joinpath(
            "queries", subdir, f"{lang}-tags.scm",
        )
    except KeyError:
        return
```

**查询文件路径结构**：

```
aider/
└── queries/
    ├── tree-sitter-languages/
    │   ├── python-tags.scm
    │   ├── javascript-tags.scm
    │   ├── rust-tags.scm
    │   └── ...
    └── tree-sitter-language-pack/
        ├── python-tags.scm
        └── ...
```

### 7.2 `get_supported_languages_md()` - 支持的语言列表

**位置**: `repomap.py:832-849`

```python
def get_supported_languages_md():
    from grep_ast.parsers import PARSERS

    res = """
| Language | File extension | Repo map | Linter |
|:--------:|:--------------:|:--------:|:------:|
"""
    data = sorted((lang, ex) for ex, lang in PARSERS.items())

    for lang, ext in data:
        fn = get_scm_fname(lang)
        repo_map = "✓" if Path(fn).exists() else ""
        linter_support = "✓"
        res += f"| {lang:20} | {ext:20} | {repo_map:^8} | {linter_support:^6} |\n"

    return res
```

### 7.3 `filter_important_files()` - 重要文件过滤

**位置**: `aider/special.py:196-203`

```python
ROOT_IMPORTANT_FILES = [
    # Version Control
    ".gitignore",
    # Documentation
    "README.md",
    # Package Management
    "requirements.txt",
    "pyproject.toml",
    "package.json",
    # Configuration
    "tsconfig.json",
    # CI/CD
    ".github/workflows/*.yml",
    # ... 更多配置文件
]

def filter_important_files(file_paths):
    return list(filter(is_important, file_paths))
```

---

## 8. 关键算法详解

### 8.1 PageRank 权重计算

**算法原理**：

PageRank 最初用于网页排名，Aider 将其应用于代码文件重要性排序。

**核心思想**：
1. 每个文件是一个节点
2. 文件 A 引用文件 B 中定义的标识符 → A 到 B 有一条有向边
3. 边的权重反映引用的重要性
4. PageRank 值高的文件更重要

**图构建示例**：

```
文件关系：
- main.py 调用 utils.py 中的 process_data()
- main.py 调用 config.py 中的 Config 类
- utils.py 调用 config.py 中的 get_config()

构建的图：
                    main.py
                   /        \
                  ↓          ↓
            utils.py ───→ config.py

边权重：
- main.py → utils.py: weight = 1.0 * 50 (聊天文件引用) * sqrt(1) = 50
- main.py → config.py: weight = 1.0 * 50 * sqrt(1) = 50
- utils.py → config.py: weight = 1.0 * sqrt(1) = 1
```

**个性化向量**：

```python
# 让聊天中的文件和提到的文件获得更高的初始分数
personalization = {
    "main.py": 10.0,    # 在聊天中
    "utils.py": 5.0,    # 被提到
}
```

### 8.2 二分搜索 Token 优化

**问题**：给定排序后的标签列表，找到最大数量的标签，使得渲染后的 token 数不超过预算。

**算法**：

```python
def binary_search_tokens(ranked_tags, max_tokens):
    lower = 0
    upper = len(ranked_tags)
    best = None
    best_tokens = 0

    while lower <= upper:
        middle = (lower + upper) // 2
        tree = render(ranked_tags[:middle])
        tokens = count_tokens(tree)

        if tokens <= max_tokens:
            if tokens > best_tokens:
                best = tree
                best_tokens = tokens
            lower = middle + 1  # 尝试更多标签
        else:
            upper = middle - 1  # 减少标签

    return best
```

**时间复杂度**：O(n * log(n))，其中 n 是标签数量

### 8.3 tree-sitter 查询语法

**查询文件格式**（以 Python 为例）：

```scheme
; 定义类
(class_definition
  name: (identifier) @name.definition.class) @definition.class

; 定义函数
(function_definition
  name: (identifier) @name.definition.function) @definition.function

; 函数调用
(call
  function: [
      (identifier) @name.reference.call
      (attribute
        attribute: (identifier) @name.reference.call)
  ]) @reference.call
```

**查询语法说明**：

| 语法 | 说明 |
|------|------|
| `(node_type)` | 匹配特定类型的节点 |
| `name: (pattern)` | 匹配名为 name 的子节点 |
| `@capture_name` | 捕获匹配的节点 |
| `[pattern1 pattern2]` | 匹配任一模式 |
| `;` | 注释 |

**JavaScript 查询示例**：

```scheme
; 方法定义
(method_definition
  name: (property_identifier) @name.definition.method) @definition.method

; 类定义
(class
  name: (_) @name.definition.class) @definition.class

; 函数调用
(call_expression
  function: (identifier) @name.reference.call) @reference.call
```

---

## 9. 调用关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           base_coder.py                                     │
│                                                                             │
│  get_repo_map() ────────────────────────────────────────────────────────────┤
│                                                                             │
└─────────────────────────────────────┬───────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              repomap.py                                     │
│                                                                             │
│  get_repo_map()                                                             │
│       │                                                                     │
│       ▼                                                                     │
│  get_ranked_tags_map() ◄───────────────────────────────────────────────────┐│
│       │                                                                    ││
│       ├──── 缓存命中 ──── 返回缓存结果                                       ││
│       │                                                                    ││
│       ▼                                                                    ││
│  get_ranked_tags_map_uncached()                                            ││
│       │                                                                    ││
│       ├──────────────────────────────┐                                     ││
│       │                              │                                     ││
│       ▼                              ▼                                     ││
│  get_ranked_tags()              filter_important_files()                   ││
│       │                              │                                     ││
│       ▼                              │                                     ││
│  get_tags() ◄───────────────────────┘                                     ││
│       │                          special.py                                 ││
│       ├──── 缓存命中 ──── 返回缓存                                          ││
│       │                                                                    ││
│       ▼                                                                    ││
│  get_tags_raw()                                                            ││
│       │                                                                    ││
│       ├──── get_scm_fname() ──── 加载查询文件                               ││
│       │                                                                    ││
│       └──── _run_captures() ──── 执行 tree-sitter 查询                      ││
│                                                                            ││
│       │                                                                    ││
│       ▼                                                                    ││
│  PageRank 排序 (networkx)                                                  ││
│       │                                                                    ││
│       ▼                                                                    ││
│  to_tree()                                                                 ││
│       │                                                                    ││
│       └──── render_tree() ──── TreeContext 渲染代码片段                     ││
│              │                                                             ││
│              └──── token_count() ──── Token 计数                           ││
│                                                                            ││
│       │                                                                    ││
│       ▼                                                                    ││
│  二分搜索优化 token 数量                                                    ││
│       │                                                                    ││
│       └────────────────────────────────────────────────────────────────────┘│
│                                                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 相关文件索引

| 文件 | 职责 |
|------|------|
| `aider/repomap.py` | RepoMap 核心实现 |
| `aider/special.py` | 重要文件识别 |
| `aider/waiting.py` | Spinner 加载动画 |
| `aider/queries/tree-sitter-languages/*.scm` | tree-sitter 查询文件 |
| `aider/queries/tree-sitter-language-pack/*.scm` | 备用查询文件 |
| `grep_ast/TreeContext` | 代码片段渲染（外部依赖） |
| `grep_ast/tsl` | tree-sitter 语言加载（外部依赖） |
