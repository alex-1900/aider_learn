# PageRank 技术实现详解

本文档深入解析 Aider 项目中 PageRank 算法的具体实现，包括图构建、权重计算、个性化向量等核心概念。

---

## 1. 概述

### 1.1 PageRank 在 Aider 中的作用

Aider 使用 PageRank 算法对代码文件和定义进行重要性排序，从而在有限的 token 预算内为 LLM 提供最相关的代码上下文。

**核心问题**：仓库可能有成百上千个文件，但 LLM 的上下文窗口有限。如何选择最重要的代码片段？

**解决方案**：将代码引用关系建模为有向图，使用 PageRank 算法计算每个文件的重要性。

### 1.2 为什么选择 PageRank

PageRank 的核心思想非常适合代码分析：

| 网页排名 | 代码排名 |
|---------|---------|
| 网页之间的超链接 | 文件之间的引用关系 |
| 被重要网页链接的网页更重要 | 被重要文件引用的文件更重要 |
| 链接数量和质量决定排名 | 引用数量和质量决定排名 |

**优势**：
1. **递归重要性**：不仅考虑直接引用，还考虑引用者的重要性
2. **权重可调**：可以根据引用类型调整边权重
3. **个性化支持**：可以优先展示当前上下文相关的文件

---

## 2. 数据结构

### 2.1 核心数据结构

`get_ranked_tags()` 方法使用四个核心数据结构：

```python
defines = defaultdict(set)      # 标识符 -> 定义它的文件集合
references = defaultdict(list)  # 标识符 -> 引用它的文件列表
definitions = defaultdict(set)  # (文件, 标识符) -> Tag 集合
personalization = dict()        # 文件 -> 个性化分数
```

**图解关系**：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          数据结构关系图                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  defines:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  "process_data" ──→ {"utils.py", "core/processor.py"}           │   │
│  │  "Config" ────────→ {"config.py"}                               │   │
│  │  "main" ──────────→ {"main.py"}                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  references:                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  "process_data" ──→ ["main.py", "api.py", "main.py"]            │   │
│  │  "Config" ────────→ ["main.py", "utils.py"]                     │   │
│  │  "main" ──────────→ ["cli.py"]                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  definitions:                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  ("utils.py", "process_data") ──→ {Tag(line=10, ...)}           │   │
│  │  ("config.py", "Config") ───────→ {Tag(line=5, ...)}            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  personalization:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  "main.py" ──→ 10.0    # 聊天中的文件                            │   │
│  │  "utils.py" ─→ 5.0     # 用户提到的文件                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据结构详解

#### defines（定义映射）

```python
# 格式：标识符名称 -> 定义该标识符的文件集合
defines = {
    "process_data": {"utils.py", "core/processor.py"},  # 两个文件都定义了 process_data
    "Config": {"config.py"},
    "UserModel": {"models/user.py"},
}
```

**用途**：确定边的目标节点。当文件 A 引用标识符 X 时，边从 A 指向 `defines[X]` 中的所有文件。

#### references（引用映射）

```python
# 格式：标识符名称 -> 引用该标识符的文件列表（可能有重复）
references = {
    "process_data": ["main.py", "api.py", "main.py"],  # main.py 引用了两次
    "Config": ["main.py", "utils.py"],
}
```

**用途**：确定边的源节点和边的数量。列表中的重复项表示同一文件多次引用同一标识符。

#### definitions（定义详情）

```python
# 格式：(文件名, 标识符) -> Tag 对象集合
definitions = {
    ("utils.py", "process_data"): {
        Tag(rel_fname="utils.py", fname="/path/utils.py", line=10, name="process_data", kind="def")
    },
    ("config.py", "Config"): {
        Tag(rel_fname="config.py", fname="/path/config.py", line=5, name="Config", kind="def"),
        Tag(rel_fname="config.py", fname="/path/config.py", line=50, name="Config", kind="def"),  # 嵌套类
    },
}
```

**用途**：存储定义的具体位置信息，用于最终生成代码片段。

#### personalization（个性化向量）

```python
# 格式：文件名 -> 个性化分数
personalization = {
    "main.py": 10.0,   # 在聊天中的文件
    "utils.py": 5.0,   # 用户提到的文件
    "models/user.py": 5.0,  # 路径匹配用户提到的标识符
}
```

