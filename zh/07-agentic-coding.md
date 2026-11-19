# 编码智能体（Agentic Coding）

编码智能体（coding agent）是能够调用工具的对话式 AI 模型，可用的工具包括读写文件、网页搜索、执行 shell 命令等。它们或运行在 IDE 中，或以独立的命令行/GUI 工具形式存在。编码智能体是高度自主且功能强大的工具，能够支撑各种各样的使用场景。

本讲建立在[开发环境与工具](/2026/development-environment/)一讲中 AI 辅助开发相关内容的基础上。作为一个快速演示，让我们继续使用 [AI 辅助开发](/2026/development-environment/#ai-powered-development)小节中的例子：

```python
from urllib.request import urlopen

def download_contents(url: str) -> str:
    with urlopen(url) as response:
        return response.read().decode('utf-8')

def extract(content: str) -> list[str]:
    import re
    pattern = r'\[.*?\]\((.*?)\)'
    return re.findall(pattern, content)

print(extract(download_contents("https://raw.githubusercontent.com/missing-semester/missing-semester/refs/heads/master/_2026/development-environment.md")))
```

我们可以尝试用下面这个任务来提示（prompt）编码智能体：

```
Turn this into a proper command-line program, with argparse for argument parsing. Add type annotations, and make sure the program passes type checking.
```

智能体会先读取文件以理解它，然后做一些修改，最后调用类型检查器来确保类型注解是正确的。如果它犯了错导致类型检查不通过，它很可能会自行迭代修正——不过这个任务比较简单，这种情况不太可能发生。由于编码智能体能够使用可能造成危害的工具，默认情况下，智能体运行框架（agent harness）会在工具调用前提示用户确认。

> 如果编码智能体犯了错——例如，你的 `$PATH` 上直接就有 `mypy` 可执行文件，而智能体却尝试调用 `python -m mypy`——你可以给它文字反馈，帮助它纠正方向。

编码智能体支持多轮交互，因此你可以通过与智能体的来回对话不断迭代工作。如果智能体正在朝错误的方向走，你甚至可以打断它。一个有用的心智模型是把自己想成实习生的管理者：实习生会做那些琐碎的具体工作，但需要指导，而且偶尔会做错事、需要被纠正。

> 想要一个更有说服力的演示，可以作为后续请求让智能体运行生成的脚本。观察输出，然后试着让它做一个修改（例如，让它只保留绝对 URL）。

# AI 模型与智能体的工作原理

完整解释现代[大语言模型（LLM）](https://en.wikipedia.org/wiki/Large_language_model)的内部工作机制以及智能体运行框架这类基础设施超出了本课程的范围。不过，对其核心思想有一个高层次的理解，有助于有效地*使用*这项前沿技术并理解它的局限性。

可以将大语言模型（LLM）理解为对“给定提示词（prompt）字符串（输入）时，补全字符串（输出）的概率分布”建模。LLM 推理（比如你向对话式聊天应用提交查询时所发生的事情）就是从这个概率分布中*采样*。LLM 有固定的*上下文窗口（context window）*，即输入和输出字符串的最大长度。

对话式聊天和编码智能体这类 AI 工具就建立在这一基础之上。对于多轮交互，聊天应用和智能体使用轮次标记（turn marker），并在每次出现新的用户提示词时把整个对话历史作为提示词字符串传入，每次用户提示词触发一次 LLM 推理。对于支持工具调用（tool-calling）的智能体，运行框架会把某些 LLM 输出解释为调用工具的请求，并把工具调用的结果作为提示词字符串的一部分传回模型（因此每次发生工具调用/响应时都会再次运行 LLM 推理）。工具调用智能体的核心概念[可以用 200 行代码实现](https://www.mihaileric.com/The-Emperor-Has-No-Clothes/)。

## 隐私

大多数 AI 编码工具在默认配置下会把你的大量数据发送到云端。有时是运行框架在本地运行、而 LLM 推理在云端运行；有时甚至更多的软件都在云端运行（例如，服务提供商实际上可能获得你整个代码仓库的副本，以及你与 AI 工具的全部交互记录）。

也有一些开源的 AI 编码工具和开源 LLM 相当不错（尽管还没有专有模型那么好），但目前对大多数用户来说，由于硬件限制，在本地运行前沿的开源 LLM 是不现实的。

# 使用场景

编码智能体可以帮上忙的任务范围很广。一些例子：

* **实现新功能。** 像上面的例子一样，你可以让编码智能体实现一个功能。如何写好一份规格说明（specification），目前与其说是科学不如说是艺术：你给智能体的输入要足够描述清楚，让智能体做出你想要的东西（至少方向正确，以便你继续迭代），但又不能事无巨细到让自己承担了太多工作。测试驱动开发（test-driven development）可能特别有效：先写测试（或让编码智能体帮你写测试），审查测试以确保它们覆盖了你的需求，然后再让编码智能体实现该功能。模型在持续进步，因此你需要不断更新自己对模型能力的直觉。
  > 我们用 Claude Code [实现](https://github.com/missing-semester/missing-semester/pull/345)了这些 Tufte 风格的边注。
* **修复错误。** 如果你的编译器、linter、类型检查器或测试报了错，你可以让智能体来修复它们，例如用“fix the issues with mypy”这样的提示词。当你能让编码模型进入一个反馈循环时，它的效果会特别好，因此尽量把环境搭建好，让模型可以直接运行失败的检查，从而自主迭代。如果这不可行，你可以手动给模型反馈。
  > 在 missing-semester 仓库的提交 [f552b55](https://github.com/missing-semester/missing-semester/commit/f552b5523462b22b8893a8404d2110c4e59613dd) 上，我们用“Review the agentic coding lecture for typos and grammatical issues”提示 Claude Code，随后让它修复所发现的问题，这些修复被提交在 [f1e1c41](https://github.com/missing-semester/missing-semester/commit/f1e1c417adba6b4149f7eef91ff5624de40dc637) 中。
* **重构。** 你可以用编码智能体以各种方式重构代码，从简单的任务（比如重命名一个方法，这类重构也受[代码智能](/2026/development-environment/#code-intelligence-and-language-servers)支持）到更复杂的任务（比如把某块功能拆分到一个独立模块中）。
  > 我们用 Claude Code 把编码智能体内容[拆分](https://github.com/missing-semester/missing-semester/pull/344)成了独立的一讲。
* **代码审查。** 你可以让编码智能体审查代码。你可以给它基本的指示，比如“review my latest changes that are not yet committed”。如果你想审查一个 pull request，而你的编码智能体支持网页抓取，或者你安装了 [GitHub CLI](https://cli.github.com/) 这类命令行工具，你甚至可以直接让编码智能体“Review the pull request {链接}”，它会接手处理到底。
* **理解代码。** 你可以向编码智能体询问关于某个代码库的问题，这在新人上手（onboarding）时特别有用。
* **当作 shell 用。** 你可以让编码智能体使用某个特定工具来完成任务，从而用自然语言调用 shell 命令，比如“use the find command to find all files older than 30 days”或“use mogrify to resize all the jpgs to 50% of their original size”。
* **氛围编程（Vibe coding）。** 智能体已经强大到你可以不亲手写任何一行代码就实现一些应用。
  > [这里有一个例子](https://github.com/cleanlab/office-presence-dashboard)，是课程讲师之一用氛围编程做的一个真实项目。

# 进阶用法

这里我们简要介绍编码智能体一些更高级的使用模式和能力。

* **可复用提示词（reusable prompts）。** 创建可复用的提示词或模板。例如，你可以写一个详细的提示词，按某种特定方式做代码审查，并把它保存为可复用提示词。
  > 智能体工具演进很快。在某些工具中，可复用提示词作为独立功能已被弃用。例如在 Codex 和 Claude Code 中，它们已被[技能（skills）](https://code.claude.com/docs/en/skills)所[吸收](https://developers.openai.com/codex/custom-prompts)。
* **并行智能体。** 编码智能体可能很慢：你给出提示词后，它可能对着一个问题工作几十分钟。你可以同时运行多个智能体副本，既可以让它们做同一个任务（LLM 是随机采样的，所以把同一件事跑多次、取最好的方案可能有帮助），也可以做不同的任务（例如同时实现两个互不重叠的功能）。为了防止不同智能体的改动相互干扰，你可以使用 [git worktrees](https://git-scm.com/docs/git-worktree)，我们在[版本控制](/2026/version-control/)一讲中会讲到。
* **MCP。** MCP 是 *Model Context Protocol*（模型上下文协议）的缩写，是一个开放协议，你可以用它把编码智能体与各种工具连接起来。例如，这个 [Notion MCP server](https://github.com/makenotion/notion-mcp-server) 可以让你的智能体读写 Notion 文档，从而支持这样的用例：“读取 {Notion 文档} 中链接的规格说明，在 Notion 中起草一份实现计划作为新页面，然后实现一个原型”。要发现可用的 MCP，你可以使用 [Pulse](https://www.pulsemcp.com/servers) 和 [Glama](https://glama.ai/mcp/servers) 这类目录站。
* **上下文管理。** 正如我们[前文](#how-ai-models-and-agents-work)提到的，编码智能体所依赖的 LLM 有有限的*上下文窗口*。要有效地使用编码智能体，就必须善于利用上下文。你要确保智能体能获取到它需要的信息，同时避免不必要的上下文，以免撑爆上下文窗口或降低模型性能（随着上下文变大，即使没溢出上下文窗口，性能也往往会下降）。智能体运行框架会自动提供并在一定程度上管理上下文，但相当大的控制权留给了用户。
  + **清空上下文窗口。** 最基本的控制手段：编码智能体支持清空上下文窗口（即开启新对话），对于不相关的查询你应该这样做。
  + **回退对话。** 有些编码智能体支持撤销对话历史中的某些步骤。与其发一条后续消息把智能体引向别的方向，在“撤销”更合理的情况下，这样做能更有效地管理上下文。
  + **压缩（compaction）。** 为了支持任意长度的对话，编码智能体支持上下文*压缩*：如果对话历史变得太长，它们会自动调用一次 LLM 对对话的前缀部分做总结，并用该总结替换掉对话历史。有些智能体把控制权交给用户，让用户在需要时手动触发压缩。
  + **llms.txt。** `/llms.txt` 文件是一个提议中的[标准](https://llmstxt.org/)位置，用于存放供 LLM 在推理时使用的文档。产品（如 [cursor.com/llms.txt](https://cursor.com/llms.txt)）、软件库（如 [ai.pydantic.dev/llms.txt](https://ai.pydantic.dev/llms.txt)）和 API（如 [apify.com/llms.txt](https://apify.com/llms.txt)）都可能提供 `llms.txt` 文件，对开发很有用。这类文档的每 token 信息密度更高，因此比起让编码智能体抓取并阅读 HTML 页面，它们的上下文利用效率更高。当编码智能体对你想使用的某个依赖没有内置知识时（例如因为它发布于 LLM 知识截止日期之后），外部文档就派上用场了。
  + **AGENTS.md。** 大多数编码智能体都支持 [AGENTS.md](https://agents.md/) 或类似机制（例如 Claude Code 会寻找 `CLAUDE.md`），把它当作面向编码智能体的 README。智能体启动时，会把 `AGENTS.md` 的全部内容预填进上下文。你可以用它给智能体提供跨会话通用的建议（例如，指示它在修改代码后总是运行类型检查器、说明如何运行单元测试，或提供可供智能体浏览的第三方文档链接）。有些编码智能体可以自动生成这个文件（例如 Claude Code 中的 `/init` 命令）。可以在[这里](https://github.com/pydantic/pydantic-ai/blob/main/CLAUDE.md)看到一个真实的 `AGENTS.md` 示例。
  + **技能（skills）。** `AGENTS.md` 中的内容总是被完整地加载进智能体的上下文窗口。*技能*增加了一层间接性以避免上下文膨胀：你可以给智能体提供一份技能列表及其描述，智能体可以按需“打开”某个技能（将其加载进上下文窗口）。
  + **子智能体（subagents）。** 有些编码智能体允许你定义子智能体，即面向特定任务工作流的智能体。顶层编码智能体可以调用子智能体来完成某个特定任务，这使顶层智能体和子智能体都能更有效地管理上下文。顶层智能体的上下文不会被子智能体看到的一切所撑大，而子智能体也只拿到其任务所需的上下文。举个例子，有些编码智能体把网络调研实现为一个子智能体：顶层智能体向子智能体提出一个查询，子智能体运行网页搜索、抓取各个网页、进行分析，然后把查询的答案返回给顶层智能体。这样一来，顶层智能体的上下文不会被所抓取网页的全部内容撑大，而子智能体的上下文里也不会有顶层智能体其余的对话历史。

对于许多需要编写提示词的进阶功能（例如技能或子智能体），你可以借助 LLM 来起步。有些编码智能体甚至内置了对此的支持。例如，Claude Code 可以从一段简短的提示词生成一个子智能体（调用 `/agents` 并创建一个新智能体）。试着用下面这段提示词创建一个子智能体：

```
A Python code checking agent that uses `mypy` and `ruff` to type-check, lint, and format *check* any files that have been modified from the last git commit.
```

然后，你可以用顶层智能体通过类似“use the code checker subagent”的消息显式调用该子智能体。你也可能让顶层智能体在合适的时候自动调用子智能体，例如在修改了任何 Python 文件之后。

# 需要警惕的问题

AI 工具会犯错。它们建立在 LLM 之上，而 LLM 只是概率性的下一 token 预测模型。它们并不像人类那样“智能”。要审查 AI 输出的正确性和安全漏洞。有时验证代码比自己写代码还难；对于关键代码，考虑亲手编写。AI 会钻进牛角尖，甚至试图对你“gaslight”（让你怀疑自己的判断）；要警惕调试漩涡。不要把 AI 当拐杖，警惕过度依赖或浅尝辄止的理解。仍然有大类的编程任务是 AI 目前无法完成的。计算思维依然宝贵。

# 推荐软件

许多 IDE / AI 编码扩展都内置了编码智能体（推荐清单见[开发环境一讲](/2026/development-environment/)）。其他流行的编码智能体包括 Anthropic 的 [Claude Code](https://www.claude.com/product/claude-code)、OpenAI 的 [Codex](https://openai.com/codex/)，以及 [opencode](https://github.com/anomalyco/opencode) 这类开源智能体。

# 练习

1. 用手工编码、AI 自动补全、内联聊天（inline chat）和智能体这四种方式各做一遍同一个编程任务，比较这四种体验。最好的候选是你正在进行的项目中的一个小型功能。如果你在寻找其他点子，可以考虑完成 GitHub 上开源项目的 “good first issue” 类任务，或者 [Advent of Code](https://adventofcode.com/)、[LeetCode](https://leetcode.com/) 的题目。
2. 用 AI 编码智能体去探索一个陌生的代码库。这最好结合一个你真正关心的项目——你想调试它或给它加一个新功能。如果你一时想不出这样的项目，试着用 AI 智能体去理解 [opencode](https://github.com/anomalyco/opencode) 智能体中与安全相关的功能是如何工作的。
3. 用氛围编程从零做一个小应用。不要亲手写任何一行代码。
4. 对于你选用的编码智能体，创建并测试一个 `AGENTS.md`（或你所选智能体的类似机制，如 `CLAUDE.md`）、一个技能（skill，例如 [Claude Code 中的技能](https://code.claude.com/docs/en/skills)或 [Codex 中的技能](https://developers.openai.com/codex/skills/)），以及一个子智能体（subagent，例如 [Claude Code 中的子智能体](https://code.claude.com/docs/en/sub-agents)）。思考一下什么时候该用其中哪一个而不是另一个。注意，你选用的编码智能体可能不支持其中某些功能；你可以跳过它们，或者换用另一个支持这些功能的编码智能体。
5. 用编码智能体完成与[代码质量一讲](/2026/code-quality/)中 Markdown 项目符号正则练习相同的目标。它是通过直接编辑文件来完成任务的吗？让智能体直接编辑文件来完成这类任务有哪些缺点和局限？想办法设计提示词，使智能体不通过直接编辑文件来完成任务。提示：让智能体使用[第一讲](/2026/course-shell/)中提到的某个命令行工具。
6. 大多数编码智能体都支持某种形式的“yolo 模式”（例如 Claude Code 中的 `--dangerously-skip-permissions`）。直接使用这种模式是不安全的，但在虚拟机或容器这样的隔离环境中运行编码智能体、然后再启用自主运行，或许是可接受的。在你的机器上把这套环境跑起来。[Claude Code devcontainers](https://code.claude.com/docs/en/devcontainer) 或 [Docker Sandboxes / Claude Code](https://docs.docker.com/ai/sandboxes/agents/claude-code/) 这类文档可能会派上用场。搭建这套环境的方法不止一种。

---

原文：https://missing.csail.mit.edu/2026/agentic-coding/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
