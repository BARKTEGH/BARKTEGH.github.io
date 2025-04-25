---
title: UV-Python包和项目管理器
date: 2025-04-25 19:04:48
tags:
- python
---


## 说明

[UV](https://docs.astral.sh/uv/getting-started/installation/), 一个用 Rust 编写的极快的 Python 包和项目管理器。

- 🚀一个工具即可替换`pip`、、、、、、、等等`pip-tools`。`pipx``poetry``pyenv``twine``virtualenv`
- ⚡️比[快 10-100 倍](https://github.com/astral-sh/uv/blob/main/BENCHMARKS.md)`pip`。
- 🗂️ 提供[全面的项目管理](https://github.com/astral-sh/uv#projects)，并带有 [通用的锁文件](https://docs.astral.sh/uv/concepts/projects/layout#the-lockfile)。
- ❇️[运行脚本](https://github.com/astral-sh/uv#scripts)，支持 [内联依赖元数据](https://docs.astral.sh/uv/guides/scripts#declaring-script-dependencies)。
- 🐍[安装和管理](https://github.com/astral-sh/uv#python-versions)Python 版本。
- 🛠️[运行并安装](https://github.com/astral-sh/uv#tools)作为 Python 包发布的工具。
- 🔩 包含与[pip 兼容的接口](https://github.com/astral-sh/uv#the-pip-interface)，可通过熟悉的 CLI 提高性能。
- 🏢 支持可扩展项目的Cargo 风格[工作区。](https://docs.astral.sh/uv/concepts/projects/workspaces)
- 💾 节省磁盘空间，具有用于依赖性重复数据删除的[全局缓存](https://docs.astral.sh/uv/concepts/cache)。
- ⏬ 无需 Rust 或 Python 即可通过`curl`或安装`pip`。
- 🖥️ 支持 macOS、Linux 和 Windows。


## 安装
### 独立安装

uv 提供了一个独立的安装程序来下载和安装 uv：
Widnow使用`irm`以下方式下载脚本并执行`iex`：

```
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
通过在 URL 中包含特定版本来请求它
```
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/0.6.14/install.ps1 | iex"
```

### WinGet

```
winget install --id=astral-sh.uv  -e
```
### Scoop

```
scoop install main/uv
```


安装完成后使用以下命令来检查uv是否可用：

```
uv
```
## 使用

uv 提供了 Python 开发的基本功能——从安装 Python 和支持简单脚本到处理支持多个 Python 版本和平台的大型项目。目前uv 主要分为下面几个功能,下面简单介绍一下使用。
- Python版本的管理
- 脚本执行
- 项目管理

### [项目管理](https://docs.astral.sh/uv/concepts/projects/)
uv 支持管理 Python 项目，在文件中定义其依赖关系`pyproject.toml`。
主要包含下面命令:
- `uv init`：创建一个新的 Python 项目。
- `uv add`：向项目添加依赖项。
- `uv remove`：从项目中删除依赖项。
- `uv sync`：将项目的依赖项与环境同步。
- `uv lock`：为项目的依赖项创建一个锁文件。
- `uv run`：在项目环境中运行命令。
- `uv tree`：查看项目的依赖关系树。
- `uv build`：将项目构建到分发档案中。
- `uv publish`：将项目发布到包索引。
#### 创建新项目
可以使用以下命令创建一个新的 Python 项目`uv init`：
```
uv init hello-world
cd hello-world
```

或者，您可以在工作目录中初始化一个项目：

```
mkdir hello-world
cd hello-world
uv init
```
uv 将创建以下文件：
```
.
├── .python-version
├── README.md
├── main.py
└── pyproject.toml
```

#### 项目结构
一个项目由几个重要的部分组成，它们协同工作，并允许 uv 管理你的项目。除了 由 创建的文件之外， 当你第一次运行项目命令（即、 或 ）时， `uv init`uv 还会在项目的根目录中创建一个虚拟环境和文件。完整列表如下：
```
.
├── .venv   //虚拟环境
│   ├── bin
│   ├── lib
│   └── pyvenv.cfg
├── .python-version  //包含项目的默认 Python 版本。此文件告诉 uv 在创建项目的虚拟环境时使用哪个 Python 版本。
├── README.md  
├── main.py
├── pyproject.toml    // 有关您的项目的元数据
└── uv.lock  
```
其中`uv.lock`是一个跨平台的锁文件，其中包含项目依赖项的精确信息。与`pyproject.toml`用于指定项目总体需求的 不同，锁文件包含项目环境中已安装的精确解析版本。==此文件应纳入版本控制，以便在不同机器上实现一致且可重复的安装==。`uv.lock`是人类可读的 TOML 文件，但由 uv 管理，不应手动编辑。

#### 管理依赖项
可以使用命令`uv add`添加依赖项，这也会更新锁pyproject.toml文件和项目环境```
```
uv add requests
# 指定特定的版本
uv add 'requests==2.31.0'
# 指定特定的源
uv add git+https://github.com/psf/requests
```
如果要删除依赖，可以使用`uv remove` 命令。
```
uv remove requests
```
要升级软件包，请`uv lock`使用以下`--upgrade-package`标志运行：

```
uv lock --upgrade-package requests
```

该`--upgrade-package`标志将尝试将指定的包更新到最新的兼容版本，同时保持锁文件的其余部分完好无损。

#### 运行命令

`uv run`可用于在您的项目环境中运行任意脚本或命令。

在每次调用之前`uv run`，uv 都会验证锁文件是否是最新的 `pyproject.toml`，并且环境是否是最新的锁文件，从而使您的项目保持同步，而无需手动干预。`uv run`保证您的命令在一致的、锁定的环境中运行。

例如，使用`flask`：
```
uv add flask
uv run -- flask run -p 3000
```
也可以使用`uv sync`手动更新环境，然后在执行命令之前激活它：
```
uv sync
source .venv\Scripts\activate
flask run -p 3000
```

#### 构建发行版
`uv build`可用于为您的项目构建源分布和二进制分布（轮子）。
默认情况下，`uv build`将在当前目录中构建项目，并将构建的工件放在`dist/`子目录中：

```
uv build
ls dist/
```

### Python管理

如果您的系统上已安装 Python，uv 将 [检测并使用](https://docs.astral.sh/uv/guides/install-python/#using-existing-python-versions)它，无需任何配置。但是，uv 还可以安装和管理 Python 版本。uv 会根据需要[自动安装](https://docs.astral.sh/uv/guides/install-python/#automatic-python-downloads)缺少的 Python 版本——您无需安装 Python 即可开始使用。

安装最新的版本
```
uv python install
```
安装特定的版本
```
uv python install 3.12
```
重新安装
```
uv python install --reinstall
```
查看已安装的Python版本
```
uv python list
```

### [运行脚本](https://docs.astral.sh/uv/guides/scripts/)
Python 脚本是用于独立执行的文件，例如`python <script>.py`。使用 uv 执行脚本可确保管理脚本依赖关系，而无需手动管理环境。