**用途**：作为 PageRank 的个性化向量，使特定文件在排名中获得优势。

---

## 3. 图的构建

### 3.1 节点与边

**节点**：每个源代码文件是一个节点。

**边**：如果文件 A 引用了文件 B 中定义的标识符，则存在一条从 A 到 B 的有向边。

```
文件引用关系示例：

main.py:
    from utils import process_data
    from config import Config

    result = process_data(data)  # 引用 utils.py 中的 process_data
    cfg = Config()               # 引用 config.py 中的 Config

utils.py:
    from config import get_config

    def process_data(data):
        return get_config().process(data)  # 引用 config.py 中的 get_config

config.py:
    class Config: ...
    def get_config(): ...

构建的图：

        main.py
       /       \
      ↓         ↓
  utils.py ──→ config.py
```

### 3.2 MultiDiGraph 的选择

Aider 使用 `networkx.MultiDiGraph`（多重有向图）：

```python
import networkx as nx
G = nx.MultiDiGraph()
```

**为什么选择 MultiDiGraph？**

| 图类型 | 特点 | 适用场景 |
|--------|------|---------|
| `DiGraph` | 两个节点间只能有一条边 | 简单引用关系 |
| `MultiDiGraph` | 两个节点间可以有多条边 | **同一文件引用多个标识符** |

**示例**：

```
main.py 引用了 utils.py 中的两个函数：
- process_data
- validate_input

使用 MultiDiGraph：
main.py ──[process_data]──→ utils.py
main.py ──[validate_input]──→ utils.py

每条边携带不同的标识符信息！
```

### 3.3 边的数据结构

每条边存储三个属性：

```python
G.add_edge(
    referencer,    # 源节点：引用方文件
    definer,       # 目标节点：定义方文件
    weight=...,    # 边权重：综合各种因素计算
    ident=...,     # 标识符名称：这条边代表的引用
)
```

### 3.4 自环边处理

对于定义了但未被引用的标识符，添加自环边：

```python
# 为没有引用的定义添加自环边
for ident in defines.keys():
    if ident in references:
        continue  # 有引用，跳过
    for definer in defines[ident]:
        G.add_edge(definer, definer, weight=0.1, ident=ident)
```

**原因**：
1. 确保 PageRank 算法能处理这些节点
2. 自环边权重较低（0.1），不会过度影响排名
3. 解决 tree-sitter 某些版本不将定义同时识别为引用的问题

```
示例：
如果 config.py 定义了 get_config()，但没有任何文件引用它：

config.py ──[get_config, weight=0.1]──→ config.py  # 自环边
```

---

## 4. 个性化向量

### 4.1 计算逻辑

个性化向量决定了 PageRank 的"起始偏好"。Aider 通过三种方式计算个性化分数：

```python
# 基础分数：100 / 文件数量
personalize = 100 / len(fnames)

for fname in fnames:
    current_pers = 0.0  # 初始分数为 0

    # 因素 1：文件在聊天中
    if fname in chat_fnames:
        current_pers += personalize

    # 因素 2：文件被用户提到
    if rel_fname in mentioned_fnames:
        current_pers = max(current_pers, personalize)  # 避免重复计算

    # 因素 3：路径匹配用户提到的标识符
    matched_idents = path_components.intersection(mentioned_idents)
    if matched_idents:
        current_pers += personalize

    if current_pers > 0:
        personalization[rel_fname] = current_pers
```

### 4.2 三种个性化因素

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        个性化向量计算流程                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  文件: main.py                                                          │
│  personalize = 100 / 10 = 10.0                                          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 因素 1: 在聊天中？                                               │   │
│  │ main.py in chat_fnames ──→ True                                 │   │
│  │ current_pers += 10.0 → current_pers = 10.0                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 因素 2: 被用户提到？                                             │   │
│  │ "main.py" in mentioned_fnames ──→ False                         │   │
│  │ (无变化)                                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 因素 3: 路径匹配提到的标识符？                                    │   │
│  │ mentioned_idents = {"UserModel", "api"}                         │   │
│  │ path_components = {"main.py"}                                   │   │
│  │ 无匹配                                                          │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  最终: personalization["main.py"] = 10.0                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 路径匹配详解

