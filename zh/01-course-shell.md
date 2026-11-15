# 课程概述与 Shell 入门

# 我们是谁？

这门课由 [Anish](https://anish.io/)、[Jon](https://thesquareplanet.com/) 和 [Jose](http://josejg.com/) 共同授课。我们都是 MIT 的毕业生，这门 MIT IAP 课程是我们在学生时代开办起来的。你可以通过 [missing-semester@mit.edu](mailto:missing-semester@mit.edu) 联系到我们全体。

我们教授这门课没有任何报酬，也绝不以任何方式从中牟利。我们把所有[课程资料](https://missing.csail.mit.edu/)和[课程录像](https://www.youtube.com/@MissingSemester)都免费放在网上。如果你想支持我们的工作，最好的方式就是向更多人推荐这门课。如果你所在的公司、大学或其他组织面向较大规模的学员使用了这些内容，请把经验报告/反馈发邮件给我们，让我们也能听到你们的故事 :)

# 动机

作为计算机科学家，我们深知计算机非常擅长辅助完成重复性的任务。然而，我们常常忘记的是：这一点不仅适用于我们让程序执行的*计算*，同样适用于我们*使用*计算机的过程。在解决任何与计算机相关的问题时，我们手边都有海量的工具可用，它们能让我们更高效，能解决更复杂的问题。但我们中的许多人只用到了其中很小的一部分；我们只是死记硬背了一些勉强够用的神秘咒语，遇到困难时就从网上盲目复制粘贴命令。

这门课就是为了[解决这个问题](/about/)。

我们希望教给你如何用好已经掌握的工具，向你展示可以加入工具箱的新工具，并希望激发你自行探索（甚至构建）更多工具的兴趣。这就是我们认为大多数计算机科学课程体系中缺失的那个学期。

# 课程安排

这门不计学分的课程包含九次时长各一小时的讲座，每次围绕[某个特定主题](/2026/)展开。各次讲座基本相互独立，不过随着学期推进，我们会假定你已经熟悉前面讲座的内容。我们把讲义放在网上，但课上讲授的某些内容（例如现场演示）可能不会写进讲义。与往年一样，我们会录制讲座并把录像发布到[网上](https://www.youtube.com/@MissingSemester)。

我们试图在短短几次一小时的讲座里覆盖非常多的内容，所以课程信息密度相当高。为了让你能按自己的节奏熟悉这些内容，每次讲座都配有一组练习，带你走一遍该讲的要点。我们不会安排专门的答疑时间，但欢迎你在 [OSSU Discord](https://ossu.dev/#community) 的 `#missing-semester-forum` 频道提问，或发邮件到 [missing-semester@mit.edu](mailto:missing-semester@mit.edu)。

由于时间有限，我们无法像一门完整课程那样详尽地讲解每一个工具。在可能的情况下，我们会指向一些资源，方便你进一步钻研某个工具或主题；但如果有什么特别吸引你，也千万别犹豫，直接来找我们要进一步的指引！

最后，如果你对这门课有任何反馈，请发邮件到 [missing-semester@mit.edu](mailto:missing-semester@mit.edu) 告诉我们。

# 主题 1：Shell

## 什么是 Shell？

如今的计算机有各种各样的接口来向它们下达指令：花哨的图形用户界面、语音界面、AR/VR，以及最近兴起的大语言模型（LLM）。这些对于 80% 的使用场景来说都很棒，但它们在允许你做什么方面往往存在根本性的限制——你无法按下一个不存在的按钮，也无法发出一条没有被编程进去的语音指令。要充分发挥计算机所提供工具的作用，我们就得回到“老派”的方式，下沉到文本界面：Shell（命令行解释器）。

你所能接触到的几乎所有平台都有这样或那样形式的 Shell，其中许多还提供了多种 Shell 供你选择。它们在细节上可能有所不同，但核心大体一致：让你能够运行程序、向程序提供输入，并以一种半结构化的方式查看程序的输出。

要打开一个 Shell *提示符*（prompt，也就是你可以输入命令的地方），你首先需要一个*终端*（terminal），它是 Shell 的可视化界面。你的设备很可能出厂就自带了一个，或者你也可以很方便地装一个：

* **Linux：** 按 `Ctrl + Alt + T`（在大多数发行版上都有效）。或者在应用程序菜单中搜索“Terminal”。
* **Windows：** 按 `Win + R`，输入 `cmd` 或 `powershell`，然后按回车。也可以在开始菜单中搜索“Terminal”或“Command Prompt”。
* **macOS：** 按 `Cmd + Space` 打开聚焦搜索（Spotlight），输入“Terminal”，然后按回车。也可以在“应用程序 → 实用工具 → 终端”中找到它。

在 Linux 和 macOS 上，打开的通常是 Bourne Again SHell，简称 “bash”。这是使用最广泛的 Shell 之一，其语法与你在许多其他 Shell 中看到的相似。在 Windows 上，迎接你的将是 “batch” 或 “powershell” Shell，具体取决于你运行的是哪条命令。这些是 Windows 特有的，并不是本门课的重点，尽管我们讲的大部分内容在其中都有对应物。你真正应该使用的是 [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/)，或者一台 Linux 虚拟机。

还存在其他 Shell，它们往往在易用性上比 bash 有许多改进（fish 和 zsh 是最常见的两种）。尽管它们非常流行（所有授课老师都在用其中之一），但它们远没有 bash 那样无处不在，而且依赖许多相同的概念，所以我们在这一讲中不会展开讲它们。

## 为什么要关心它？

Shell 不只是（通常）比“到处点击”快得多，它还具备一种表达能力，是你在任何一个图形化程序中都难以找到的。我们接下来会看到，Shell 让你能够以富有创意的方式*组合*各种程序，从而实现几乎任何任务的自动化。

熟悉 Shell 对你在开源软件的世界里闯荡也非常有用（开源软件的安装说明常常离不开 Shell），此外它还能帮助你为软件项目搭建持续集成（如[《代码质量》一讲](/2026/code-quality/)中所述），并在其他程序出错时调试问题。

## 在 Shell 中导航

当你启动终端时，你会看到一个*提示符*，它通常长这样：

```
missing:~$  
```

这是与 Shell 交互的主文本界面。它告诉你当前所在的机器是 `missing`，而你的“当前工作目录”（current working directory，也就是你此刻所处的位置）是 `~`（“home” 的缩写）。`$` 表示你不是 root 用户（稍后会详述）。在这个提示符处，你可以输入一条*命令*，随后由 Shell 来解释执行。最基本的命令就是运行一个程序：

```
missing:~$  dateFri 10 Jan 2020 11:49:31 AM EST missing:~$  
```

这里我们运行了 `date` 程序，它（也许并不意外）会打印当前的日期和时间。接着 Shell 会请你输入下一条命令。我们也可以在执行命令时附带*参数*（arguments）：

```
missing:~$  echo hello 
```

在这个例子中，我们让 Shell 以参数 `hello` 运行程序 `echo`。`echo` 程序的作用很简单，就是把它的参数打印出来。Shell 通过按空白字符分割来解析命令，然后运行第一个词所指代的程序，并把之后的每个词都作为程序可访问的参数传给它。如果你想提供包含空格或其他特殊字符的参数（例如名为 “My Photos” 的目录），可以用 `'` 或 `"` 把参数括起来（`"My Photos"`），或者只用 `\` 转义相关的字符（`My\ Photos`）。

你刚开始学习时，最重要的命令也许是 `man`，它是 “manual”（手册）的缩写。`man` 程序能让你查询系统中任何命令的更多信息（此外还有别的功能）。例如，运行 `man date`，它会解释 `date` 是什么，以及你可以传给它、用来改变其行为的各种参数。通常你也可以给大多数命令传 `--help` 作为参数，来获取简短版的帮助信息。

> 建议在 `man` 之外安装并使用 [`tldr`](https://tldr.sh/)，因为它会直接在终端里给你展示常见的用法示例。大语言模型通常也非常擅长解释命令的工作原理，以及如何调用它们来实现你想要的功能。

在 `man` 之后，最需要学习的命令是 `cd`，即 “change directory”（切换目录）。这条命令实际上是内置于 Shell 之中的，并不是一个独立的程序（也就是说，`which cd` 会回答 “no cd found”）。你给它传一个路径（path），该路径就会成为你的当前工作目录。你还会看到工作目录反映在 Shell 提示符中：

```
missing:~$  cdmissing:/bin$  cdmissing:/$  cdmissing:~$  
```

> 注意，Shell 自带自动补全功能，所以通常只需按 ! 键就能更快地补全路径。

许多命令在没有特别指定时都会作用于当前工作目录。如果你不确定自己身在何处，可以运行 `pwd` 或打印 `$PWD` 环境变量（用 `echo $PWD`），两者都会给出当前工作目录。

当前工作目录还有一个便利之处：它让我们可以使用*相对*路径（relative paths）。到目前为止我们见过的路径都是*绝对*路径（absolute paths）——它们以 `/` 开头，给出从文件系统根目录（`/`）出发到达某个位置所需的完整目录序列。在实际使用中，你更常用的会是相对路径；之所以叫相对路径，是因为它们是相对于当前工作目录而言的。在相对路径（任何*不*以 `/` 开头的路径）中，第一个路径组成部分会在当前工作目录下查找，后续组成部分则照常逐级深入。例如：

```
missing:~$  cdmissing:/$  cd missing:/bin$  
```

每个目录中还存在两个“特殊”的组成部分：`.` 和 `..`。`.` 表示“当前目录”，`..` 表示“上一级目录”（父目录）。所以：

```
missing:~$  cdmissing:/$  cd missing:/$  
```

对于任何命令的参数，你通常都可以互换地使用绝对路径和相对路径；只是在使用相对路径时，务必清楚自己的当前工作目录是什么！

> 建议安装并使用 [`zoxide`](https://github.com/ajeetdsouza/zoxide) 来加速你的 `cd` 操作——`z` 会记住你经常访问的路径，让你用更少的输入就能到达。

## Shell 里有什么可用的？

但是 Shell 是怎么知道去哪里找 `date` 或 `echo` 这样的程序的呢？当 Shell 被要求执行一条命令时，它会查阅一个名为 `$PATH` 的*环境变量*（environment variable），其中列出了 Shell 在接到命令时应当搜索程序的目录：

```
missing:~$  echo $PATH/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin missing:~$  echo/bin/echo missing:~$  $PATH/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin 
```

当我们运行 `echo` 命令时，Shell 知道它应该执行程序 `echo`，于是沿着 `$PATH` 中以 `:` 分隔的目录列表逐个查找同名文件。找到之后就运行它（前提是该文件是*可执行*的；稍后详述）。我们可以用 `which` 程序来查明给定程序名实际执行的是哪个文件。我们也可以完全绕过 `$PATH`，直接给出想要执行的文件的*路径*。

这也给了我们一条线索，让我们可以确定在 Shell 中能够执行的*所有*程序：列出 `$PATH` 中所有目录的内容即可。我们可以把某个目录路径传给列出文件的 `ls` 程序来做到这一点：

```
missing:~$  ls
```

> 建议安装并使用 [`eza`](https://eza.rocks/)，它是一个对人更友好的 `ls`。

在大多数计算机上，这会打印出*非常非常多*的程序，不过这里我们只介绍其中最重要的一些。先看几个简单的：

* `cat file`，打印 `file` 的内容。
* `sort file`，按排序后的顺序打印 `file` 的各行。
* `uniq file`，去除 `file` 中连续的重复行。
* `head file` 和 `tail file`，分别打印 `file` 开头和结尾的若干行。

> 建议用 [`bat`](https://github.com/sharkdp/bat) 代替 `cat`，它支持语法高亮和滚动浏览。

还有 `grep pattern file`，它在 `file` 中查找匹配 `pattern` 的行。这个命令值得多花一点篇幅，因为它*非常*有用，而且功能比人们预想的要丰富得多。`pattern` 其实是一个*正则表达式*（regular expression），可以表达非常复杂的模式——我们会在[代码质量一讲中讲解](/2026/code-quality/#regular-expressions)正则表达式。你也可以指定一个目录而不是文件（或者干脆省略，默认为 `.`），并传入 `-r` 来递归搜索目录下的所有文件。

> 建议用 [`ripgrep`](https://github.com/BurntSushi/ripgrep) 代替 `grep`，它是更快、对人更友好的选择（但可移植性较差）。`ripgrep` 默认还会递归搜索当前工作目录！

还有一些接口略微复杂但非常有用的工具。其中首推 `sed`，它是一个可编程的文件编辑器。它有自己的编程语言，用于对文件做自动化修改，但它最常见的用法是：

```
missing:~$  sed -i's/pattern/replacement/g'
```

这会把 `file` 中所有 `pattern` 出现的地方替换为 `replacement`。`-i` 表示我们希望就地（inline）完成替换（而不是保持 `file` 不变、只把替换后的内容打印出来）。`s/` 是 sed 编程语言中表示我们要做一次替换（substitution）的写法。`/` 用来分隔模式和替换文本。结尾的 `/g` 表示我们要替换每一行上的*所有*匹配，而不只是第一个。和 `grep` 一样，这里的 `pattern` 是正则表达式，这给了你很强的表达能力。正则表达式替换还允许 `replacement` 回引用（back-reference）匹配到的模式中的某些部分；我们马上就会看到一个例子。

接下来是 `find`，它让你（递归地）查找满足某些条件的文件。例如：

```
missing:~$  -type -name"*.zip" -mtime
```

在下载目录中查找超过 30 天未修改的 ZIP 文件。

```
missing:~$  -type -size -exec ls -lh{}\\;
```

在你的主目录中查找大于 100M 的文件并列出它们。注意 `-exec` 接受一个以单独的 `;` 结尾的*命令*（这个 `;` 需要转义，和空格的情况很像），其中的 `{}` 会被 `find` 替换为每个匹配到的文件路径。

```
missing:~$  . -name"*.py" -exec grep -l "TODO"{}\\;
```

查找其中含有 TODO 事项的所有 `.py` 文件。

`find` 的语法可能有点令人生畏，但希望这些例子能让你体会到它有多有用！

> 建议用 [`fd`](https://github.com/sharkdp/fd) 代替 `find`，它能带来对人更友好的体验（但可移植性较差！）。

下一个登场的是 `awk`，它和 `sed` 一样拥有自己的编程语言。`sed` 是为编辑文件而生的，而 `awk` 是为解析文件而生的。`awk` 最常见的用途，是处理语法规整的数据文件（比如 CSV 文件），从每条记录（即每一行）中只抽取你需要的某些部分：

```
missing:~$  awk'{print $2}'
```

打印 `file` 每一行中以空白分隔的第二列。如果加上 `-F,`，它就会打印每行中以逗号分隔的第二列。`awk` 还能做多得多的事情——过滤行、计算聚合值等等——可以看看练习，尝尝鲜。

把这些工具组合起来，我们就能玩出这样的花样：

```
missing:~$  'journalctl -u sshd -b-1 | grep "Disconnected from"' \\ | sed -E 's/.*Disconnected from .* user (.*) [^ ]+ port.*/\\1/' \\ | sort | uniq -c \\ | sort -nk1,1 | tail -n10 \\  | awk '{print $}' | paste -sd, postgres,mysql,oracle,dell,ubuntu,inspur,test,admin,user,root 
```

这条命令从一台远程服务器抓取 SSH 日志（关于 `ssh` 我们会在下一讲详细讨论），搜索断开连接的消息，从每条这样的消息中提取用户名，然后以逗号分隔打印出出现次数最多的前 10 个用户名。全都在一条命令里完成！把每一步拆解开留作练习。

## Shell 语言（bash）

上面的例子引入了一个新概念：管道（pipe，记作 `|`）。它让你可以把一个程序的输出和另一个程序的输入串联起来。这之所以可行，是因为大多数命令行程序在没有给出 `file` 参数时，会对自己的“标准输入”（standard input，也就是你敲键盘的输入通常进入的地方）进行操作。`|` 把 `|` 之前那个程序的“标准输出”（standard output，也就是通常打印到终端上的内容）变成 `|` 之后那个程序的标准输入。这让你能够*组合*出各种 Shell 程序，也正是 Shell 成为如此高效的工作环境的原因之一！

事实上，大多数 Shell（比如 bash）都实现了完整的编程语言，就像 Python 或 Ruby 一样。它有变量、条件语句、循环和函数。你在 Shell 里运行命令时，其实就是在写一小段由 Shell 解释执行的代码。今天我们不会教你 bash 的全部内容，但有一些东西你会发现特别有用：

首先是重定向（redirect）：`>file` 让你把程序的标准输出写入 `file`，而不是打到终端上。这便于事后分析。`>>file` 会追加到 `file` 末尾而不是覆盖它。此外还有 `告诉 Shell 从文件而不是你的键盘读取内容，作为某个程序的标准输入。`

> 这里正好提一下 `tee` 程序。`tee` 会把标准输入打印到标准输出（就像 `cat` 一样！），但*同时*还会把它写入一个文件。所以 `verbose cmd | tee verbose.log | grep CRITICAL` 既能把完整的详细日志保存到文件，又能保持终端清爽！

接下来是条件语句：`if command1; then command2; command3; fi` 会执行 `command1`，如果没有引发错误，就接着运行 `command2` 和 `command3`。你也可以根据需要加上 `else` 分支。最常用作 `command1` 的是 `test` 命令，它常被简写为 `[`，让你能求值诸如“文件是否存在”（`test -f file` / `[ -f file ]`）或“一个字符串是否等于另一个”（`[ "$var" = "string" ]`）之类的条件。在 bash 中还有 `[[ ]]`，它是 `test` 的一个“更安全”的内置版本，在引号处理方面的怪异行为更少。

Bash 还有两种形式的循环：`while` 和 `for`。`while command1; do command2; command3; done` 的工作方式与对应的 `if` 命令完全一样，只不过只要 `command1` 不出错，它就会一遍又一遍地重新执行整个循环体。`for varname in a b c d; do command; done` 会执行 `command` 四次，每次把 `$varname` 分别设为 `a`、`b`、`c`、`d` 其中之一。除了显式列出这些项，你更常用的会是“命令替换”（command substitution），比如：

```
for in$(seq ); do
```

这会执行命令 `seq 1 10`（它打印从 1 到 10 的所有整数），然后用该命令的输出替换整个 `$()`，从而得到一个迭代 10 次的 for 循环。在较老的代码里，你有时会看到用字面的反引号（比如 `for i in `seq 1 10`; do`）来代替 `$()`，但你应该强烈倾向于使用 `$()` 形式，因为它可以嵌套。

虽然你*可以*直接在提示符处写很长的 Shell 脚本，但你通常会想把它们写进一个 `.sh` 文件。例如，下面这个脚本会循环运行某个程序直到它失败为止，只打印失败那一次的输出，同时在后台让 CPU 保持满负荷（比如用来复现不稳定的测试就很有用）：

```
#!/bin/bash set -euo # Start CPU stress in background --cpu STRESS_PID =$! # Setup log file LOGFILE = "test_runs_$(date).log" echo "Logging to $LOGFILE " # Run tests until one fails RUN = while test > " $LOGFILE "; do echo "Run $RUN  passed"(()) done # Cleanup and report kill $STRESS_PID echo "Test failed on run $RUN " echo"Last 20 lines of output:" tail -n " $LOGFILE " echo"Full log: $LOGFILE "
```

这里面有不少新东西，我建议你花点时间仔细研究，因为它们在编写有用的 Shell 命令时非常实用，比如用后台任务（`&`）并发运行程序、更巧妙的 [Shell 重定向](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)，以及[算术展开](https://www.gnu.org/software/bash/manual/html_node/Arithmetic-Expansion.html)。

不过，程序的前两行值得单独花点时间讲讲。第一行是 “shebang”（释伴）——你在 Shell 脚本之外的其他文件顶部也会看到它。当一个以 `#!/path` 这个神奇咒语开头的文件被执行时，Shell 会启动 `/path` 处的程序，并把该文件的内容作为输入传给它。对 Shell 脚本来说，这意味着把脚本内容传给 `/bin/bash`；但你也可以写出 shebang 行为 `/usr/bin/python` 的 Python 脚本！

第二行是让 bash 变得“更严格”的一种方式，可以规避写 Shell 脚本时的许多坑。`set` 能接受很多参数，简单来说：`-e` 使得任何命令失败时脚本都会提前退出；`-u` 使得使用未定义变量会直接让脚本崩溃，而不是当作空字符串；`-o pipefail` 使得 `|` 串联的程序中如果有哪个失败，整个 Shell 脚本也会提前退出。

> Shell 编程和任何编程语言一样是个很深的主题，但要提醒你：bash 的陷阱多得异乎寻常，甚至有[多个](https://tldp.org/LDP/abs/html/gotchas.html)网站专门[罗列它们](https://mywiki.wooledge.org/BashPitfalls)。我强烈建议在编写时大量使用 [shellcheck](https://www.shellcheck.net/)。大语言模型同样很擅长编写和调试 Shell 脚本，也能在脚本变得过于笨重（100 行以上）时把它们翻译成一门“真正的”编程语言（比如 Python）。

# 下一步

到目前为止，你已经掌握了足够的 Shell 知识来完成基本任务。你应该能够四处导航找到感兴趣的文件，并使用大多数程序的基本功能。在下一讲中，我们将讨论如何用 Shell 以及众多方便的命令行程序来执行和自动化更复杂的任务。

# 练习

本课程的每一讲都配有一系列练习。有些会给你一个明确的任务，另一些则是开放式的，比如“试试 X 和 Y 程序”。我们非常鼓励你动手尝试。

我们没有为练习编写参考答案。如果你在某个具体问题上卡住了，欢迎在 [Discord](https://ossu.dev/#community) 的 `#missing-semester-forum` 频道发帖，或给我们发邮件描述你目前已经尝试过的内容，我们会尽力帮助你。这些练习也很适合作为你与大语言模型对话的初始提示词，在对话中你可以交互式地深入钻研这些主题。这些练习真正的价值在于寻找答案的过程，而不是答案本身。我们鼓励你在做练习时顺着好奇心追根究底、多问“为什么”，而不是只寻找通往答案的最短路径。

1. 本课程要求你使用 Unix 风格的 Shell，比如 Bash 或 ZSH。如果你用的是 Linux 或 macOS，不需要做任何特别的准备。如果你用的是 Windows，你需要确保自己运行的不是 cmd.exe 或 PowerShell；你可以使用 [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/) 或 Linux 虚拟机来使用 Unix 风格的命令行工具。要确认你运行的是合适的 Shell，可以试试命令 `echo $SHELL`。如果它输出类似 `/bin/bash` 或 `/usr/bin/zsh` 的内容，说明你运行的是正确的程序。
2. `ls` 的 `-l` 旗标是做什么的？运行 `ls -l /` 并观察输出。每一行的前 10 个字符是什么意思？（提示：`man ls`）
3. 在命令 `find ~/Downloads -type f -name "*.zip" -mtime +30` 中，`*.zip` 是一个“glob”（通配模式）。什么是 glob？创建一个测试目录和若干文件，试试 `ls *.txt`、`ls file?.txt`、`ls {a,b,c}.txt` 这样的模式。参见 Bash 手册中的[模式匹配](https://www.gnu.org/software/bash/manual/html_node/Pattern-Matching.html)（Pattern Matching）。
4. `'单引号'`、`"双引号"` 和 `$'ANSI 引号'` 之间有什么区别？写一条命令，echo 一个包含字面 `$`、一个 `!` 和一个换行符的字符串。参见[引用](https://www.gnu.org/software/bash/manual/html_node/Quoting.html)（Quoting）。
5. Shell 有三个标准流（standard streams）：stdin（0）、stdout（1）和 stderr（2）。运行 `ls /nonexistent /tmp`，把 stdout 重定向到一个文件、stderr 重定向到另一个文件。如何把两者重定向到同一个文件？参见[重定向](https://www.gnu.org/software/bash/manual/html_node/Redirections.html)（Redirections）。
6. `$?` 保存上一条命令的退出状态（exit status，0 = 成功）。`&&` 只有在前一条命令成功时才运行下一条命令；`||` 只有在前一条命令失败时才运行它。写一个单行命令，仅当 `/tmp/mydir` 尚不存在时才创建它。参见[退出状态](https://www.gnu.org/software/bash/manual/html_node/Exit-Status.html)（Exit Status）。
7. 为什么 `cd` 必须内置于 Shell 本身，而不能是一个独立的程序？（提示：想想子进程能和不能影响其父进程的哪些方面。）
8. 写一个脚本，以文件名作为参数（`$1`），并用 `test -f` 或 `[ -f ... ]` 检查该文件是否存在。它应该根据文件是否存在打印不同的消息。参见 [Bash 条件表达式](https://www.gnu.org/software/bash/manual/html_node/Bash-Conditional-Expressions.html)（Bash Conditional Expressions）。
9. 把上一个练习中的脚本保存到文件（例如 `check.sh`）。试着用 `./check.sh somefile` 运行它。发生了什么？现在运行 `chmod +x check.sh` 再试一次。为什么这一步是必要的？（提示：对比 `chmod` 前后 `ls -l check.sh` 的输出。）
10. 如果在脚本的 `set` 旗标中加上 `-x` 会发生什么？用一个简单脚本试一试，观察输出。参见 [set 内建命令](https://www.gnu.org/software/bash/manual/html_node/The-Set-Builtin.html)（The Set Builtin）。
11. 写一条命令，把一个文件复制为文件名中带今天日期的备份（例如 `notes.txt` → `notes_2026-01-12.txt`）。（提示：`$(date +%Y-%m-%d)`）。参见[命令替换](https://www.gnu.org/software/bash/manual/html_node/Command-Substitution.html)（Command Substitution）。
12. 修改讲座中的不稳定测试脚本，让它接受测试命令作为参数，而不是硬编码 `cargo test my_test`。（提示：`$1` 或 `$@`）。参见[特殊参数](https://www.gnu.org/software/bash/manual/html_node/Special-Parameters.html)（Special Parameters）。
13. 用管道找出你的主目录中最常见的 5 种文件扩展名。（提示：组合使用 `find`、`grep` 或 `sed` 或 `awk`、`sort`、`uniq -c` 和 `head`。）
14. `xargs` 把 stdin 中的各行转换成命令参数。把 `find` 和 `xargs` 配合使用（不要用 `find -exec`），找出一个目录下的所有 `.sh` 文件，并用 `wc -l` 统计每个文件的行数。附加题：让它能正确处理带空格的文件名。（提示：`-print0` 和 `-0`）。参见 `man xargs`。
15. 用 `curl` 抓取课程网站（`https://missing.csail.mit.edu/`）的 HTML，并通过管道传给 `grep` 来统计页面上列出了多少讲。（提示：找一个每讲出现一次的模式；用 `curl -s` 关闭进度输出。）
16. [`jq`](https://jqlang.github.io/jq/) 是处理 JSON 数据的强大工具。用 `curl` 抓取示例数据 `https://microsoftedge.github.io/Demos/json-dummy-data/64KB.json`，并用 `jq` 只提取 version 大于 6 的人的姓名。（提示：先通过管道传给 `jq .` 看看数据结构；然后试试 `jq '.[] | select(...) | .name'`）
17. `awk` 可以根据列值过滤行并操纵输出。例如，`awk '$3 ~ /pattern/ {$4=""; print}'` 只打印第三列匹配 `pattern` 的行，同时省略第四列。写一条 `awk` 命令，只打印第二列大于 100 的行，并交换第一列和第三列。用以下内容测试：`printf 'a 50 x\nb 150 y\nc 200 z\n'`
18. 拆解讲座中的 SSH 日志管道：每一步分别做了什么？然后构造一个类似的管道，从 `~/.bash_history`（或 `~/.zsh_history`）中找出你最常用的 Shell 命令。

---

原文：https://missing.csail.mit.edu/2026/course-shell/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
