# Aider 中 Tree-sitter 技术实现详解

## 1. 概述

### 1.1 Tree-sitter 简介

Tree-sitter 是一个解析器生成工具和增量解析库，具有以下特点：

- **增量解析**：当代码修改时，只重新解析受影响的部分，性能极高
- **错误容忍**：即使代码有语法错误，也能生成有效的语法树
- **多语言支持**：支持 50+ 种编程语言
- **无依赖**：生成的解析器是自包含的，不需要外部依赖

### 1.2 Aider 使用 Tree-sitter 的原因

Aider 作为一个 AI 编程助手，需要理解代码结构来为 LLM 提供上下文。使用 Tree-sitter 可以：

1. **精确提取符号**：识别函数、类、方法、变量等定义和引用
2. **语法检查**：检测代码中的语法错误
3. **构建代码地图**：生成 repo map 帮助 LLM 理解项目结构
4. **智能排序**：使用 PageRank 算法确定符号重要性

---

## 2. 核心概念

### 2.1 AST（抽象语法树）

AST 是源代码的树状结构表示，每个节点代表代码中的一个构造。

```python
# 源代码
def hello(name):
    print(f"Hello, {name}!")

# 对应的 AST 结构（简化）
# function_definition
#   ├── name: identifier "hello"
#   ├── parameters: (identifier "name")
#   └── body: block
#         └── call
#               ├── function: identifier "print"
#               └── arguments: ...
```

### 2.2 解析器 (Parser)

解析器将源代码文本转换为 AST：

```python
from grep_ast.tsl import get_parser

parser = get_parser("python")
tree = parser.parse(bytes(code, "utf-8"))
root_node = tree.root_node
```

### 2.3 查询语言 (.scm)

Tree-sitter 使用 S-expression 语法的查询语言来匹配 AST 节点：

```scm
; 匹配 Python 类定义
(class_definition
  name: (identifier) @name.definition.class) @definition.class

; 匹配 Python 函数定义
(function_definition
  name: (identifier) @name.definition.function) @definition.function

; 匹配函数调用
(call
  function: [
      (identifier) @name.reference.call
      (attribute
        attribute: (identifier) @name.reference.call)
  ]) @reference.call
```

**语法说明**：
- `(class_definition ...)` - 匹配类型为 class_definition 的节点
- `name: (identifier)` - 匹配 name 字段为 identifier 类型的子节点
- `@name.definition.class` - 命名捕获，将匹配的节点标记为此名称
- `@definition.class` - 匿名捕获，用于分类
- `[ ... ]` - 匹配多个可能的模式之一

### 2.4 捕获 (@)

捕获用于从查询结果中提取信息：

```python
from tree_sitter import Query

query = Query(language, query_scm)
captures = query.captures(root_node)

# captures 返回 (节点, 捕获名) 的列表
for node, capture_name in captures:
    if capture_name == "name.definition.function":
        print(f"函数名: {node.text.decode('utf-8')}, 行号: {node.start_point[0]}")
```

---

## 3. 依赖关系

```
aider
  │
  ├── grep_ast (封装层)
  │     ├── TreeContext          # 渲染代码上下文
  │     ├── filename_to_lang()   # 文件名 → 语言
  │     └── tsl/
  │           ├── get_language() # 获取语言对象
  │           ├── get_parser()   # 获取解析器
  │           └── USING_TSL_PACK # 使用哪个语言包的标志
  │
  ├── tree_sitter (Python 绑定)
  │     ├── Parser
  │     ├── Tree
  │     ├── Node
  │     └── Query
  │
  └── tree-sitter-languages / tree-sitter-language-pack
        └── 各语言的解析器共享库
```

**两个语言包的区别**：
- `tree-sitter-languages`：旧版，包含预编译的语言解析器
- `tree-sitter-language-pack`：新版，统一打包的语言解析器

Aider 通过 `USING_TSL_PACK` 标志自动检测使用哪个包，并使用不同的缓存版本。

---

## 4. 关键模块详解

### 4.1 repomap.py - 标签提取

#### Tag 数据结构