```python
# 路径匹配逻辑
path_obj = Path(rel_fname)  # 例如 "models/user.py"
path_components = set(path_obj.parts)  # {"models", "user.py"}

basename_with_ext = path_obj.name  # "user.py"
basename_without_ext, _ = os.path.splitext(basename_with_ext)  # "user"

components_to_check = path_components.union({basename_with_ext, basename_without_ext})
# {"models", "user.py", "user"}

matched_idents = components_to_check.intersection(mentioned_idents)
if matched_idents:
    current_pers += personalize
```

**匹配示例**：

| 文件路径 | mentioned_idents | 匹配结果 |
|---------|-----------------|---------|
| `models/user.py` | `{"user"}` | 匹配（basename 无扩展名） |
| `api/handlers.py` | `{"api"}` | 匹配（路径组件） |
| `utils/helpers.py` | `{"UserModel"}` | 不匹配 |

### 4.4 个性化向量在 PageRank 中的作用

```python
if personalization:
    pers_args = dict(personalization=personalization, dangling=personalization)
else:
    pers_args = dict()

ranked = nx.pagerank(G, weight="weight", **pers_args)
```

**参数说明**：

| 参数 | 作用 |
|------|------|
| `personalization` | 指定节点的初始概率分布 |
| `dangling` | 处理无出边节点（dangling nodes）时使用的分布 |

**数学含义**：

标准 PageRank：
```
PR(p) = (1-d)/N + d * Σ(PR(q)/L(q))  for all q linking to p
```

个性化 PageRank：
```
PR(p) = (1-d)*P(p) + d * Σ(PR(q)/L(q))  for all q linking to p
```

其中 `P(p)` 是个性化向量中节点 p 的值。

---

## 5. 边权重计算

### 5.1 权重计算公式

边权重是多个因子的乘积：

```
weight = base_mul * ident_mul * chat_mul * sqrt(num_refs)
```

### 5.2 基础权重因子

```python
mul = 1.0  # 初始权重

# === 标识符命名风格检查 ===
is_snake = ("_" in ident) and any(c.isalpha() for c in ident)  # snake_case
is_kebab = ("-" in ident) and any(c.isalpha() for c in ident)  # kebab-case
is_camel = any(c.isupper() for c in ident) and any(c.islower() for c in ident)  # camelCase
```

### 5.3 提权因子

| 条件 | 乘数 | 原因 |
|------|------|------|
| 用户提到的标识符 | ×10 | 提高相关性 |
| 有意义的命名风格 + 长度 ≥ 8 | ×10 | 过滤短名称和无意义标识符 |

```python
# 用户提到的标识符大幅提权
if ident in mentioned_idents:
    mul *= 10

# 有意义的命名风格提权（过滤 i, j, x 等短变量名）
if (is_snake or is_kebab or is_camel) and len(ident) >= 8:
    mul *= 10
```

**命名风格检测示例**：

| 标识符 | is_snake | is_kebab | is_camel | 长度 ≥ 8 | 提权 |
|--------|----------|----------|----------|---------|------|
| `process_data` | ✓ | ✗ | ✗ | ✓ | ×10 |
| `user-name` | ✗ | ✓ | ✗ | ✓ | ×10 |
| `UserModel` | ✗ | ✗ | ✓ | ✓ | ×10 |
| `i` | ✗ | ✗ | ✗ | ✗ | ×1 |
| `idx` | ✗ | ✗ | ✗ | ✗ | ×1 |
| `getData` | ✗ | ✗ | ✓ | ✗ (7) | ×1 |

### 5.4 降权因子

| 条件 | 乘数 | 原因 |
|------|------|------|
| 以 `_` 开头 | ×0.1 | 私有标识符，通常不需要外部引用 |
| 定义次数 > 5 | ×0.1 | 通用名称，如 `get`, `set`, `data` |

```python
# 私有标识符降权
if ident.startswith("_"):
    mul *= 0.1

# 过度定义的标识符降权（可能是通用名称）
if len(defines[ident]) > 5:
    mul *= 0.1
```

**过度定义示例**：

```python
# 多个文件都定义了同名函数
defines = {
    "get": {"api.py", "db.py", "cache.py", "config.py", "utils.py", "models.py"},
    # 定义次数 = 6 > 5，降权
}
```

