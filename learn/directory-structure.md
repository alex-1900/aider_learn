# Aider 项目目录结构

## 项目概述

Aider 是一个终端中的 AI 结对编程工具 - 一个 CLI 工具，让用户可以与 LLM 进行结对编程，编辑 git 仓库中的代码。它支持多个 LLM 提供商和多种代码修改的编辑格式。

---

## 根目录文件

### 配置文件
| 文件 | 说明 |
|------|------|
| `pyproject.toml` | Python 项目配置，包含依赖和构建信息 |
| `pytest.ini` | pytest 测试框架配置 |
| `requirements.txt` | 核心依赖列表 |
| `.pre-commit-config.yaml` | pre-commit 钩子配置 |
| `.flake8` | flake8 代码检查配置 |
| `.gitignore` | Git 忽略规则 |

### 文档文件
| 文件 | 说明 |
|------|------|
| `README.md` | 项目主文档 |
| `CONTRIBUTING.md` | 贡献指南 |
| `HISTORY.md` | 版本历史记录 |
| `LICENSE.txt` | 许可证文件 |
| `CLAUDE.md` | Claude Code 项目指南 |

---

## 主要目录

### `aider/` - 核心 Python 包

项目的核心代码，包含所有主要功能模块。

#### 核心模块
| 文件 | 说明 |
|------|------|
| `main.py` | CLI 入口点，参数解析，初始化设置 |
| `__main__.py` | 支持通过 `python -m aider` 运行 |
| `models.py` | 模型管理，封装 LLM 配置，模型别名 |
| `io.py` | 输入输出系统，终端 I/O 处理 |
| `commands.py` | 斜杠命令实现 (/add, /drop, /diff 等) |
| `repo.py` | Git 集成，自动提交，管理 .gitignore |
| `repomap.py` | 代码映射，使用 tree-sitter 构建代码库地图 |
| `args.py` | 命令行参数定义 |

#### 其他重要模块
| 文件 | 说明 |
|------|------|
| `llm.py` | litellm 懒加载，统一 LLM API 访问 |
| `linter.py` | 代码检查功能 |
| `analytics.py` | 分析数据收集 |
| `editor.py` | 编辑器集成 |
| `gui.py` | 图形界面功能 |
| `scrape.py` | 网页抓取功能 |
| `voice.py` | 语音输入功能 |
| `watch.py` | 文件监视功能 |
| `history.py` | 历史记录管理 |
| `onboarding.py` | 新用户引导流程 |
| `utils.py` | 工具函数 |
| `exceptions.py` | 异常定义 |

---

### `aider/coders/` - 编辑器实现

不同编辑格式的 Coder 类实现，每个 Coder 决定如何解析 LLM 响应并应用代码修改。

| 文件 | 说明 |
|------|------|
| `base_coder.py` | 基类 Coder，处理主聊天循环、文件管理、LLM 交互 |
| `editblock_coder.py` | 搜索/替换块格式（默认 "diff" 格式） |
| `wholefile_coder.py` | 完整文件重写格式 |
| `udiff_coder.py` | 标准统一 diff 格式 |
| `patch_coder.py` | Git patch 格式 |
| `architect_coder.py` | 架构师模式 |
| `ask_coder.py` | 问答模式 |
| `help_coder.py` | 帮助模式 |
| `context_coder.py` | 上下文模式 |
| `shell.py` | Shell 命令处理 |
| `search_replace.py` | 搜索替换功能 |

每个 Coder 都有对应的 `*_prompts.py` 文件，定义指导 LLM 如何格式化编辑的系统提示词。

---

### `aider/queries/` - Tree-sitter 语言包

用于代码解析的 tree-sitter 查询文件，支持多种编程语言。

#### 子目录
- `tree-sitter-language-pack/` - 新版语言包（约 30 种语言）
- `tree-sitter-languages/` - 旧版语言包

支持的语言包括：Python, JavaScript, TypeScript, Go, Rust, Java, C/C++, Ruby, Elixir, OCaml 等。

---

### `aider/resources/` - 静态资源

| 文件 | 说明 |
|------|------|
| `model-metadata.json` | 模型元数据 |
| `model-settings.yml` | 模型设置配置 |

---

### `aider/website/` - 静态网站

基于 Jekyll 的项目官方网站。

