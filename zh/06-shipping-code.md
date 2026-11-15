# 打包与发布代码

让代码按预期工作很难；让同一段代码在不同于你自己的机器上运行往往更难。

发布代码（shipping code）指的是把你写的代码转换成一种可用的形式，让别人无需具备与你完全相同的计算机环境也能运行它。发布代码有许多形式，取决于编程语言、系统库、操作系统等许多因素的选择。它还取决于你构建的东西：软件库、命令行工具和 Web 服务都有不同的需求和部署步骤。无论如何，所有这些场景有一个共同的模式：我们需要定义交付物是什么 —— 即 *构建产物*（artifact）—— 以及它对周围环境作了哪些假设。

本讲将涵盖：

* [依赖与环境](#dependencies--environments)
* [构建产物与打包](#artifacts--packaging)
* [发布与版本管理](#releases--versioning)
* [可复现性](#reproducibility)
* [虚拟机与容器](#vms--containers)
* [配置](#configuration)
* [服务与编排](#services--orchestration)
* [发布](#publishing)

我们将通过 Python 生态系统中的例子来解释这些概念，因为具体的例子有助于理解。虽然其他编程语言生态系统的工具各不相同，但背后的概念基本一致。

# 依赖与环境

在现代软件开发中，抽象层无处不在。程序自然地把逻辑分派给其他库或服务。然而，这会在你的程序和它正常运行所需的库之间引入一种 *依赖*（dependency）关系。例如，在 Python 中，要获取一个网站的内容，我们通常这样做：

```
import requests response = requests. get("https://missing.csail.mit.edu")
```

然而 `requests` 库并不随 Python 运行时捆绑发布，所以如果我们没有安装 `requests` 就尝试运行这段代码，Python 会抛出错误：

```
$  Traceback (most recent call last):  File "fetch.py", line 1, in   import requests ModuleNotFoundError: No module named 'requests' 
```

为了能使用这个库，我们需要先运行 `pip install requests` 来安装它。`pip` 是 Python 编程语言提供的用于安装包的命令行工具。执行 `pip install requests` 会依次产生以下动作：

1. 在 Python 包索引（[PyPI](https://pypi.org/)）中搜索 requests
2. 为我们要在其中运行的平台搜索合适的构建产物
3. 解析依赖 —— `requests` 库本身还依赖其他包，因此安装器必须找到所有传递依赖（transitive dependency）的兼容版本并提前安装它们
4. 下载构建产物，然后解包并将文件复制到我们文件系统中正确的位置

```
$  install Collecting requests Downloading requests-2.32.3-py3-none-any.whl (64 kB) Collecting charset-normalizer<4,> = Downloading charset_normalizer-3.4.0-cp311-cp311-manylinux_x86_64.whl (142 kB) Collecting idna<4,> = Downloading idna-3.10-py3-none-any.whl (70 kB) Collecting urllib3<3,> = Downloading urllib3-2.2.3-py3-none-any.whl (126 kB) Collecting certifi> = Downloading certifi-2024.8.30-py3-none-any.whl (167 kB) Installing collected packages: urllib3, idna, charset-normalizer, certifi, requests Successfully installed certifi-2024.8.30 charset-normalizer-3.4.0 idna-3.10 requests-2.32.3 urllib3-2.2.3 
```

这里可以看到 `requests` 有自己的依赖，例如 `certifi` 或 `charset-normalizer`，而且必须先安装它们才能安装 `requests`。安装完成后，Python 运行时在导入时就能找到这个库。

```
$  -c'import requests; print(requests.__path__)'['/usr/local/lib/python3.11/dist-packages/requests'] $  grep requests 2.32.3 
```

各种编程语言有不同的工具、约定和实践来安装和发布库。在某些语言（如 Rust）中，工具链是统一的 —— `cargo` 负责构建、测试、依赖管理和发布。在另一些语言（如 Python）中，统一发生在规范层面 —— 不是由单一工具统一，而是有标准化的规范定义打包如何工作，从而每个环节都有多个相互竞争的工具（`pip` vs [`uv`](https://docs.astral.sh/uv/)，`setuptools` vs [`hatch`](https://hatch.pypa.io/) vs [`poetry`](https://python-poetry.org/)）。而在像 LaTeX 这样的生态系统中，TeX Live 或 MacTeX 等发行版捆绑预装了成千上万个包。

引入依赖也会引入依赖冲突。当程序要求同一依赖的不兼容版本时就会发生冲突。例如，如果 `tensorflow==2.3.0` 要求 `numpy>=1.16.0,<1.19.0`，而 `pandas==1.2.0` 要求 `numpy>=1.16.5`，那么任何满足 `numpy>=1.16.5,<1.19.0` 的版本都是有效的。但如果你的项目中另一个包要求 `numpy>=1.19`，就产生了冲突，没有任何版本能同时满足所有约束。

这种多个包对共享依赖要求互相不兼容版本的情况，通常被称为 *依赖地狱*（dependency hell）。应对冲突的一种方法是把每个程序的依赖隔离到各自的 *环境*（environment）中。在 Python 中，我们通过运行以下命令创建虚拟环境：

```
$  /usr/bin/python $  pwd/home/missingsemester $  -m $  source $  /home/missingsemester/venv/bin/python $  /home/missingsemester/venv/bin/pip $  -c'import requests; print(requests.__path__)'['/home/missingsemester/venv/lib/python3.11/site-packages/requests'] $  Package Version ------- ------- pip 24.0 
```

你可以把环境想象成一个完整的、独立的语言运行时版本，带有自己的一组已安装的包。这种虚拟环境（virtual environment，venv）将已安装的依赖与全局 Python 安装隔离开来。最佳实践是为每个项目建立一个虚拟环境，包含它所需要的依赖。

> 虽然许多现代操作系统都自带 Python 等编程语言运行时，但修改这些安装并不明智，因为操作系统自身的功能可能依赖它们。请改用独立的环境。

在某些语言中，安装协议不是由某个工具定义，而是以规范的形式定义。在 Python 中，[PEP 517](https://peps.python.org/pep-0517/) 定义了构建系统接口，[PEP 621](https://peps.python.org/pep-0621/) 规定了项目元数据如何存储在 `pyproject.toml` 中。这使得开发者能够在 `pip` 之上加以改进，产出更优化的工具，比如 `uv`。安装 `uv` 只需执行 `pip install uv`。

用 `uv` 代替 `pip` 遵循相同的接口，但速度要快得多：

```
$  install Resolved 5 packages in 12ms Prepared 5 packages in 0.45ms Installed 5 packages in 8ms + certifi==2024.8.30 + charset-normalizer==3.4.0 + idna==3.10 + requests==2.32.3 + urllib3==2.2.3 
```

> 我们强烈建议尽可能使用 `uv pip` 代替 `pip`，因为它能极大地缩短安装时间。

除了依赖隔离之外，环境还允许你使用不同版本的编程语言运行时。

```
$  --pythonUsing CPython 3.12.7 Creating virtual environment at: venv312 $  source && --versionPython 3.12.7 $  --pythonUsing CPython 3.11.10 Creating virtual environment at: venv311 $  source && --versionPython 3.11.10 
```

当你需要在多个 Python 版本上测试代码，或者某个项目要求特定版本时，这很有帮助。

> 在某些编程语言中，每个项目会自动获得自己专属的依赖环境，而无需你手动创建，但原理是相同的。如今大多数语言也都有在单个系统上管理多个语言版本的机制，并可为各个项目指定使用哪个版本。

# 构建产物与打包

在软件开发中，我们区分源代码和构建产物。开发者编写和阅读源代码，而构建产物是由源代码产生的打包好的、可分发的输出 —— 随时可以安装或部署。构建产物可以简单到一个我们要运行的代码文件，也可以复杂到包含应用程序所需全部零零碎碎的整个虚拟机。看这个例子，当前目录下有一个 Python 文件 `greet.py`：

```
$  cat def greet(name): print(f"Hello, {name}!") $  -c"from greet import greet; greet('World')"Hello, World! $  cd $  -c"from greet import greet; greet('World')"ModuleNotFoundError: No module named 'greet' 
```

一旦切换到其他目录，导入就会失败，因为 Python 只在特定位置搜索模块（当前目录、已安装的包，以及 `PYTHONPATH` 中的路径）。打包通过把代码安装到已知位置来解决这个问题。

在 Python 中，打包一个库意味着生成一种构建产物，供 `pip` 或 `uv` 这类包安装器用来安装相关文件。Python 的构建产物称为 *wheel*（wheel 包），包含安装一个包所需的全部信息：代码文件、关于该包的元数据（名称、版本、依赖），以及把文件放到环境中何处的说明。构建构建产物要求我们编写一个项目文件（也常称为清单，manifest），详细说明项目的具体情况、所需依赖、包的版本等信息。在 Python 中，我们用 `pyproject.toml` 来做这件事。

> `pyproject.toml` 是现代的、推荐的方式。虽然早期的打包方式如 `requirements.txt` 或 `setup.py` 仍受支持，但你应尽可能优先使用 `pyproject.toml`。

下面是一个同时提供命令行工具的库的最小化 `pyproject.toml`：

```
[project] name = "greeting" version ="0.1.0" description = "A simple greeting library" dependencies =["typer>=0.9"][project.scripts] greet ="greeting:cli"[build-system] requires =["setuptools>=61.0"]build-backend ="setuptools.build_meta"
```

`typer` 库是一个流行的 Python 包，能以极少的样板代码创建命令行界面。

以及对应的 `greeting.py`：

```
import typer def greet(name: str) -> None: print(f"Hello, {name}!") def cli() -> None: typer. run(greet) if __name__ == "__main__": cli()
```

有了这个文件，我们现在就可以构建 wheel 包：

```
$  Building source distribution... Building wheel from source distribution... Successfully built dist/greeting-0.1.0.tar.gz Successfully built dist/greeting-0.1.0-py3-none-any.whl $  ls greeting-0.1.0-py3-none-any.whl greeting-0.1.0.tar.gz 
```

`.whl` 文件就是 wheel 包（一种具有特定结构的 zip 归档），`.tar.gz` 是供需要从源代码构建的系统使用的源代码发行版（source distribution）。

你可以检查 wheel 包的内容，看看里面打包了什么：

```
$  -lArchive: dist/greeting-0.1.0-py3-none-any.whl Length Date Time Name --------- ---------- ----- ---- 150 2024-01-15 10:30 greeting.py 312 2024-01-15 10:30 greeting-0.1.0.dist-info/METADATA 92 2024-01-15 10:30 greeting-0.1.0.dist-info/WHEEL 9 2024-01-15 10:30 greeting-0.1.0.dist-info/top_level.txt 435 2024-01-15 10:30 greeting-0.1.0.dist-info/RECORD --------- ------- 998 5 files 
```

现在如果我们把这个 wheel 包交给别人，他们可以通过运行以下命令来安装它：

```
$  install $  Hello, Alice! 
```

这会把我们之前构建的库安装到他们的环境中，包括 `greet` 命令行工具。

这种方法有其局限性。特别是当我们的库依赖平台特定的库时（例如用于 GPU 加速的 CUDA），我们的构建产物只能在装有这些特定库的系统上工作，而且我们可能需要为不同平台（Linux、macOS、Windows）和架构（x86、ARM）分别构建 wheel 包。

安装软件时，从源代码安装和安装预编译二进制文件之间有一个重要区别。从源代码安装意味着下载原始代码并在你的机器上编译 —— 这要求安装编译器和构建工具，而且对大型项目来说可能要花费相当多的时间。

安装预编译二进制文件意味着下载别人已经编译好的构建产物 —— 更快也更简单，但二进制文件必须与你的平台和架构匹配。例如，[ripgrep 的发布页面](https://github.com/BurntSushi/ripgrep/releases) 提供了适用于 Linux（x86\_64、ARM）、macOS（Intel、Apple Silicon）和 Windows 的预编译二进制文件。

# 发布与版本管理

代码的构建是一个连续的过程，而发布则是离散的。在软件开发中，开发环境和生产环境有明确的区分。代码需要先在开发环境中被验证可用，然后才会被*发布*（ship）到生产环境。发布过程涉及许多步骤，包括测试、依赖管理、版本管理、配置、部署和公开发布。

软件库并非一成不变，而是随着时间推移不断演进，获得修复和新功能。我们通过离散的版本标识符来追踪这种演进，每个版本标识符对应库在某一时间点的状态。库行为的变化范围很广：从修复非关键功能的补丁，到扩展其功能的新特性，再到破坏向后兼容的变更。变更日志（changelog）记录一个版本带来了哪些变化 —— 这是软件开发者用来传达与新版本相关变更的文档。

然而，追踪每一个依赖的持续变化是不现实的，当我们把传递依赖 —— 也就是依赖的依赖 —— 也考虑进来时就更是如此。

> 你可以用 `uv tree` 可视化项目的整个依赖树，它以树状结构显示所有包及其传递依赖。

为了简化这个问题，人们约定了给软件打版本号的惯例，其中最流行的是[语义化版本](https://semver.org/)（Semantic Versioning，SemVer）。在语义化版本规范下，版本标识符的形式是 MAJOR.MINOR.PATCH，每个值都是整数。简单来说，升级时：

* PATCH（例如 1.2.3 → 1.2.4）应该只包含错误修复，并且完全向后兼容
* MINOR（例如 1.2.3 → 1.3.0）以向后兼容的方式添加新功能
* MAJOR（例如 1.2.3 → 2.0.0）表示可能需要修改代码的破坏性变更

> 这只是简化说法，我们建议阅读完整的 SemVer 规范，以理解比如为什么从 0.1.3 升到 0.2.0 可能会引入破坏性变更，或者 1.0.0-rc.1 是什么意思。Python 打包支持 [PEP 440](https://peps.python.org/pep-0440/) 定义的标准化版本标识符，因此在指定依赖版本时我们可以使用各种约束：

在 `pyproject.toml` 中，我们有不同的方式来约束依赖的兼容版本范围：

```
[project] dependencies =["requests= =2.32. 3", # Exact version - only this specific version  "click> =8.0", # Minimum version - 8.0 or newer  "numpy> =1.24,<2.0", # Range - at least 1.24 but less than 2.0  "pandas~ =2.1. 0", # Compatible release - >=2.1.0 and <2.2.0  ]
```

版本标识符在许多包管理器（npm、cargo 等）中都存在，具体语义各有不同。`~=` 运算符是 Python 的“兼容发布”（compatible release）运算符 —— `~=2.1.0` 表示“任何与 2.1.0 兼容的版本”，即 `>=2.1.0` 且 `<2.2.0`。这大致相当于 npm 和 cargo 中的插入符（`^`）运算符，后者遵循 SemVer 的兼容性概念。

并非所有软件都使用语义化版本。一种常见的替代方案是日历版本（Calendar Versioning，CalVer），其版本号基于发布日期而非语义含义。例如，Ubuntu 使用 `24.04`（2024 年 4 月）和 `24.10`（2024 年 10 月）这样的版本号。CalVer 让人一眼就能看出一个版本有多旧，但它不传达任何兼容性信息。最后，语义化版本也并非万无一失，维护者有时会在次版本或补丁版本中无意引入破坏性变更。

# 可复现性

在现代软件开发中，你写的代码建立在大量的抽象层之上。这包括编程语言运行时、第三方库、操作系统，甚至硬件本身。这些层中任何一层的任何差异都可能改变你代码的行为，甚至让它无法按预期工作。此外，即使底层硬件的差异也会影响你发布软件的能力。

锁定（pin）一个库指的是指定精确版本而非版本范围，例如用 `requests==2.32.3` 而不是 `requests>=2.0`。

包管理器工作的一部分是考虑依赖 —— 以及传递依赖 —— 提供的所有约束，然后产出一份能满足所有约束的有效版本清单。这份具体的版本清单随后可以保存到文件中以实现可复现性（reproducibility）；这类文件被称为 *锁文件*（lockfile）。

```
$  Resolved 12 packages in 45ms $  cat head -20version = 1 requires-python = "> = "  [[package]] name = "certifi" version = "2024.8.30" source = { registry = "https://pypi.org/simple" } sdist = { url = "https://files.pythonhosted.org/...", hash = "sha256:..." } wheels = [ { url = "https://files.pythonhosted.org/...", hash = "sha256:..." }, ] ... 
```

处理依赖版本管理和可复现性时，一个关键区别是库与应用程序/服务之间的差异。库旨在被其他代码导入和使用，而这些代码可能有自己的依赖，因此指定过于严格的版本约束可能与用户的其他依赖产生冲突。相比之下，应用程序或服务是软件的最终消费者，通常通过用户界面或 API 而非编程接口来暴露其功能。对库而言，良好的做法是指定版本范围，以最大化与更广泛包生态系统的兼容性。对应用程序而言，锁定精确版本可确保可复现性 —— 每个运行该应用程序的人使用的都是完全相同的依赖。

对于需要最大可复现性的项目，[Nix](https://nixos.org/) 和 [Bazel](https://bazel.build/) 等工具提供*密封*（hermetic）构建 —— 其中每个输入（包括编译器、系统库，甚至构建环境本身）都被锁定并以内容寻址。这保证了无论构建在何时何地运行，输出都是逐位一致的。

> 你甚至可以用 NixOS 来管理整台计算机的安装，这样你就能轻而易举地复制出新的计算机环境副本，并通过受版本控制的配置文件管理它们的完整配置。

软件开发中一个永无止境的矛盾是：新版本软件有意或无意地引入破坏性变更，而另一方面，旧版本软件随时间推移会因安全漏洞而变得不再安全。我们可以通过持续集成（CI）流水线来应对（我们将在[代码质量与 CI](/2026/code-quality/) 一讲中深入了解），用新版本的软件测试我们的应用程序，并部署自动化来检测依赖的新版本何时发布，例如 [Dependabot](https://github.com/dependabot)。

即使有了 CI 测试，升级软件版本时仍会出现问题，往往是因为开发环境与生产环境之间不可避免的不一致。在这种情况下，最好的办法是制定*回滚*（rollback）计划：撤销版本升级，转而重新部署一个已知可用的版本。

# 虚拟机与容器

当你开始依赖更复杂的依赖项时，你的代码的依赖很可能会超出包管理器所能处理的范围。一个常见的原因是需要与特定的系统库或硬件驱动交互。例如，在科学计算和 AI 中，程序常常需要专门的库和驱动才能使用 GPU 硬件。许多系统级依赖（GPU 驱动、特定版本的编译器、像 OpenSSL 这样的共享库）仍然需要系统范围的安装。

传统上，这个更广泛的依赖问题用虚拟机（Virtual Machine，VM）来解决。虚拟机抽象了整台计算机，提供带有自己专属操作系统的完全隔离环境。更现代的方法是容器（container），它把应用程序连同其依赖、库和文件系统一起打包，但与宿主操作系统共享内核，而不是虚拟化整台计算机。容器比虚拟机更轻量，因为它们共享内核，启动更快、运行更高效。

最流行的容器平台是 [Docker](https://www.docker.com/)。Docker 引入了构建、分发和运行容器的标准化方式。在底层，Docker 使用 containerd 作为其容器运行时 —— 这是一个行业标准，Kubernetes 等其他工具也在使用。

运行容器很简单。例如，要在容器内运行一个 Python 解释器，我们使用 `docker run`（`-it` 标志使容器带有交互式终端。当你退出时，容器就会停止。）。

```
$  -itPython 3.12.7 (main, Nov 5 2024, 02:53:25) [GCC 12.2.0] on linux >>>("Hello from inside a container!")Hello from inside a container! 
```

实际上你的程序可能依赖整个文件系统。为了解决这个问题，我们可以使用容器镜像（container image），它把应用程序的整个文件系统作为构建产物一起分发。容器镜像是以编程方式创建的。用 docker，我们通过 Dockerfile 语法精确指定镜像的依赖、系统库和配置：

```
FROM python:3.12 RUN RUN install -y RUN install -y RUN install RUN install COPY . /app WORKDIR /app RUN install
```

一个重要的区别：Docker **镜像**（image）是打包好的构建产物（就像模板），而**容器**是该镜像的运行实例。你可以用同一个镜像运行多个容器。镜像以分层方式构建，Dockerfile 中的每条指令（`FROM`、`RUN`、`COPY` 等）都会创建一个新层。Docker 会缓存这些层，所以如果你修改了 Dockerfile 中的某一行，只需重建该层及其后续的层。

上面的 Dockerfile 有几个问题：它使用完整版 Python 镜像而非 slim 变体，执行多条独立的 `RUN` 命令创建了不必要的层，版本没有锁定，而且它不清理包管理器缓存，把不必要的文件也随构建产物一起分发了。其他常见错误包括不安全地以 root 身份运行容器，以及意外地把密钥嵌入到层中。

下面是改进后的版本

```
FROM python:3.12-slim COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv RUN && \ install -y --no-install-recommends && \ rm -rf* WORKDIR /app ENV PATH="/app/.venv/bin:$PATH" COPY pyproject.toml uv.lock ./ RUN sync --locked --no-dev --no-install-project COPY . . RUN sync --locked --no-dev
```

在上面的例子中我们看到，我们不是从源代码安装 `uv`，而是从 `ghcr.io/astral-sh/uv:latest` 镜像复制预编译好的二进制文件。这被称为 *builder* 模式。采用这种模式，我们无需分发编译代码所需的所有工具，只需分发运行应用程序所需的最终二进制文件（这里是 `uv`）。

Docker 有一些需要注意的重要局限。首先，容器镜像往往是平台特定的 —— 为 `linux/amd64` 构建的镜像无法在 `linux/arm64`（Apple Silicon Mac）上原生运行，除非使用速度很慢的模拟（emulation）。其次，Docker 容器需要 Linux 内核，所以在 macOS 和 Windows 上，Docker 实际上在底层运行着一个轻量级 Linux 虚拟机，带来了额外开销。第三，Docker 的隔离性弱于虚拟机 —— 容器共享宿主内核，这在多租户环境中是个安全隐患。

> 如今，越来越多的项目也通过 [nix flakes](https://serokell.io/blog/practical-nix-flakes) 使用 nix 来按项目管理甚至“系统级”的库和应用程序。

# 配置

软件天生是可配置的。在[命令行环境](/2026/command-line-environment/) 一讲中我们看到，程序通过标志（flag）、环境变量，甚至配置文件（即 dotfile）来接收选项。这对更复杂的应用同样成立，而且已有成熟的规模化配置管理范式。软件配置不应嵌入代码中，而应在运行时提供。两种常见方式是环境变量和配置文件。

下面是一个通过环境变量配置的应用示例：

```
import os DATABASE_URL = os. environ. get("DATABASE_URL","sqlite:///local.db") DEBUG = os. environ. get("DEBUG", "false"). lower() == "true" API_KEY = os. environ["API_KEY"]# Required - will raise if not set 
```

应用也可以通过配置文件来配置（例如一个通过 `yaml.load` 加载配置的 Python 程序），`config.yaml`：

```
database: url: "postgresql://localhost/myapp" pool_size: 5 server: host: "0.0.0.0" port: 8080 debug: false
```

思考配置问题的一条好的经验法则是：同一份代码库只需更改配置（而非代码）就应该能部署到不同的环境（开发、预发布、生产）。

在众多配置项中，往往包含 API 密钥之类的敏感数据。密钥（secret）需要小心处理以避免意外泄露，而且绝不能纳入版本控制。

# 服务与编排

现代应用程序很少孤立存在。一个典型的 Web 应用可能需要用于持久化存储的数据库、用于提升性能的缓存、用于后台任务的消息队列，以及各种其他辅助服务。现代架构往往不是把所有东西打包进单个单体应用，而是把功能拆分成可以独立开发、部署和扩展的多个服务。

举个例子，如果我们判断应用可以从缓存中受益，与其自己造轮子，不如利用 [Redis](https://redis.io/) 或 [Memcached](https://memcached.org/) 这类久经考验的现成方案。我们可以通过把 Redis 作为容器的一部分来构建，将其嵌入应用的依赖中，但这意味着要统一 Redis 与我们应用之间的所有依赖，这可能很有挑战性甚至不可行。取而代之，我们可以把每个应用分别部署在各自的容器中。这就是通常所说的微服务（microservice）架构，其中每个组件作为独立服务运行，通过网络（通常是 HTTP API）进行通信。

[Docker Compose](https://docs.docker.com/compose/) 是一个用于定义和运行多容器应用的工具。你不必逐个管理容器，而是在一个 YAML 文件中声明所有服务并统一编排（orchestrate）它们。现在我们的完整应用包含不止一个容器：

```
# docker-compose.yml services: web: build:. ports: - "8080:8080" environment: -REDIS_URL=redis://cache:6379 depends_on: - cache cache: image:redis:7-alpine volumes: -redis_data:/data volumes: redis_data:
```

执行 `docker compose up`，两个服务会一起启动，Web 应用可以使用主机名 `cache` 连接 Redis（Docker 的内部 DNS 会自动解析服务名）。Docker Compose 让我们声明想如何部署一个或多个服务，并负责把它们一起启动、在其间建立网络、管理用于数据持久化的共享卷（volume）等编排工作。

对于生产部署，你通常希望 docker compose 服务在开机时自动启动，并在失败时自动重启。一种常见的做法是用 systemd 管理 docker compose 部署：

```
# /etc/systemd/system/myapp.service [Unit] Description = My Application Requires =docker.service After =docker.service[Service] Type = oneshot RemainAfterExit = yes WorkingDirectory =/opt/myapp ExecStart =/usr/bin/docker compose up -d ExecStop =/usr/bin/docker compose down[Install] WantedBy =multi-user.target
```

这个 systemd 单元（unit）文件确保你的应用在系统启动时（Docker 就绪之后）启动，并提供 `systemctl start myapp`、`systemctl stop myapp`、`systemctl status myapp` 等标准控制命令。

随着部署需求变得更加复杂 —— 需要跨多台机器的可扩展性、服务崩溃时的容错能力以及高可用性保证 —— 组织会转向 Kubernetes（k8s）这类成熟的容器编排平台，它们可以管理跨机器集群的成千上万个容器。不过，Kubernetes 学习曲线陡峭、运维开销显著，对较小的项目来说往往是杀鸡用牛刀。

这种多容器设置之所以在一定程度上可行，是因为现代服务通过标准化的 API（即 HTTP REST API）相互通信。例如，每当程序与 OpenAI 或 Anthropic 这类 LLM 提供商交互时，底层都是在向它们的服务器发送 HTTP 请求并解析响应：

```
$  \ -H "x-api-key: $ " \   -H "content-type: application/json" \ -H "anthropic-version: 2023-06-01" \ -d '{"model": "claude-sonnet-4-20250514", "max_tokens": 256, "messages": [{"role": "user", "content": "Explain containers vs VMs in one sentence."}]}' 
```

# 发布

一旦你的代码被证明可用，你可能就想把它分发出去供他人下载和安装。分发有许多形式，且与你所用的编程语言和环境有着内在的联系。

最简单的分发形式是上传构建产物，让人们下载并在本地安装。这种方式至今仍很常见，你可以在 [Ubuntu 的软件包归档](http://archive.ubuntu.com/ubuntu/pool/main/) 这样的地方找到它，它本质上是一个 `.deb` 文件的 HTTP 目录列表。

如今，GitHub 已成为发布源代码和构建产物的事实标准平台。虽然源代码通常是公开的，但 GitHub Releases 允许维护者把预编译二进制文件和其他构建产物附加到打了标签的版本上。

包管理器有时支持直接从 GitHub 安装，可以从源代码安装，也可以从预构建的 wheel 包安装：

```
#  source() $  install  #  $  install  #  $  install 
```

事实上，像 Go 这样的语言采用去中心化的分发模式 —— Go 模块不是通过中心化的包仓库，而是直接从其源代码仓库分发。像 `github.com/gorilla/mux` 这样的模块路径指明了代码存放的位置，`go get` 直接从那里获取。不过，大多数包管理器（如 `pip`、`cargo` 或 `brew`）都有预打包项目的中心化索引，以便于分发和安装。如果我们运行

```
$  install --verbose --no-cache grep -F'.whl'DEBUG Selecting: requests==2.32.5 [compatible] (requests-2.32.5-py3-none-any.whl) DEBUG No cache entry for: https://files.pythonhosted.org/packages/1e/db/4254e3eabe8020b458f1a747140d32277ec7a271daf1d235b70dc0b4e6e3/requests-2.32.5-py3-none-any.whl.metadata DEBUG No cache entry for: https://files.pythonhosted.org/packages/1e/db/4254e3eabe8020b458f1a747140d32277ec7a271daf1d235b70dc0b4e6e3/requests-2.32.5-py3-none-any.whl 
```

就能看到我们从哪里获取 `requests` 的 wheel 包。注意文件名中的 `py3-none-any` —— 这意味着该 wheel 包适用于任何 Python 3 版本、任何操作系统、任何架构。对于包含编译代码的包，wheel 包是平台特定的：

```
$  install --verbose --no-cache grep -F'.whl'DEBUG Selecting: numpy==2.2.1 [compatible] (numpy-2.2.1-cp312-cp312-macosx_14_0_arm64.whl) 
```

这里 `cp312-cp312-macosx_14_0_arm64` 表示这个 wheel 包专门用于 macOS 14 及以上、ARM64（Apple Silicon）平台上的 CPython 3.12。如果你在其他平台上，`pip` 会下载不同的 wheel 包或从源代码构建。

反过来，要让人们能够找到我们创建的包，我们需要把它发布到这些注册中心（registry）之一。在 Python 中，主要的注册中心是 [Python 包索引（PyPI）](https://pypi.org)。与安装一样，发布包也有多种方式。`uv publish` 命令提供了把包上传到 PyPI 的现代化接口：

```
$  --publish-urlPublishing greeting-0.1.0.tar.gz Publishing greeting-0.1.0-py3-none-any.whl 
```

这里我们使用的是 [TestPyPI](https://test.pypi.org) —— 一个独立的包注册中心，专门用于测试你的发布流程，而不会污染真正的 PyPI。上传之后，你可以从 TestPyPI 安装：

```
$  install --index-url
```

发布软件时的一个关键考量是信任。用户如何验证他们下载的包确实来自你、且没有被篡改？包注册中心使用校验和（checksum）来验证完整性，一些生态系统还支持包签名（package signing），以提供作者身份的密码学证明。

不同的语言有自己的包注册中心：Rust 用 [crates.io](https://crates.io)，JavaScript 用 [npm](https://www.npmjs.com)，Ruby 用 [RubyGems](https://rubygems.org)，容器镜像用 [Docker Hub](https://hub.docker.com)。与此同时，对于私有或内部的包，组织通常会部署自己的包仓库（如私有 PyPI 服务器或私有 Docker registry），或使用云提供商的托管方案。

把 Web 服务部署到互联网上还涉及额外的基础设施：域名注册、将域名指向你的服务器的 DNS 配置，以及通常还需要 nginx 这样的反向代理（reverse proxy）来处理 HTTPS 和路由流量。对于文档或静态站点这类更简单的用例，[GitHub Pages](https://pages.github.com/) 可以直接从仓库提供免费托管。

# 练习

1. 用 `printenv` 把你的环境保存到一个文件，创建一个 venv 并激活它，用 `printenv` 保存到另一个文件，然后 `diff before.txt after.txt`。环境中有什么变化？为什么 shell 优先使用 venv？（提示：查看激活前后的 `$PATH`。）运行 `which deactivate` 并推断 deactivate 这个 bash 函数做了什么。
2. 创建一个带有 `pyproject.toml` 的 Python 包，并在虚拟环境中安装它。创建一个锁文件并查看其内容。
3. 安装 Docker，并用它通过 docker compose 在本地构建 Missing Semester 课程网站。
4. 为一个简单的 Python 应用编写 Dockerfile。然后编写一个 `docker-compose.yml`，让你的应用与一个 Redis 缓存一起运行。
5. 把一个 Python 包发布到 TestPyPI（除非真的值得分享，否则不要发布到真正的 PyPI！）。然后用该包构建一个 Docker 镜像并推送到 `ghcr.io`。
6. 用 [GitHub Pages](https://docs.github.com/en/pages/quickstart) 搭建一个网站。附加（非）学分：为它配置自定义域名。

---

原文：https://missing.csail.mit.edu/2026/shipping-code/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