### 5.5 聊天文件引用加成

```python
for referencer, num_refs in Counter(references[ident]).items():
    for definer in definers:
        use_mul = mul

        # 聊天文件中的引用大幅提权
        if referencer in chat_rel_fnames:
            use_mul *= 50

        # 平滑高频引用
        num_refs = math.sqrt(num_refs)

        G.add_edge(referencer, definer, weight=use_mul * num_refs, ident=ident)
```

**聊天文件加成示例**：

```
场景：main.py 在聊天中，引用了 utils.py 中的 process_data

普通文件引用：
  other.py ──→ utils.py, weight = 1.0 * sqrt(1) = 1.0

聊天文件引用：
  main.py ──→ utils.py, weight = 1.0 * 50 * sqrt(1) = 50.0
```

### 5.6 引用次数平滑

```python
num_refs = math.sqrt(num_refs)  # 使用平方根平滑
```

**原因**：防止高频引用过度影响权重。

**示例**：

| 实际引用次数 | 平滑后 |
|-------------|-------|
| 1 | 1.0 |
| 4 | 2.0 |
| 9 | 3.0 |
| 100 | 10.0 |

### 5.7 权重计算完整示例

```
标识符: process_data
定义文件: utils.py
引用情况: main.py 引用 4 次，api.py 引用 1 次
mentioned_idents: {"process_data"}  # 用户提到了
chat_rel_fnames: {"main.py"}  # main.py 在聊天中

计算过程：

1. 基础权重: mul = 1.0

2. 用户提到: mul *= 10 → mul = 10.0

3. 命名风格: is_snake = True, len = 12 ≥ 8 → mul *= 10 → mul = 100.0

4. 私有检查: 不以 _ 开头 → 无变化

5. 定义次数: len(defines["process_data"]) = 1 ≤ 5 → 无变化

边权重计算：

main.py → utils.py:
  use_mul = 100.0 * 50 (聊天文件) = 5000.0
  weight = 5000.0 * sqrt(4) = 5000.0 * 2 = 10000.0

api.py → utils.py:
  use_mul = 100.0 (非聊天文件)
  weight = 100.0 * sqrt(1) = 100.0
```

---

## 6. PageRank 调用

### 6.1 调用方式

```python
if personalization:
    pers_args = dict(personalization=personalization, dangling=personalization)
else:
    pers_args = dict()

try:
    ranked = nx.pagerank(G, weight="weight", **pers_args)
except ZeroDivisionError:
    # Issue #1536: 处理空图或异常情况
    try:
        ranked = nx.pagerank(G, weight="weight")
    except ZeroDivisionError:
        return []  # 完全无法计算，返回空列表
```

### 6.2 networkx.pagerank 参数

```python
nx.pagerank(
    G,                          # 图对象
    alpha=0.85,                 # 阻尼系数（默认值）
    personalization=...,        # 个性化向量
    dangling=...,              # dangling 节点处理
    weight="weight",           # 使用边的 weight 属性
)
```

**参数详解**：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `alpha` | 0.85 | 阻尼系数，表示继续随机跳转的概率 |
| `personalization` | None | 个性化向量，决定起始分布 |
| `dangling` | None | dangling 节点的转移分布 |
| `weight` | None | 边权重属性名 |

### 6.3 dangling 节点处理

**dangling 节点**：没有出边的节点。

```
示例：
  A ──→ B ──→ C
           ↑
           D  (dangling: D 没有出边)

处理方式：
当随机游走到达 D 时，按照 dangling 参数指定的分布跳转
```

Aider 将 `dangling` 设置为与 `personalization` 相同：

```python
dangling=personalization
```

这意味着当游走到达 dangling 节点时，会优先跳转到聊天中的文件或用户提到的文件。

### 6.4 错误处理

```python
try:
    ranked = nx.pagerank(G, weight="weight", **pers_args)
except ZeroDivisionError:
    # 第一次尝试：不使用个性化向量
    try:
        ranked = nx.pagerank(G, weight="weight")
    except ZeroDivisionError:
        # 第二次尝试：完全失败
        return []
```

**ZeroDivisionError 的原因**：
1. 图为空（没有节点）
2. 所有节点都是孤立的
3. 个性化向量总和为 0

---

## 7. 排名结果转换