```python
# aider/repomap.py:29
Tag = namedtuple("Tag", "rel_fname fname line name kind".split())
```

| 字段 | 说明 |
|------|------|
| `rel_fname` | 相对于项目根目录的文件路径 |
| `fname` | 绝对文件路径 |
| `line` | 符号所在的行号（0-based） |
| `name` | 符号名称 |
| `kind` | 符号类型，如 "def"、"ref"、"definition.class"、"reference.call" 等 |

#### get_tags_raw() 核心流程

```python
def get_tags_raw(self, fname, rel_fname):
    # 1. 检测语言
    lang = filename_to_lang(fname)
    if not lang:
        return

    # 2. 获取解析器和语言对象
    language = get_language(lang)
    parser = get_parser(lang)

    # 3. 加载查询文件
    query_scm = get_scm_fname(lang)  # 查找 .scm 文件
    query_scm = query_scm.read_text()

    # 4. 解析代码
    code = self.io.read_text(fname)
    tree = parser.parse(bytes(code, "utf-8"))

    # 5. 执行查询
    query = Query(language, query_scm)
    captures = self._run_captures(query, tree.root_node)

    # 6. 提取标签
    for node, tag in captures:
        if tag.startswith("name.definition."):
            kind = "def"
        elif tag.startswith("name.reference."):
            kind = "ref"
        else:
            continue

        tag = Tag(rel_fname, fname, node.start_point[0], node.text.decode('utf-8'), kind)
        tags.append(tag)

    return tags
```

#### _run_captures() API 兼容处理

Tree-sitter 0.23 和 0.24 版本的 API 不同：

```python
def _run_captures(self, query: Query, node):
    # 兼容 tree-sitter 0.23.2 (旧 API)
    if hasattr(query, "captures"):
        return query.captures(node)

    # 兼容 tree-sitter 0.24+ (新 API)
    from tree_sitter import QueryCursor
    cursor = QueryCursor(query)
    return cursor.captures(node)
```

#### get_ranked_tags() - PageRank 排序

提取标签后，使用 PageRank 算法计算符号重要性：

```python
def get_ranked_tags(self, chat_files, other_files, max_map_tokens, ...):
    # 1. 收集所有标签
    defines = defaultdict(set)    # 符号名 → 定义所在文件
    references = defaultdict(list) # 符号名 → 引用所在文件列表

    # 2. 构建引用图 (NetworkX)
    import networkx as nx
    G = nx.DiGraph()

    # 3. 添加边：被引用的文件 → 引用它的文件
    for ref, fname in references.items():
        def_fname = defines.get(ref)
        if def_fname and def_fname != fname:
            G.add_edge(def_fname, fname, weight=1)

    # 4. PageRank 计算
    ranked = nx.pagerank(G, weight="weight", personalization=personalization)

    # 5. 按排名排序标签
    # ...
```

### 4.2 linter.py - 语法检查

#### py_lint() 方法

```python
def py_lint(self, fname, code):
    # 1. 使用 tree-sitter 基本检查
    basic_errors = self.basic_lint(fname, code)

    # 2. 编译检查
    compile_errors = self.lint_python_compile(fname, code)

    # 3. flake8 检查（只检查严重错误）
    flake8_errors = self.flake8_lint(fname, code)

    return LintResult(text=errors, lines=error_lines)
```

#### traverse_tree() - 递归检测错误

```python
def traverse_tree(node):
    """递归遍历 AST 检测语法错误"""
    errors = []

    # 检查 ERROR 节点（语法错误）
    if node.type == "ERROR":
        errors.append(node.start_point[0])

    # 检查缺失节点（缺少必要的语法元素）
    if node.is_missing:
        errors.append(node.start_point[0])

    # 递归检查子节点
    for child in node.children:
        errors += traverse_tree(child)

    return errors
```

#### TreeContext 渲染错误上下文

```python
from grep_ast import TreeContext

context = TreeContext(
    fname,
    code,
    color=False,
    line_number=True,
    child_context=False,
    last_line=False,
    margin=0,
    mark_lois=True,      # 标记关注行
    loi_pad=3,           # 关注行周围的填充
    show_top_of_file_parent_scope=False,
)
context.add_lines_of_interest(error_lines)
context.close()
```