#### 子目录
| 目录 | 说明 |
|------|------|
| `_data/` | 排行榜和基准测试数据 (YAML) |
| `_includes/` | 可复用的页面组件 |
| `_layouts/` | 页面布局模板 |
| `_posts/` | 博客文章 (2023-2025) |
| `_sass/` | 样式文件 |
| `assets/` | 图片、视频、音频等静态资源 |
| `docs/` | 文档页面 |
| `examples/` | 使用示例 |

---

### `tests/` - 测试套件

按功能组织的完整测试套件。

| 目录 | 说明 |
|------|------|
| `basic/` | 基础功能测试（约 30 个测试文件） |
| `browser/` | 浏览器相关测试 |
| `help/` | 帮助功能测试 |
| `scrape/` | 网页抓取测试 |
| `fixtures/` | 测试数据和示例代码 |

#### fixtures 子目录
- `languages/` - 各种编程语言的测试文件
- `sample-code-base/` - 示例代码库

---

### `benchmark/` - 性能基准测试

SWE-bench 集成和性能评估工具。

| 文件 | 说明 |
|------|------|
| `benchmark.py` | 基准测试主程序 |
| `swe_bench.py` | SWE-bench 集成 |
| `plots.py` | 绘图工具 |
| `over_time.py` | 时间趋势分析 |
| `prompts.py` | 测试提示词 |

---

### `scripts/` - 开发实用脚本

开发和维护相关的实用工具脚本。

| 文件 | 说明 |
|------|------|
| `versionbump.py` | 版本号管理 |
| `homepage.py` | 主页生成 |
| `update-history.py` | 更新历史记录 |
| `blame.py` | Git blame 相关 |
| `jekyll_build.sh` | Jekyll 构建脚本 |
| `pip-compile.sh` | 依赖编译脚本 |

---

### `requirements/` - 依赖文件

按用途分离的依赖管理。

| 文件 | 说明 |
|------|------|
| `requirements.in` | 核心依赖源文件 |
| `requirements-dev.in` | 开发依赖 |
| `requirements-browser.in` | 浏览器功能依赖 |
| `requirements-playwright.in` | Playwright 依赖 |
| `requirements-help.in` | 帮助功能依赖 |
| `tree-sitter.in` | tree-sitter 依赖 |

每个 `.in` 文件都有对应的 `.txt` 编译后文件。

---

### `.github/` - GitHub 配置

| 目录 | 说明 |
|------|------|
| `workflows/` | CI/CD 工作流（测试、发布、Docker 等） |
| `ISSUE_TEMPLATE/` | Issue 模板 |

---

### `docker/` - Docker 配置

| 文件 | 说明 |
|------|------|
| `Dockerfile` | Docker 镜像构建文件 |

---

### `learn/` - 项目学习笔记

用于存放项目学习和分析笔记的目录。

---

## 架构关系图

```
┌─────────────────────────────────────────────────────────────┐
│                         CLI 入口                             │
│  main.py → args.py (参数解析) → Coder 实例化                │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       Coder 核心                             │
│  base_coder.py                                              │
│  ├── 聊天循环                                                │
│  ├── 文件管理                                                │
│  └── LLM 交互                                               │
└─────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
   │ EditBlock   │     │ WholeFile   │     │ UnifiedDiff │
   │ Coder       │     │ Coder       │     │ Coder       │
   └─────────────┘     └─────────────┘     └─────────────┘

┌─────────────────────────────────────────────────────────────┐
│                       支撑模块                               │
│  ├── io.py (I/O)          ├── repo.py (Git)                │
│  ├── models.py (LLM)      ├── repomap.py (代码地图)        │
│  └── commands.py (命令)    └── linter.py (检查)            │
└─────────────────────────────────────────────────────────────┘
```

---

## 代码风格

- **无类型注解** - 项目明确避免使用类型注解
- **行长度**: 最大 100 字符
- **格式化**: Black + isort (black profile)
- **Python 版本**: 3.10 到 3.14

---

## 关键模式

- `from .dump import dump` 用于调试
- 测试使用 `unittest.TestCase` 配合 `MagicMock`/`patch`
- `GitTemporaryDirectory` 上下文管理器用于创建临时 git 仓库
- LLM 通信通过 litellm 抽象层