### 7.1 从文件排名到定义排名

PageRank 返回的是文件级别的排名，但最终需要的是定义级别的排名。

```python
# ranked: {文件名 -> PageRank 分数}
ranked = {"main.py": 0.05, "utils.py": 0.03, "config.py": 0.02}
```

### 7.2 分配算法

将文件的 PageRank 分数按边权重比例分配给其出边指向的定义：

```python
ranked_definitions = defaultdict(float)

for src in G.nodes:
    src_rank = ranked[src]  # 源文件的 PageRank 分数

    # 计算源文件所有出边的总权重
    total_weight = sum(
        data["weight"]
        for _src, _dst, data in G.out_edges(src, data=True)
    )

    # 按权重比例分配分数
    for _src, dst, data in G.out_edges(src, data=True):
        data["rank"] = src_rank * data["weight"] / total_weight
        ident = data["ident"]
        ranked_definitions[(dst, ident)] += data["rank"]
```

### 7.3 分配示例

```
图结构：
  main.py (rank=0.05)
    ├── [process_data, weight=100] ──→ utils.py
    └── [Config, weight=50] ──→ config.py

计算：
  total_weight = 100 + 50 = 150

  边 1: rank = 0.05 * 100 / 150 = 0.0333
    ranked_definitions[("utils.py", "process_data")] += 0.0333

  边 2: rank = 0.05 * 50 / 150 = 0.0167
    ranked_definitions[("config.py", "Config")] += 0.0167
```

### 7.4 最终标签列表生成

```python
ranked_tags = []

# 按分数降序排序
ranked_definitions = sorted(
    ranked_definitions.items(),
    reverse=True,
    key=lambda x: (x[1], x[0])  # 先按分数，再按名称（保证稳定性）
)

for (fname, ident), rank in ranked_definitions:
    # 跳过聊天中的文件（已经在上下文中）
    if fname in chat_rel_fnames:
        continue

    # 添加该定义的所有 Tag
    ranked_tags += list(definitions.get((fname, ident), []))

# 添加没有标签的高排名文件
for rank, fname in top_rank:
    if fname not in fnames_already_included:
        ranked_tags.append((fname,))

# 添加其他文件（无标签、低排名）
for fname in rel_other_fnames_without_tags:
    ranked_tags.append((fname,))
```

### 7.5 输出格式

`ranked_tags` 是一个列表，元素可能是：

1. **Tag 对象**：包含具体定义信息
   ```python
   Tag(rel_fname="utils.py", fname="/path/utils.py", line=10, name="process_data", kind="def")
   ```

2. **文件名元组**：仅包含文件名
   ```python
   ("config.py",)
   ```

---

## 8. 完整示例

### 8.1 示例仓库结构

```
project/
├── main.py          # 入口文件
├── utils.py         # 工具函数
├── config.py        # 配置管理
├── models/
│   └── user.py      # 用户模型
└── api.py           # API 处理
```

### 8.2 代码内容

```python
# main.py
from utils import process_data
from config import Config
from models.user import UserModel

def main():
    cfg = Config()
    data = UserModel.get_data()
    result = process_data(data)
    return result

# utils.py
from config import get_config

def process_data(data):
    config = get_config()
    return config.transform(data)

def validate_input(input_data):
    return input_data is not None

# config.py
class Config:
    def __init__(self):
        self.settings = {}

    def transform(self, data):
        return data

def get_config():
    return Config()

# models/user.py
class UserModel:
    @staticmethod
    def get_data():
        return {}

# api.py
from utils import process_data, validate_input

def handle_request(request):
    if validate_input(request.data):
        return process_data(request.data)
```

### 8.3 标签提取结果

```python
defines = {
    "main": {"main.py"},
    "process_data": {"utils.py"},
    "Config": {"config.py"},
    "get_config": {"config.py"},
    "UserModel": {"models/user.py"},
    "get_data": {"models/user.py"},
    "validate_input": {"utils.py"},
    "handle_request": {"api.py"},
}

references = {
    "process_data": ["main.py", "api.py"],
    "Config": ["main.py"],
    "get_config": ["utils.py"],
    "UserModel": ["main.py"],
    "get_data": ["main.py"],
    "validate_input": ["api.py"],
}
```

### 8.4 图构建过程