### 4.3 查询文件 (.scm)

#### 目录结构

```
aider/queries/
├── tree-sitter-languages/      # 旧版语言包的查询
│   ├── python-tags.scm
│   ├── javascript-tags.scm
│   ├── java-tags.scm
│   └── ...
└── tree-sitter-language-pack/  # 新版语言包的查询
    ├── python-tags.scm
    ├── javascript-tags.scm
    └── ...
```

#### Python 查询示例对比

**tree-sitter-languages 版本**：
```scm
(class_definition
  name: (identifier) @name.definition.class) @definition.class

(function_definition
  name: (identifier) @name.definition.function) @definition.function

(call
  function: [
      (identifier) @name.reference.call
      (attribute
        attribute: (identifier) @name.reference.call)
  ]) @reference.call
```

**tree-sitter-language-pack 版本**（多了常量定义）：
```scm
(module (expression_statement (assignment left: (identifier) @name.definition.constant) @definition.constant))

(class_definition
  name: (identifier) @name.definition.class) @definition.class

(function_definition
  name: (identifier) @name.definition.function) @definition.function

(call
  function: [
      (identifier) @name.reference.call
      (attribute
        attribute: (identifier) @name.reference.call)
  ]) @reference.call
```

#### 其他语言查询示例

**Java** (`java-tags.scm`)：
```scm
(class_declaration
  name: (identifier) @name.definition.class) @definition.class

(method_declaration
  name: (identifier) @name.definition.method) @definition.method

(interface_declaration
  name: (identifier) @name.definition.interface) @definition.interface
```

**Rust** (`rust-tags.scm`)：
```scm
(struct_item
    name: (type_identifier) @name.definition.class) @definition.class

(function_item
    name: (identifier) @name.definition.function) @definition.function

(impl_item
    type: (type_identifier) @name.reference.impl) @reference.impl
```

---

## 5. 数据流

### 5.1 完整流程

```
┌─────────────────────────────────────────────────────────────────┐
│                        源代码文件                                │
│                   example.py (def hello(): ...)                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  1. 语言检测                                                     │
│     lang = filename_to_lang("example.py")  →  "python"          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  2. 获取解析器                                                   │
│     parser = get_parser("python")                               │
│     language = get_language("python")                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  3. 解析代码生成 AST                                             │
│     tree = parser.parse(bytes(code, "utf-8"))                   │
│     root_node = tree.root_node                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  4. 加载查询文件                                                 │
│     query_scm = read("aider/queries/.../python-tags.scm")       │
│     query = Query(language, query_scm)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  5. 执行查询，匹配 AST 节点                                      │
│     captures = query.captures(root_node)                        │
│     # [(node1, "name.definition.function"),                     │
│     #  (node2, "definition.function"), ...]                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  6. 提取标签                                                     │
│     Tag(rel_fname="example.py", fname="/path/to/example.py",    │
│         line=0, name="hello", kind="def")                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  7. 构建引用图 & PageRank 排序                                   │
│     G = nx.DiGraph()                                            │
│     ranked = nx.pagerank(G, ...)                                │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│  8. 生成 Repo Map                                               │
│     # 按重要性排序的符号列表，提供给 LLM 作为上下文              │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 缓存检查流程

```
get_tags(fname, rel_fname)
    │
    ├── 检查缓存是否存在
    │   └── 缓存键: (rel_fname, file_mtime)
    │
    ├── 缓存命中 → 返回缓存的标签
    │
    └── 缓存未命中 → get_tags_raw()
                       │
                       ├── 解析文件
                       ├── 提取标签
                       └── 存入缓存
