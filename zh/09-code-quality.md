# 代码质量

有各种各样的工具和技术可以帮助开发者编写高质量的代码。在本讲中，我们将涵盖：

* [格式化](#formatting)
* [代码检查](#linting)
* [测试](#testing)
* [提交前钩子](#pre-commit-hooks)
* [持续集成](#continuous-integration)
* [命令运行器](#command-runners)

作为一个附加主题，我们还将介绍[正则表达式](#regular-expressions)，这是一个贯穿多个领域的主题，既在代码质量方面有应用（例如，运行与某个模式匹配的测试子集），也在其他领域有应用，比如 IDE（例如，查找与替换）。

这些工具大多是特定于语言的（例如，用于 Python 的 [Ruff](https://docs.astral.sh/ruff/) 代码检查器(linter)/格式化工具）。在某些情况下，工具会支持多种语言（例如，[Prettier](https://prettier.io/) 代码格式化工具）。不过，这些概念几乎是通用的——任何编程语言都能找到代码格式化工具、代码检查器、测试库等等。

# 格式化

代码自动格式化工具（auto-formatter）会自动美化表层语法。这样，你就可以把精力集中在更深层、更具挑战性的问题上，而让自动格式化工具处理诸如字符串使用 `'` 还是 `"` 语法的一致性、二元运算符两侧加上空格（用 `x + y` 而不是 `x+y`）、`import` 语句按顺序排列，以及避免行过长这类琐碎细节。代码格式化工具的一大好处是，它们能为所有在某个代码库上工作的开发者统一代码风格。

像 Prettier 这样的工具[高度可配置](https://prettier.io/docs/configuration)；你应该把配置文件纳入项目的[版本控制](/2026/version-control/)。其他工具，如 [Black](https://github.com/psf/black) 和 [gofmt](https://pkg.go.dev/cmd/gofmt)，可配置性有限或完全不可配置，以减少[在无关紧要的细节上争论不休（bikeshedding）](https://en.wikipedia.org/wiki/Law_of_triviality)。

你可以为代码格式化工具配置 [IDE 集成](/2026/development-environment/#code-intelligence-and-language-servers)，这样你的代码就会在你输入时或保存文件时被自动格式化。你还可以在项目中添加 [EditorConfig](https://editorconfig.org/) 文件，它会向你的 IDE 传达某些项目级设置，比如每种文件类型的缩进宽度。

# 代码检查（Linting）

代码检查器（linter）运行静态分析（在不运行代码的情况下分析代码），以发现代码中的反模式和潜在问题。这些工具比自动格式化工具更深入，不只看表层语法。分析的深度因工具而异。

代码检查器自带一系列*规则*，并提供可在项目层面配置的预设。有些检查规则会产生误报，因此你可以按文件或按行禁用它们。

好的代码检查器会内置帮助或文档，解释每条检查规则——这条规则在检查什么、为什么这样做不好，以及对于该代码模式更好的替代方案是什么。例如，参见 [Ruff](https://docs.astral.sh/ruff/) 中 [SIM102](https://docs.astral.sh/ruff/rules/collapsible-if/) 规则的文档，这条规则用于捕获 Python 代码中不必要的嵌套 `if` 语句。

有些代码检查器不仅能标记问题，还能自动为你修复某些问题。

除了特定于语言的代码检查器之外，另一个可能派上用场的工具是 [semgrep](https://github.com/semgrep/semgrep)，它是一个“语义 grep”工具，工作在 AST（抽象语法树）层面（而不是像 grep 那样工作在字符层面），并且支持多种语言。你可以用 semgrep 轻松地为你的项目编写自定义检查规则。例如，如果你想阻止 Python 中危险的 `subprocess.Popen(..., shell=True)`，可以用下面的方式找到该代码模式：

```bash
semgrep -l python -e "subprocess.Popen(..., shell=True, ...)"
```

# 测试

软件测试是提升你对代码正确性信心的标准手段。你编写代码，然后编写用于运行你所写代码的代码，如果代码不按预期工作就抛出错误。

你可以针对不同粒度的代码块编写测试：针对单个函数的*单元测试*，针对模块或服务之间交互的*集成测试*，以及针对端到端场景的*功能测试*。你可以采用*测试驱动开发*，即在编写任何实现代码之前先写测试。当你在代码中发现 bug 时，可以编写*回归测试*，这样如果该功能将来被破坏，你就能捕获到它。你还可以编写*基于属性的测试*，这种方法由 Haskell 中的 [QuickCheck](https://hackage.haskell.org/package/QuickCheck) 开创，并已在许多库中实现，比如 Python 的 [Hypothesis](https://hypothesis.readthedocs.io/)。哪种测试方法合适取决于你的项目；通常，你会采用某种组合。

如果你的程序有数据库或 Web API 之类的外部依赖，在测试中*模拟（mock）*这些依赖可能会有所帮助，而不是让你的代码在测试时与第三方依赖进行交互。

## 代码覆盖率

代码覆盖率（code coverage）是一个用来衡量你的测试有多好的指标。代码覆盖率关注的是运行测试时你代码中的哪些行被执行了，因此你可以确保自己覆盖了所有代码路径。代码覆盖率工具可以逐行显示覆盖率，指导你编写测试。像 [Codecov](https://app.codecov.io) 这样的服务提供了 Web 界面，用于跟踪和查看项目历史中的代码覆盖率。

和任何指标一样，代码覆盖率并不完美；不要过分执着于覆盖率，而要专注于编写高质量的测试。

# 提交前钩子

Git 提交前[钩子](https://git-scm.com/book/en/v2/Customizing-Git-Git-Hooks)（pre-commit hooks）在 [pre-commit](https://pre-commit.com/) 框架的帮助下变得更容易使用，它会在每次 Git 提交之前自动运行用户指定的代码。项目通常使用提交前钩子在每次提交前自动运行格式化工具和代码检查器，有时还包括测试，以确保提交的代码符合项目代码风格且不存在某些问题。

# 持续集成（CI）

持续集成（continuous integration，CI）服务，如 [GitHub Actions](https://github.com/features/actions)，可以在你每次推送代码时（或在每个拉取请求上，或按计划）为你运行脚本。开发者常用 CI 服务来运行代码质量工具，包括格式化工具、代码检查器和测试。对于编译型语言，你可以确保代码能够编译；对于静态类型语言，你可以确保它通过类型检查。在每次推送新提交时运行 CI，可以捕获被引入代码主版本的错误；在拉取请求上运行，可以捕获贡献者提交中的问题；按计划运行，可以捕获外部依赖的问题（例如，某位开发者意外地以[符合语义化版本](/2026/shipping-code/#releases--versioning)的方式发布了一个破坏性变更）。

由于 CI 脚本运行在开发者的机器之外，你可以轻松地在那里运行耗时很长的任务。例如，可以利用这一点在不同操作系统和编程语言版本上运行一个测试*矩阵*，以确保软件在所有这些环境中都能正常工作。

一般来说，在 CI 中运行的脚本不会直接修改你的代码：它会以“仅检查”模式而不是“修复”模式运行工具，例如，当代码不符合格式要求时，自动格式化工具会报错。

代码仓库经常在其 README 中包含[状态徽章](https://docs.github.com/en/actions/how-tos/monitor-workflows/add-a-status-badge)，显示 CI 状态以及代码覆盖率等其他信息。例如，下面是 Missing Semester 当前的构建状态。

[![Build Status](https://github.com/missing-semester/missing-semester/actions/workflows/build.yml/badge.svg)](https://github.com/missing-semester/missing-semester/actions/workflows/build.yml) [![Links Status](https://github.com/missing-semester/missing-semester/actions/workflows/links.yml/badge.svg)](https://github.com/missing-semester/missing-semester/actions/workflows/links.yml)

> 我们的[链接检查器](https://github.com/missing-semester/missing-semester/blob/master/.github/workflows/links.yml)使用了 [proof-html](https://github.com/anishathalye/proof-html) 这个 GitHub Action，它经常失败，通常是由于第三方网站的问题。尽管如此，它帮助我们发现并修复了许多失效链接（有时是由于拼写错误，更多时候是由于网站移动了内容却没有添加重定向，或者网站直接消失了）。

学习 CI 服务、格式化工具、代码检查器和测试库的具体细节，一个好方法是通过范例来学习。在 GitHub 上找一些高质量的开源项目——在编程语言、领域、规模和范围等方面与你的项目越相似越好——研究它们的 `pyproject.toml`、`.github/workflows/`、`DEVELOPMENT.md` 以及其他相关文件。

## 持续部署

持续部署利用 CI 基础设施来实际*部署*变更。例如，Missing Semester 的仓库使用持续部署到 GitHub Pages，这样每当我们 `git push` 更新讲义时，网站就会自动构建并部署。你可以在 CI 中构建其他类型的[构建产物](/2026/shipping-code/)，例如应用程序的二进制文件或服务的 Docker 镜像。

# 命令运行器

像 [just](https://github.com/casey/just) 这样的命令运行器（command runner）简化了在项目上下文中运行命令这项任务。当你为项目搭建代码质量基础设施时，你不会希望让你的开发者死记 `uv run ruff check --fix` 这样的命令。有了命令运行器，它可以变成 `just lint`，而且你可以有类似的调用，如 `just format`、`just typecheck` 等等，涵盖开发者可能想为你的项目运行的所有不同工具。

某些特定于语言的项目或包管理器内置了对这类功能的支持，这意味着你不需要使用 `just` 这样的语言无关工具。例如，[npm](https://nodejs.org/en/learn/getting-started/an-introduction-to-the-npm-package-manager)（Node.js）的 `package.json` 中的 `scripts` 部分，以及 [Hatch](https://hatch.pypa.io/)（Python）的 `pyproject.toml` 中的 `tool.hatch.envs.*.scripts` 部分，都支持这一功能。

# 正则表达式

*正则表达式*（regular expression，常缩写为 “regex”）是一种用于表示字符串集合的语言。正则模式常用于命令行工具和 IDE 等各种场景中的模式匹配。例如，[ag](https://github.com/ggreer/the_silver_searcher) 支持用正则模式在整个代码库中搜索（例如，`ag "import .* as .*"` 会找出 Python 中所有重命名的导入），而 [go test](https://pkg.go.dev/cmd/go#hdr-Test_packages) 支持 `-run [regexp]` 选项来选择测试子集。此外，编程语言都有对正则表达式匹配的内置支持或第三方库，因此你可以把正则用于模式匹配、验证和解析等功能。

为了帮助建立直觉，下面给出一些正则模式的示例。在本讲中，我们使用 [Python 正则语法](https://docs.python.org/3/library/re.html)。正则表达式有许多不同的“方言”，彼此之间略有差异，尤其是在更复杂的功能上。你可以使用像 [regex101](https://regex101.com/) 这样的在线正则测试器来开发和调试正则表达式。

* `abc` —— 匹配字面量 “abc”。
* `missing|semester` —— 匹配字符串 “missing” 或字符串 “semester”。
* `\d{4}-\d{2}-\d{2}` —— 匹配 YYYY-MM-DD 格式的日期，例如 “2026-01-14”。除了确保该字符串由四个数字、一个连字符、两个数字、一个连字符和两个数字组成之外，它并不验证日期本身，所以 “2026-01-99” 也能匹配这个正则模式。
* `.+@.+` —— 匹配电子邮件地址，即包含一些文本、接着一个 “@”、然后再包含一些文本的字符串。这只是最基本的验证，会匹配像 “nonsense@@@email” 这样的字符串。一个没有任何误报和漏报的匹配电子邮件地址的正则表达式是[存在的](https://pdw.ex-parrot.com/Mail-RFC822-Address.html)，但并不实用。

## 正则语法

你可以在[这份文档](https://docs.python.org/3/library/re.html#regular-expression-syntax)（或网上众多其他资源之一）中找到正则语法的全面指南。下面是一些基本构件：

* `abc` 匹配字面字符串，当这些字符没有特殊含义时（在这个例子中是 “abc”）
* `.` 匹配任意单个字符
* `[abc]` 匹配包含在方括号内的单个字符（在这个例子中是 “a”、“b” 或 “c”）
* `[^abc]` 匹配除方括号内字符之外的任意单个字符（例如 “d”）
* `[a-f]` 匹配包含在方括号所示范围内的单个字符（例如 “c”，但不是 “q”）
* `a|b` 匹配两个模式中的任意一个（例如 “a” 或 “b”）
* `\d` 匹配任意数字字符（例如 “3”）
* `\w` 匹配任意单词字符（例如 “x”）
* `\b` 匹配任意单词*边界*（例如，在字符串 “missing semester” 中，匹配 “m” 之前、“g” 之后、“s” 之前以及 “r” 之后的位置）
* `(...)` 匹配一个模式分组
* `...?` 匹配零个或一个模式，例如 `words?` 可匹配 “word” 或 “words”
* `...*` 匹配任意数量的模式，例如 `.*` 可匹配任意数量的任意字符
* `...+` 匹配一个或多个模式，例如 `\d+` 可匹配任意非零位数的数字
* `...{N}` 匹配恰好 N 个模式，例如 `\d{4}` 匹配 4 个数字
* `\.` 匹配字面量 “.”
* `\\` 匹配字面量 “\”
* `^` 匹配行首
* `$` 匹配行尾

## 捕获组与引用

如果你使用正则分组 `(...)`，就可以引用匹配中的子部分，用于提取或查找替换。例如，要从 YYYY-MM-DD 风格的日期中只提取月份，可以使用以下 Python 代码：

```python
>>> import re
>>> re.match(r"\d{4}-(\d{2})-\d{2}", "2026-01-14").group(1)
'01'
```

在文本编辑器中，你可以在替换模式里引用捕获组。语法在不同 IDE 之间可能有所不同。例如，在 VS Code 中，你可以使用 `$1`、`$2` 等变量；在 Vim 中，你可以使用 `\1`、`\2` 等来引用分组。

## 局限性

[正则语言](https://en.wikipedia.org/wiki/Regular_language)功能强大但有其局限；有些字符串集合是无法用标准正则表达式表示的（例如，[不可能](https://en.wikipedia.org/wiki/Pumping_lemma_for_regular_languages)写出一个能匹配字符串集合 {a^n b^n | n ≥ 0}（即由若干个 “a” 后跟相同数量的 “b” 组成的字符串的集合）的正则表达式；更实际地说，像 HTML 这样的语言不是正则语言）。在实践中，现代正则引擎支持前瞻（lookahead）和反向引用（backreference）等特性，把支持范围扩展到了正则语言之外，它们在实践中极其有用，但重要的是要明白它们的表达能力仍然是有限的。对于更复杂的语言，你可能需要求助于能力更强的解析器（一个例子是 [pyparsing](https://github.com/pyparsing/pyparsing)，一个 [PEG](https://en.wikipedia.org/wiki/Parsing_expression_grammar) 解析器）。

## 学习正则表达式

我们建议先学习基础知识（即本讲涵盖的内容），然后在需要时查阅正则参考文档，而不是去背下这门语言的全部内容。

对话式 AI 工具可以有效地帮助你生成正则模式。例如，试着用下面的提示词询问你最喜欢的 LLM：

```
Write a Python-style regex pattern that matches the requested path from log lines from Nginx. Here is an example log line:

169.254.1.1 - - [09/Jan/2026:21:28:51 +0000] "GET /feed.xml HTTP/2.0" 200 2995 "-" "python-requests/2.32.3"
```

# 练习

1. 为你正在做的某个项目配置格式化工具、代码检查器和提交前钩子。如果有很多错误：自动格式化应该能解决格式错误。对于代码检查器报出的错误，试着用 [AI 智能体](/2026/agentic-coding/)来修复所有 linter 错误。确保 AI 智能体能够运行代码检查器并观察结果，这样它就能以迭代循环的方式修复所有问题。仔细检查结果，确保 AI 没有破坏你的代码！
2. 为你熟悉的某种语言学习一个测试库，并为你正在做的项目编写一个单元测试。运行代码覆盖率工具，生成 HTML 格式的覆盖率报告，并观察结果。你能找到被覆盖的那些行吗？你的代码覆盖率很可能非常低。试着手动编写一些测试来提高它。试着用 [AI 智能体](/2026/agentic-coding/)来提高覆盖率；确保编码智能体能够运行带覆盖率统计的测试并生成逐行覆盖率报告，这样它才知道该重点关注哪里。AI 生成的测试真的好吗？
3. 为你正在做的项目设置持续集成，在每次推送时运行。让 CI 运行格式化、代码检查和测试。故意破坏你的代码（例如，引入一个 linter 违规），并确保 CI 能捕获它。
4. 试着编写一个[正则模式](#regular-expressions)，并使用 `grep` [命令行工具](/2026/course-shell/)在你的代码中查找 `subprocess.Popen(..., shell=True)` 的出现。现在，试着“击溃”这个正则模式。[semgrep](#linting) 是否仍然能成功匹配到那段让你的 grep 调用失效的危险代码？
5. 在你的 IDE 或文本编辑器中练习正则查找替换：在[这份讲义](https://raw.githubusercontent.com/missing-semester/missing-semester/refs/heads/master/_2026/code-quality.md)中把 `-` [Markdown 无序列表标记](https://spec.commonmark.org/0.31.2/#bullet-list-marker)替换为 `*` 列表标记。注意，把文件中所有的 “-” 字符都替换掉是不对的，因为该字符有很多并非列表标记的用法。
6. 编写一个正则，从形如 `{"name": "Alyssa P. Hacker", "college": "MIT"}` 的 JSON 结构中捕获姓名（例如本例中的 `Alyssa P. Hacker`）。提示：在你的第一次尝试中，你可能会写出一个提取出 `Alyssa P. Hacker", "college": "MIT` 的正则；阅读 [Python 正则文档](https://docs.python.org/3/library/re.html)中关于贪婪量词（greedy quantifier）的内容，弄清楚该如何修正它。
   1. 让这个正则模式在姓名中含有 `"` 字符的情况下也能工作（双引号在 JSON 中可以用 `\"` 转义）。
   2. 在实践中，我们**不**建议用正则表达式来解决复杂的解析问题。想办法用你的编程语言的 JSON 解析器来完成这个任务。编写一个命令行程序，从 stdin 读入上述形式的 JSON 结构，并将姓名输出到 stdout。你只需要几行代码就能做到这一点。在 Python 中，除了 `import json` 之外，一行代码即可轻松完成。

---

原文：https://missing.csail.mit.edu/2026/code-quality/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