```
假设：chat_fnames = {"main.py"}, mentioned_idents = {"UserModel"}

个性化向量：
  personalization = {
      "main.py": 10.0,  # 在聊天中
  }

边权重计算（简化版，不考虑命名风格）：

main.py → utils.py [process_data]:
  mul = 1.0, chat_mul = 50
  weight = 1.0 * 50 * sqrt(1) = 50.0

main.py → config.py [Config]:
  mul = 1.0, chat_mul = 50
  weight = 1.0 * 50 * sqrt(1) = 50.0

main.py → models/user.py [UserModel]:
  mul = 10.0 (用户提到), chat_mul = 50
  weight = 10.0 * 50 * sqrt(1) = 500.0

main.py → models/user.py [get_data]:
  mul = 1.0, chat_mul = 50
  weight = 1.0 * 50 * sqrt(1) = 50.0

utils.py → config.py [get_config]:
  mul = 1.0 (非聊天文件引用)
  weight = 1.0 * sqrt(1) = 1.0

api.py → utils.py [process_data]:
  mul = 1.0
  weight = 1.0 * sqrt(1) = 1.0

api.py → utils.py [validate_input]:
  mul = 1.0
  weight = 1.0 * sqrt(1) = 1.0
```

### 8.5 图可视化

```
                    main.py (chat)
                   /   |    |    \
                  /    |    |     \
            50.0/  50.0|    |500.0 \
               /       |    |       \
              ↓        ↓    ↓        ↓
         utils.py  config.py  models/user.py
            ↑          ↑
          1.0        1.0
            |          |
         api.py ────────

边权重图例：
  500.0: 用户提到的标识符 + 聊天文件引用
  50.0:  聊天文件引用
  1.0:   普通引用
```

### 8.6 PageRank 计算

```python
# 使用 networkx.pagerank 计算（简化结果）
ranked = {
    "main.py": 0.25,      # 高分：聊天文件 + 多条高权重出边
    "models/user.py": 0.20,  # 高分：被高权重边指向
    "utils.py": 0.18,     # 被 main.py 和 api.py 指向
    "config.py": 0.17,    # 被 main.py 和 utils.py 指向
    "api.py": 0.10,       # 普通文件
}
```

### 8.7 定义排名计算

```
main.py (rank=0.25) 的出边：
  total_weight = 50 + 50 + 500 + 50 = 650

  utils.py/process_data: 0.25 * 50 / 650 = 0.0192
  config.py/Config: 0.25 * 50 / 650 = 0.0192
  models/user.py/UserModel: 0.25 * 500 / 650 = 0.1923
  models/user.py/get_data: 0.25 * 50 / 650 = 0.0192

utils.py (rank=0.18) 的出边：
  total_weight = 1

  config.py/get_config: 0.18 * 1 / 1 = 0.18

api.py (rank=0.10) 的出边：
  total_weight = 1 + 1 = 2

  utils.py/process_data: 0.10 * 1 / 2 = 0.05
  utils.py/validate_input: 0.10 * 1 / 2 = 0.05

最终 ranked_definitions（累加后）：
  ("models/user.py", "UserModel"): 0.1923
  ("config.py", "get_config"): 0.18
  ("utils.py", "process_data"): 0.0192 + 0.05 = 0.0692
  ("config.py", "Config"): 0.0192
  ("models/user.py", "get_data"): 0.0192
  ("utils.py", "validate_input"): 0.05
```

### 8.8 最终输出

```python
ranked_tags = [
    # UserModel 定义（最高分）
    Tag(rel_fname="models/user.py", line=0, name="UserModel", kind="def"),

    # get_config 定义
    Tag(rel_fname="config.py", line=10, name="get_config", kind="def"),

    # process_data 定义
    Tag(rel_fname="utils.py", line=5, name="process_data", kind="def"),

    # validate_input 定义
    Tag(rel_fname="utils.py", line=15, name="validate_input", kind="def"),

    # Config 定义
    Tag(rel_fname="config.py", line=0, name="Config", kind="def"),

    # get_data 定义
    Tag(rel_fname="models/user.py", line=5, name="get_data", kind="def"),

    # 其他文件（无标签）
    ("api.py",),
]
```

---

## 9. 算法优化与边界情况

### 9.1 自环边处理