```

---

## 6. 多语言支持

### 6.1 支持的语言

Aider 通过查询文件支持 55+ 种编程语言，包括：

| 类别 | 语言 |
|------|------|
| 脚本语言 | Python, JavaScript, TypeScript, Ruby, PHP, Lua |
| 系统语言 | C, C++, Rust, Go, Zig |
| JVM 语言 | Java, Kotlin, Scala |
| 函数式语言 | Haskell, OCaml, Elixir, Elm, Clojure |
| 其他 | Swift, Dart, Julia, R, MATLAB, Fortran, ... |

### 6.2 两套查询系统

Aider 维护两套查询文件以兼容不同的语言包：

```python
# aider/repomap.py
def get_scm_fname(lang):
    # 优先使用 tree-sitter-language-pack 版本
    if USING_TSL_PACK:
        scm_fname = Path(...) / "tree-sitter-language-pack" / f"{lang}-tags.scm"
    else:
        scm_fname = Path(...) / "tree-sitter-languages" / f"{lang}-tags.scm"
    return scm_fname
```

### 6.3 添加新语言支持

要为新语言添加支持：

1. **创建查询文件** `aider/queries/tree-sitter-languages/{lang}-tags.scm`

2. **编写查询规则**：
   ```scm
   ; 查看该语言的 tree-sitter 语法定义
   ; https://tree-sitter.github.io/tree-sitter/
   (function_definition
     name: (identifier) @name.definition.function) @definition.function
   ```

3. **测试查询**：使用 `tree-sitter-cli` 测试查询是否正确

---

## 7. 缓存与性能优化

### 7.1 缓存架构

```python
class RepoMap:
    TAGS_CACHE_DIR = f".aider.tags.cache.v{CACHE_VERSION}"

    def load_tags_cache(self):
        # 使用 diskcache 持久化到 SQLite
        self.cache = Cache(self.cache_dir)

    def __init__(self, ...):
        # 内存缓存
        self.tree_cache = {}           # AST 缓存
        self.tree_context_cache = {}   # TreeContext 缓存
        self.map_cache = {}            # Repo Map 缓存
```

### 7.2 缓存版本控制

```python
CACHE_VERSION = 3
if USING_TSL_PACK:
    CACHE_VERSION = 4  # 不同语言包使用不同缓存版本
```

当语言包切换或查询文件更新时，缓存会自动失效。

### 7.3 缓存键设计

```python
# 缓存键: (rel_fname, file_mtime)
cache_key = (rel_fname, Path(fname).stat().st_mtime)
```

基于文件修改时间，只有文件变更时才重新解析。

### 7.4 性能优化策略

1. **增量解析**：只处理修改过的文件
2. **Token 估算**：采样估算文本 token 数量，避免完整计算
3. **二分查找**：动态调整 repo map 大小以适应 token 限制
4. **进度显示**：使用 tqdm 显示处理进度
5. **回退机制**：tree-sitter 失败时回退到 pygments

---

## 8. 错误处理与回退

### 8.1 解析失败处理

```python
def get_tags_raw(self, fname, rel_fname):
    try:
        language = get_language(lang)
        parser = get_parser(lang)
    except Exception as err:
        # 解析器不可用时跳过文件
        print(f"Skipping file {fname}: {err}")
        return
```

### 8.2 缓存失败回退

```python
def load_tags_cache(self):
    try:
        self.cache = Cache(self.cache_dir)
    except SQLITE_ERRORS:
        # SQLite 错误时回退到内存字典
        self.cache = dict()
```

### 8.3 Tree-sitter 不可用时

```python
def get_tags(self, fname, rel_fname):
    # 尝试 tree-sitter 解析
    tags = self.get_tags_raw(fname, rel_fname)

    if tags is None:
        # 回退到 pygments 提取标识符
        tags = self.get_tags_pygments(fname, rel_fname)

    return tags
```

---

## 9. 总结

Aider 对 Tree-sitter 的使用展示了如何利用语法分析技术增强 AI 编程助手的能力：

1. **精确的代码理解**：通过 AST 解析准确提取代码符号
2. **智能的上下文选择**：PageRank 算法确定重要符号
3. **健壮的错误处理**：多层回退机制确保稳定性
4. **广泛的语言支持**：55+ 种编程语言
5. **高效的缓存设计**：增量更新，最小化重复计算

这套实现为 LLM 提供了高质量的代码上下文，是 Aider 能够有效理解和修改代码的关键技术基础。
