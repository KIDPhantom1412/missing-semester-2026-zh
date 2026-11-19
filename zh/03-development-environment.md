# 开发环境与工具

*开发环境*（development environment）是开发软件的一整套工具。开发环境的核心是文本编辑功能，以及语法高亮、类型检查、代码格式化、自动补全（autocomplete）等配套功能。*集成开发环境*（IDE，Integrated Development Environment）如 [VS Code](https://code.visualstudio.com/) 将所有这些功能整合到一个应用程序中。基于终端的开发工作流则组合使用 [tmux](https://github.com/tmux/tmux)（终端复用器）、[Vim](https://www.vim.org/)（文本编辑器）、[Zsh](https://www.zsh.org/)（shell）以及特定语言的命令行工具，如 [Ruff](https://docs.astral.sh/ruff/)（Python linter 和代码格式化器）和 [Mypy](https://mypy-lang.org/)（Python 类型检查器）。

IDE 和基于终端的工作流各有优缺点。例如，图形化 IDE 更容易上手，如今的 IDE 通常有更好的开箱即用的 AI 集成，如 AI 自动补全；另一方面，基于终端的工作流非常轻量，在没有 GUI 或无法安装软件的环境中，它可能是你唯一的选择。我们建议你对两者都有基本的了解，并至少精通其中之一。如果你还没有偏好的 IDE，我们建议从 [VS Code](https://code.visualstudio.com/) 开始。

在本讲中，我们将涵盖：

* [文本编辑与 Vim](#text-editing-and-vim)
* [代码智能与语言服务器](#code-intelligence-and-language-servers)
* [AI 驱动的开发](#ai-powered-development)
* [扩展与其他 IDE 功能](#extensions-and-other-ide-functionality)

# 文本编辑与 Vim

编程时，你的大部分时间花在在代码中导航、阅读代码片段和修改代码上，而不是长篇书写或从头到尾通读文件。[Vim](https://www.vim.org/) 是一款针对这种任务分布而优化的文本编辑器。

**Vim 的理念。** Vim 的根基是一个优美的理念：它的界面本身就是一门编程语言，专为导航和编辑文本而设计。按键（带有助记的名称）就是命令，而且这些命令可以组合。Vim 避免使用鼠标，因为它太慢；Vim 甚至避免使用方向键，因为它需要的动作太多。结果就是：一个用起来如同脑机接口、与你思考速度相匹配的编辑器。

**其他软件对 Vim 的支持。** 你不必亲自使用 [Vim](https://www.vim.org/) 也能受益于其核心思想。许多涉及文本编辑的程序都支持"Vim 模式"（Vim mode），或作为内置功能，或作为插件（plugin）。例如，VS Code 有 [VSCodeVim](https://marketplace.visualstudio.com/items?itemName=vscodevim.vim) 插件，Zsh [内置支持](https://zsh.sourceforge.io/Guide/zshguide04.html)Vim 模拟，甚至 Claude Code 也[内置支持](https://code.claude.com/docs/en/interactive-mode#vim-editor-mode)Vim 编辑器模式。你使用的任何涉及文本编辑的工具，都很可能以某种方式支持 Vim 模式。

## 模态编辑

Vim 是一款*模态编辑器*（modal editor）：针对不同类型的任务，它有不同的操作模式。

* **普通模式**（Normal）：在文件中移动并进行编辑
* **插入模式**（Insert）：插入文本
* **替换模式**（Replace）：替换文本
* **可视模式**（Visual，普通、行或块）：选择文本块
* **命令行模式**（Command-line）：运行命令

在不同的操作模式下，按键有不同的含义。例如，字母 `x` 在插入模式下只是插入一个字面字符 "x"，但在普通模式下，它会删除光标下的字符，而在可视模式下，它会删除选中的内容。

在默认配置下，Vim 会在左下角显示当前模式。初始/默认模式是普通模式。你的大部分时间通常在普通模式和插入模式之间度过。

按下 `<ESC>`（Esc 键）可以从任何模式切换回普通模式。在普通模式下，用 `i` 进入插入模式，`R` 进入替换模式，`v` 进入可视模式，`V` 进入可视行模式，`<C-v>`（即 Ctrl-V，有时也写作 `^V`）进入可视块模式，用 `:` 进入命令行模式。

使用 Vim 时你会频繁用到 `<ESC>` 键：可以考虑把 Caps Lock 重映射为 Escape（[macOS 教程](https://vim.fandom.com/wiki/Map_caps_lock_to_escape_in_macOS)），或者用一个简单的按键序列为 `<ESC>` 创建一个[替代映射](https://vim.fandom.com/wiki/Avoid_the_escape_key#Mappings)。

## 基础：插入文本

在普通模式下，按 `i` 进入插入模式。此时，Vim 的行为和其他任何文本编辑器一样，直到你按下 `<ESC>` 返回普通模式。这连同上面介绍的基础知识，就是你开始用 Vim 编辑文件所需的全部（不过如果你所有的时间都待在插入模式里编辑，效率并不会太高）。

## Vim 的界面是一门编程语言

Vim 的界面是一门编程语言。按键（带有助记的名称）就是命令，而且这些命令可以*组合*。这使得高效的移动和编辑成为可能，尤其当这些命令成为肌肉记忆之后——就像你熟悉键盘布局之后，打字也会变得非常高效一样。

### 移动

你应该把大部分时间花在普通模式下，用移动命令在文件中导航。Vim 中的移动也被称为"名词"（nouns），因为它们指向一块块文本。

* 基本移动：`hjkl`（左、下、上、右）
* 单词：`w`（下一个单词），`b`（单词开头），`e`（单词结尾）
* 行：`0`（行首），`^`（第一个非空白字符），`$`（行尾）
* 屏幕：`H`（屏幕顶部），`M`（屏幕中部），`L`（屏幕底部）
* 滚动：`Ctrl-u`（向上），`Ctrl-d`（向下）
* 文件：`gg`（文件开头），`G`（文件结尾）
* 行号：`:{number}<CR>` 或 `{number}G`（第 {number} 行）
  + `<CR>` 指回车 / Enter 键
* 杂项：`%`（匹配项，如圆括号或花括号）
* 查找：`f{character}`、`t{character}`、`F{character}`、`T{character}`
  + 在当前行内向前/向后查找（find）或跳至（to）{character}
  + 用 `,` / `;` 在各个匹配之间导航
* 搜索：`/{regex}`，用 `n` / `N` 在各个匹配之间导航

### 选择

可视模式：

* 可视：`v`
* 可视行：`V`
* 可视块：`Ctrl-v`

可以配合移动键来做出选择。

### 编辑

过去你用鼠标做的每件事，现在都可以用键盘完成：使用可与移动命令组合的编辑命令。Vim 的界面从这里开始像一门编程语言了。Vim 的编辑命令也被称为"动词"（verbs），因为动词作用于名词。

* `i` 进入插入模式
  + 但要操作/删除文本，光靠退格键是不够的
* `o` / `O` 在下方 / 上方插入新行
* `d{motion}` 删除 {motion}
  + 例如 `dw` 是删除单词，`d$` 是删除到行尾，`d0` 是删除到行首
* `c{motion}` 修改 {motion}
  + 例如 `cw` 是修改单词
  + 相当于 `d{motion}` 后接 `i`
* `x` 删除字符（等价于 `dl`）
* `s` 替换字符（等价于 `cl`）
* 可视模式 + 操作
  + 选中文本后，按 `d` 删除或按 `c` 修改
* `u` 撤销，`<C-r>` 重做
* `y` 复制 / "yank"（其他一些命令如 `d` 也会复制）
* `p` 粘贴
* 还有很多可学的：例如 `~` 翻转字符的大小写，`J` 合并行

### 计数

你可以把名词和动词与一个计数组合起来，让指定动作执行相应的次数。

* `3w` 向前移动 3 个单词
* `5j` 向下移动 5 行
* `7dw` 删除 7 个单词

### 修饰符

你可以用修饰符来改变名词的含义。修饰符包括 `i`，意思是"内部"（inner/inside），以及 `a`，意思是"周围"（around）。

* `ci(` 修改当前一对圆括号内的内容
* `ci[` 修改当前一对方括号内的内容
* `da'` 删除一个单引号字符串，包括两侧的单引号

## 融会贯通

下面是一个有 bug 的 [fizz buzz](https://en.wikipedia.org/wiki/Fizz_buzz) 实现：

```python
def fizz_buzz(limit):
    for i in range(limit):
        if i % 3 == 0:
            print("fizz", end="")
        if i % 5 == 0:
            print("fizz", end="")
        if i % 3 and i % 5:
            print(i, end="")
        print()

def main():
    fizz_buzz(20)
```

我们从普通模式开始，用以下命令序列来修复这些问题：

* main 从未被调用
  + `G` 跳到文件末尾
  + `o` 在下方**打开**（**o**pen）一个新行
  + 输入 `if __name__ == "__main__": main()`
    - 如果你的编辑器支持 Python 语言，它可能会在插入模式下为你做一些自动缩进
  + `<ESC>` 回到普通模式
* 起始是 0 而不是 1
  + 先按 `/`，接着输入 `range` 并按 `<CR>` 来搜索 "range"
  + `ww` 向前移动两个**单词**（**w**ord）（你也可以用 `2w`，但实际上对于小的计数，重复按同一个键比使用计数功能更常见）
  + `i` 切换到**插入**（**i**nsert）模式，并输入 `1,`
  + `<ESC>` 回到普通模式
  + `e` 跳到下一个单词的**结尾**（**e**nd）
  + `a` 开始**追加**（**a**ppend）文本，并输入 `+ 1`
  + `<ESC>` 回到普通模式
* 对 5 的倍数打印 "fizz"
  + `:6<CR>` 跳到第 6 行
  + `ci"` **修改**（**c**hange）**引号内**（**i**nside）'**"**' 的内容，改成 `"buzz"`
  + `<ESC>` 回到普通模式

## 学习 Vim

学习 Vim 的最佳方式是先掌握基础（也就是我们目前讲过的这些），然后在你所有的软件中启用 Vim 模式，在实践中开始使用它。抵制使用鼠标或方向键的诱惑；在某些编辑器中，你可以解绑方向键，强迫自己养成好习惯。

### 更多资源

* 本课程上一期的 [Vim 讲座](/2020/editors/)——我们在那里更深入地讲解了 Vim
* `vimtutor` 是随 Vim 一起安装的教程——如果安装了 Vim，你应该能在 shell 中运行 `vimtutor`
* [Vim Adventures](https://vim-adventures.com/) 是一个学习 Vim 的游戏
* [Vim Tips Wiki](https://vim.fandom.com/wiki/Vim_Tips_Wiki)
* [Vim Advent Calendar](https://vimways.org/2019/) 收录了各种 Vim 技巧
* [VimGolf](https://www.vimgolf.com/) 是[代码高尔夫](https://en.wikipedia.org/wiki/Code_golf)（code golf），只不过编程语言是 Vim 的 UI
* [Vi/Vim Stack Exchange](https://vi.stackexchange.com/)
* [Vim Screencasts](http://vimcasts.org/)
* [Practical Vim](https://pragprog.com/titles/dnvim2/)（书）

# 代码智能与语言服务器

IDE 通常通过 IDE 扩展提供需要理解代码语义的特定语言支持，这些扩展连接到实现了[语言服务器协议](https://microsoft.github.io/language-server-protocol/)（Language Server Protocol，LSP）的*语言服务器*（language server）。例如，[VS Code 的 Python 扩展](https://marketplace.visualstudio.com/items?itemName=ms-python.python)依赖 [Pylance](https://marketplace.visualstudio.com/items?itemName=ms-python.vscode-pylance)，[VS Code 的 Go 扩展](https://marketplace.visualstudio.com/items?itemName=golang.go)依赖第一方的 [gopls](https://go.dev/gopls/)。通过为你使用的语言安装相应的扩展和语言服务器，你可以在 IDE 中启用许多特定语言的功能，例如：

* **代码补全。** 更好的自动补全和自动建议，例如在输入 `object.` 之后能看到对象的字段和方法。
* **内联文档。** 在悬停和自动建议时看到文档。
* **跳转到定义。** 从使用处跳转到定义，例如从字段引用 `object.field` 跳转到该字段的定义。
* **查找引用。** 与上面相反，找到某个特定项目（如字段或类型）被引用的所有位置。
* **导入辅助。** 整理导入、移除未使用的导入、标记缺失的导入。
* **代码质量。** 这些工具可以独立使用，但这类功能通常也由语言服务器提供。代码格式化会自动缩进并自动格式化代码，类型检查器和 linter 会在你输入时发现代码中的错误。我们将在[代码质量](/2026/code-quality/)一讲中更深入地介绍这类功能。

## 配置语言服务器

对某些语言来说，你只需安装扩展和语言服务器就万事大吉。对另一些语言来说，要从语言服务器获得最大收益，你需要把你的环境告诉 IDE。例如，将 VS Code 指向你的 [Python 环境](https://code.visualstudio.com/docs/python/environments)，语言服务器就能看到你已安装的软件包。环境在我们的[打包与发布代码](/2026/shipping-code/)一讲中有更深入的介绍。

视语言而定，语言服务器可能有一些可供你配置的设置。例如，使用 VS Code 的 Python 支持时，你可以为不使用 Python 可选类型注解的项目关闭静态类型检查。

# AI 驱动的开发

自 2021 年中 [GitHub Copilot](https://github.com/features/copilot/ai-code-editor) 使用 OpenAI 的 [Codex 模型](https://openai.com/index/openai-codex/) 以来，[大语言模型](https://en.wikipedia.org/wiki/Large_language_model)（LLM）在软件工程中得到了广泛采用。目前主要有三种使用形态：自动补全、内联对话和编程智能体（coding agent）。

## 自动补全

AI 驱动的自动补全与 IDE 中传统自动补全的形态相同：在你输入时于光标位置给出补全建议。有时它被当作"开箱即用"的被动功能来使用。除此之外，AI 自动补全通常借助代码注释来给出[提示](https://en.wikipedia.org/wiki/Prompt_engineering)（prompt）。

例如，我们要写一个脚本来下载这份讲义的内容并提取其中所有链接。可以这样开始：

```python
import requests

def download_contents(url: str) -> str:
```

模型会补全函数体：

```python
    response = requests.get(url)
    return response.text
```

我们可以用注释进一步引导补全。例如，如果我们要写一个提取所有 Markdown 链接的函数，但函数名并没有特别强的描述性：

```python
def extract(contents: str) -> list[str]:
```

模型会补全出类似这样的内容：

```python
    lines = contents.splitlines()
    return [line for line in lines if line.strip()]
```

我们可以通过代码注释来引导补全：

```python
def extract(content: str) -> list[str]:
    # extract all Markdown links from the content
```

这一次，模型给出了更好的补全：

```python
    import re
    pattern = r'\[.*?\]\((.*?)\)'
    return re.findall(pattern, content)
```

从这里我们看到这类 AI 编程工具的一个缺点：它只能在光标处提供补全。在这个例子中，更好的做法是把 `import re` 放在模块层级，而不是放在函数内部。

上面的例子用了一个命名不佳的函数来演示如何用注释引导代码补全；在实践中，你应该把函数命名得更具描述性，比如 `extract_links`，而且应该编写文档字符串（docstring）（基于这些，模型应当能生成与上面类似的补全）。

出于演示目的，我们可以补全这个脚本：

```python
print(extract(download_contents("https://raw.githubusercontent.com/missing-semester/missing-semester/refs/heads/master/_2026/development-environment.md")))
```

## 内联对话

内联对话让你选中一行或一段代码，然后直接提示 AI 模型提出一处编辑。在这种交互模式下，模型可以修改已有的代码（这与自动补全不同，自动补全只能补全光标之后的代码）。

继续上面的例子，假设我们决定不再使用第三方的 `requests` 库。我们可以选中相关的三行代码，调用内联对话，然后输入类似这样的内容：

```
use built-in libraries instead
```

模型提出：

```python
from urllib.request import urlopen

def download_contents(url: str) -> str:
    with urlopen(url) as response:
        return response.read().decode('utf-8')
```

## 编程智能体

编程智能体将在[智能体编程](/2026/agentic-coding/)一讲中深入介绍。

## 推荐软件

一些流行的 AI IDE 包括装有 [GitHub Copilot](https://github.com/features/copilot/ai-code-editor) 扩展的 [VS Code](https://code.visualstudio.com/) 和 [Cursor](https://cursor.com/)。GitHub Copilot 目前[面向学生](https://github.com/education/students)、教师以及流行开源项目的维护者免费提供。这是一个发展迅速的领域。许多领先产品的功能大体相当。

# 扩展与其他 IDE 功能

IDE 是强大的工具，而*扩展*（extension）让它变得更加强大。我们无法在一讲中涵盖所有这些功能，这里只为大家指出几个流行的扩展。我们鼓励你自行探索这个领域；网上有很多流行 IDE 扩展的清单，比如面向 Vim 插件的 [Vim Awesome](https://vimawesome.com/)，以及[按热度排序的 VS Code 扩展](https://marketplace.visualstudio.com/search?target=VSCode&category=All%20categories&sortBy=Installs)。

* [开发容器](https://containers.dev/)（Development containers）：受主流 IDE 支持（例如，[VS Code 支持](https://code.visualstudio.com/docs/devcontainers/containers)），开发容器让你可以用容器来运行开发工具。这对可移植性或隔离性很有帮助。[打包与发布代码](/2026/shipping-code/)一讲会更深入地介绍容器。
* 远程开发：通过 SSH 在远程机器上进行开发（例如，使用 [VS Code 的 Remote SSH 插件](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh)）。这在某些场景下很方便，比如你想在云端一台强劲的 GPU 机器上开发并运行代码。
* 协同编辑：像 Google 文档那样编辑同一个文件（例如，使用 [VS Code 的 Live Share 插件](https://marketplace.visualstudio.com/items?itemName=MS-vsliveshare.vsliveshare)）。

# 练习

1. 在你所有支持 Vim 模式的软件（比如你的编辑器和 shell）中启用 Vim 模式，接下来一个月里的所有文本编辑都使用 Vim 模式。每当某件事看起来效率低下，或者当你心想"一定有更好的做法"时，试着 Google 一下，很可能确实有更好的做法。
2. 完成一个 [VimGolf](https://www.vimgolf.com/) 上的挑战。
3. 为你正在做的一个项目配置 IDE 扩展和语言服务器。确保所有预期的功能都能正常工作，例如对库依赖的跳转到定义。如果你没有可以用于这个练习的代码，可以使用 GitHub 上的某个开源项目（比如[这个](https://github.com/spf13/cobra)）。
4. 浏览一份 IDE 扩展清单，安装一个你觉得有用的扩展。

---

原文：https://missing.csail.mit.edu/2026/development-environment/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