```python
# 为没有引用的定义添加自环边
for ident in defines.keys():
    if ident in references:
        continue
    for definer in defines[ident]:
        G.add_edge(definer, definer, weight=0.1, ident=ident)
```

**问题**：某些 tree-sitter 版本不会将定义同时识别为引用。

**示例**：
```python
# tree-sitter 0.23.2 解析 Ruby 代码
def greet(name)  # 只识别为 "def"，不识别为 "ref"
  puts name
end
```

**解决**：添加低权重自环边，确保节点参与 PageRank 计算。

### 9.2 空引用处理

```python
if not references:
    references = dict((k, list(v)) for k, v in defines.items())
```

**场景**：当仓库中只有定义没有引用时（例如纯数据文件），将定义文件作为引用来源。

### 9.3 ZeroDivisionError 处理

```python
try:
    ranked = nx.pagerank(G, weight="weight", **pers_args)
except ZeroDivisionError:
    try:
        ranked = nx.pagerank(G, weight="weight")
    except ZeroDivisionError:
        return []
```

**触发条件**：
1. 图为空（没有文件被解析）
2. 所有边权重为 0
3. 个性化向量总和为 0

### 9.4 性能优化

**缓存机制**：

```python
# 文件解析结果缓存
self.TAGS_CACHE[fname] = {"mtime": mtime, "data": tags}

# 地图结果缓存
self.map_cache[cache_key] = result
```

**进度显示**：

```python
if len(fnames) - cache_size > 100:
    fnames = tqdm(fnames, desc="Scanning repo")
```

大型仓库首次扫描时显示进度条。

### 9.5 边界情况汇总

| 情况 | 处理方式 |
|------|---------|
| 文件不存在 | 警告并跳过 |
| 语言不支持 | 跳过文件 |
| 解析失败 | 跳过文件 |
| 无引用关系 | 使用定义作为引用 |
| 图为空 | 返回空列表 |
| 个性化向量为空 | 使用默认分布 |

---

## 10. 总结

### 10.1 算法流程图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        PageRank 算法完整流程                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  输入: chat_fnames, other_fnames, mentioned_fnames, mentioned_idents    │
│                                                                         │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  1. 初始化数据结构                                               │   │
│  │     defines, references, definitions, personalization            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  2. 遍历所有文件                                                 │   │
│  │     - 计算个性化分数                                             │   │
│  │     - 解析标签 (get_tags)                                        │   │
│  │     - 填充 defines, references, definitions                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  3. 构建图 (MultiDiGraph)                                        │   │
│  │     - 添加自环边（无引用的定义）                                   │   │
│  │     - 计算边权重（命名风格、用户提到、聊天文件）                    │   │
│  │     - 添加引用边                                                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  4. 运行 PageRank                                                │   │
│  │     - 设置个性化向量                                             │   │
│  │     - 处理 dangling 节点                                         │   │
│  │     - 错误处理                                                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  5. 分配排名到定义                                               │   │
│  │     - 按边权重比例分配文件分数                                    │   │
│  │     - 累加得到定义分数                                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │  6. 生成排序后的标签列表                                         │   │
│  │     - 排序定义                                                   │   │
│  │     - 过滤聊天文件                                               │   │
│  │     - 添加无标签文件                                             │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              ▼                                          │
│  输出: ranked_tags (排序后的标签列表)                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 10.2 关键设计决策

| 决策 | 原因 |
|------|------|
| 使用 MultiDiGraph | 支持同一文件对多个标识符的引用 |
| 个性化向量 = dangling | 保持一致性，优先跳转到相关文件 |
| 平方根平滑引用次数 | 防止高频引用过度影响 |
| 聊天文件引用 ×50 | 大幅提升当前上下文相关性 |
| 自环边权重 0.1 | 确保孤立节点参与计算但不影响排名 |

### 10.3 与标准 PageRank 的差异

| 特性 | 标准 PageRank | Aider PageRank |
|------|--------------|----------------|
| 图类型 | 静态网页图 | 动态代码引用图 |
| 边权重 | 均匀或基于链接数 | 多因子综合计算 |
| 个性化 | 可选 | 核心特性 |
| dangling 处理 | 均匀分布 | 个性化分布 |
| 输出 | 页面排名 | 定义排名 + 文件排名 |
