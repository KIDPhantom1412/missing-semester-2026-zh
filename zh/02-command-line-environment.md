# 命令行环境

正如我们在上一讲中介绍的，大多数 shell 并不只是用来启动其他程序的启动器，实际上它们提供了一整套编程语言，充满常见的模式和抽象。然而与大多数编程语言不同，shell 脚本中的一切都是围绕运行程序、并让程序之间简单高效地通信来设计的。

尤其是，shell 脚本深受*约定（convention）*的约束。一个命令行界面（command line interface，CLI）程序要想在整个 shell 环境中良好协作，就需要遵循一些通用的模式。接下来我们将讲解理解命令行程序工作原理所需的许多概念，以及在使用和配置它们方面无处不在的约定。

# 命令行界面

在大多数编程语言中，编写一个函数大致是这样的：

```
def add(x: int, y: int) -> int:
    return x + y
```

这里我们可以显式地看到程序的输入和输出。相比之下，shell 脚本乍看起来可能很不一样。

```shell
#!/usr/bin/env bash

if [[ -f $1 ]]; then
    echo "Target file already exists"
    exit 1
else
    if $DEBUG; then
        grep 'error' - | tee $1
    else
        grep 'error' - > $1
    fi
    exit 0
fi
```

要正确理解这样的脚本里发生了什么，我们首先需要介绍几个在 shell 程序之间或与 shell 环境通信时经常出现的概念：

* 参数
* 流
* 环境变量
* 返回码
* 信号

## 参数

shell 程序在执行时会收到一个参数列表。在 shell 中参数就是普通的字符串，如何解释它们由程序自己决定。例如当我们执行 `ls -l folder/` 时，实际是以参数 `['-l', 'folder/']` 执行程序 `/bin/ls`。

在 shell 脚本内部，我们通过特殊的 shell 语法来访问这些参数。访问第一个参数用变量 `$1`，第二个参数用 `$2`，依此类推直到 `$9`。要把所有参数作为一个列表来访问用 `$@`，要获取参数的个数则用 `$#`。此外还可以用 `$0` 访问程序名。

对大多数程序来说，参数由*标志（flag）*和普通字符串混合组成。标志的前面带有短横线（`-`）或双短横线（`--`），据此可以识别它们。标志通常是可选的，作用是修改程序的行为。例如 `ls -l` 会改变 `ls` 格式化其输出的方式。

你会看到带长名称的双短横线标志，如 `--all`，以及单短横线标志如 `-a`，后者通常后面只跟一个字母。同一个选项可能有两种写法，`ls -a` 和 `ls --all` 是等价的。单短横线标志经常组合书写，所以 `ls -l -a` 和 `ls -la` 也是等价的。标志的顺序通常也不重要，`ls -la` 和 `ls -al` 产生同样的结果。有些标志非常常见，随着你对 shell 环境越来越熟悉，你会自然而然地想到使用它们，例如（`--help`、`--verbose`、`--version`）。

> 标志是 shell 约定的第一个很好的例子。shell 语言并不要求我们的程序以这种特定方式使用 `-` 或 `--`。没有什么能阻止我们写出语法为 `myprogram +myoption myfile` 的程序，但这会造成困惑，因为人们的预期是我们使用短横线。实际上，大多数编程语言都提供 CLI 标志解析库（例如 Python 中用 `argparse` 来解析带短横线语法的参数）。

CLI 程序中另一个常见约定是接受同一类型的可变数量的参数。以这种方式给出参数时，命令会对其中每个参数执行相同的操作。

```shell
mkdir src
mkdir docs
# is equivalent to
mkdir src docs
```

这种语法糖起初看似没有必要，但与*通配符展开（globbing）*结合起来时会变得非常强大。通配符展开（globbing）或称 glob 是一些特殊模式，shell 会在调用程序之前先对它们进行展开。

假设我们想删除当前文件夹中所有 .py 文件（不递归）。根据上一讲所学，我们可以通过运行下面的命令来实现

```shell
for file in $(ls | grep -P '\.py$'); do
    rm "$file"
done
```

但我们只需 `rm *.py` 就能取代它！

当我们在终端中输入 `rm *.py` 时，shell 不会用参数 `['*.py']` 调用 `/bin/rm` 程序。相反，shell 会在当前文件夹中搜索匹配模式 `*.py` 的文件，其中 `*` 可以匹配零个或多个任意字符组成的任意字符串。所以如果我们的文件夹中有 `main.py` 和 `utils.py`，那么 `rm` 程序收到的参数将是 `['main.py', 'utils.py']`。

你最常见到的 glob 是通配符 `*`（零个或多个任意字符）、`?`（恰好一个任意字符）以及花括号。花括号 `{}` 会把一个以逗号分隔的模式列表展开为多个参数。

在实践中，最好通过具体的例子来理解 glob。

```shell
touch folder/{a,b,c}.py
# Will expand to
touch folder/a.py folder/b.py folder/c.py

convert image.{png,jpg}
# Will expand to
convert image.png image.jpg

cp /path/to/project/{setup,build,deploy}.sh /newpath
# Will expand to
cp /path/to/project/setup.sh /path/to/project/build.sh /path/to/project/deploy.sh /newpath

# Globbing techniques can also be combined
mv *{.py,.sh} folder
# Will move all *.py and *.sh files
```

> 有些 shell（例如 zsh）支持更高级的通配符展开形式，例如 `**`，它会展开为包含递归路径。因此 `rm **/*.py` 会递归删除所有 .py 文件。

## 流

每当我们执行像下面这样的程序管道（pipeline）时

```shell
cat myfile | grep -P '\d+' | uniq -c
```

我们会看到 `grep` 程序同时在与 `cat` 和 `uniq` 程序通信。

这里一个重要观察是这三个程序都在同时执行。也就是说，shell 并不是先调用 cat、再调用 grep、然后调用 uniq。相反，三个程序同时被生成（spawn），shell 把 cat 的输出连接到 grep 的输入，把 grep 的输出连接到 uniq 的输入。使用管道运算符 `|` 时，shell 操作的是从链条中的一个程序流向下一个程序的数据流。

我们可以演示这种并发性，管道中的所有命令都会立即启动：

```console
$ (sleep 15 && cat numbers.txt) | grep -P '^\d$' | sort | uniq  &
[1] 12345
$ ps | grep -P '(sleep|cat|grep|sort|uniq)'
  32930 pts/1    00:00:00 sleep
  32931 pts/1    00:00:00 grep
  32932 pts/1    00:00:00 sort
  32933 pts/1    00:00:00 uniq
  32948 pts/1    00:00:00 grep
```

可以看到除 `cat` 之外的所有进程都立刻在运行。shell 会生成所有进程并在其中任何一个完成之前连接好它们的流。`cat` 要等到 sleep 结束后才会启动，`cat` 的输出会被发送给 grep，依此类推。

每个程序都有一个输入流，标记为 stdin（standard input，标准输入）。使用管道时，stdin 会被自动连接。在脚本中，许多程序接受 `-` 作为文件名，表示“从 stdin 读取”：

```shell
# These are equivalent when data comes from a pipe
echo "hello" | grep "hello"
echo "hello" | grep "hello" -
```

类似地，每个程序有两个输出流：stdout 和 stderr。标准输出（stdout）是最常用到的那一个，它用于把程序的输出通过管道传给管道中的下一个命令。标准错误（stderr）是一个备用流，供程序报告警告和其他类型的问题，这样这些输出就不会被链条中的下一个命令解析。

```console
$ ls /nonexistent
ls: cannot access '/nonexistent': No such file or directory
$ ls /nonexistent | grep "pattern"
ls: cannot access '/nonexistent': No such file or directory
# The error message still appears because stderr is not piped
$ ls /nonexistent 2>/dev/null
# No output - stderr was redirected to /dev/null
```

shell 提供了重定向（redirect）这些流的语法。下面是一些说明性的例子。

```shell
# Redirect stdout to a file (overwrite)
echo "hello" > output.txt

# Redirect stdout to a file (append)
echo "world" >> output.txt

# Redirect stderr to a file
ls foobar 2> errors.txt

# Redirect both stdout and stderr to the same file
ls foobar &> all_output.txt

# Redirect stdin from a file
grep "pattern" < input.txt

# Discard output by redirecting to /dev/null
cmd > /dev/null 2>&1
```

另一个体现 Unix 哲学的强大工具是 [`fzf`](https://github.com/junegunn/fzf)，一个模糊查找器（fuzzy finder）。它从 stdin 读取各行，并提供一个交互式界面来筛选和选择：

```console
$ ls | fzf
$ cat ~/.bash_history | fzf
```

`fzf` 可以与许多 shell 操作集成。在讨论 shell 定制时，我们会看到它的更多用法。

## 环境变量

在 bash 中给变量赋值使用 `foo=bar` 这样的语法，然后用 `$foo` 语法访问变量的值。注意 `foo = bar` 是无效语法，因为 shell 会把它解析为以参数 `['=', 'bar']` 调用程序 `foo`。在 shell 脚本中，空格字符的作用是执行参数分割（argument splitting）。这种行为可能令人困惑、也不容易习惯，请牢记这一点。

shell 变量没有类型，它们全都是字符串。注意在 shell 中书写字符串表达式时，单引号和双引号不可互换。用 `'` 包围的字符串是字面字符串（literal string），不会展开变量、执行命令替换或处理转义序列，而用 `"` 包围的字符串则会。

```shell
foo=bar
echo "$foo"
# prints bar
echo '$foo'
# prints $foo
```

要把命令的输出捕获到变量中，我们使用*命令替换（command substitution）*。当我们执行

```shell
files=$(ls)
echo "$files" | grep README
echo "$files" | grep ".py"
```

时，ls 的输出（具体说是 stdout）被放进变量 `$files`，我们可以在之后访问它。`$files` 变量的内容确实包含 ls 输出中的换行符，`grep` 这类程序正是凭这一点知道要对每一项独立操作。

一个较少人知道的类似特性是*进程替换（process substitution）*，`<( CMD )` 会执行 `CMD` 并把输出放进一个临时文件，同时用该文件的名字替换 `<()`。当命令期望以文件而不是 STDIN 的方式传入值时，这很有用。例如，`diff <(ls src) <(ls docs)` 会显示 `src` 和 `docs` 目录中文件之间的差异。

每当 shell 程序调用另一个程序时，它会传递一组变量，这些变量通常被称为*环境变量（environment variable）*。在 shell 中，我们可以通过运行 `printenv` 来查看当前的环境变量。要显式地传递一个环境变量，我们可以在命令前加上一个变量赋值

> 环境变量按惯例全部用大写字母书写（例如 `HOME`、`PATH`、`DEBUG`）。这是一种约定，而不是技术要求，但遵循它有助于把环境变量与通常为小写的局部 shell 变量区分开来。

```shell
TZ=Asia/Tokyo date  # prints the current time in Tokyo
echo $TZ  # this will be empty, since TZ was only set for the child command
```

另外，我们也可以使用 `export` 内建（built-in）命令，它会修改我们当前的环境，从而让所有子进程都继承该变量：

```shell
export DEBUG=1
# All programs from this point onwards will have DEBUG=1 in their environment
bash -c 'echo $DEBUG'
# prints 1
```

要删除一个变量，使用 `unset` 内建命令，例如 `unset DEBUG`。

> 环境变量是另一种 shell 约定。它们可以隐式地修改许多程序的行为，而不必显式传参。例如，shell 会把 `$HOME` 环境变量设置为当前用户主目录的路径。然后程序可以访问这个变量来获取该信息，而不需要显式的 `--home /home/alice`。另一个常见例子是 `$TZ`，许多程序用它按照指定的时区来格式化日期和时间。

## 返回码

正如我们前面所见，shell 程序的主要输出是通过 stdout/stderr 流和文件系统的副作用来传递的。

默认情况下 shell 脚本返回退出码（exit code）零。约定是零表示一切顺利，而非零表示遇到了某些问题。要返回非零退出码，我们必须使用 shell 内建的 `exit NUM`。通过访问特殊变量 `$?`，可以获取最后运行的命令的返回码。

shell 有布尔运算符 `&&` 和 `||`，分别执行 AND 和 OR 运算。与普通编程语言中的那些不同，shell 中的这些运算符作用于程序的返回码。两者都是[短路求值（short-circuiting）](https://en.wikipedia.org/wiki/Short-circuit_evaluation)运算符。这意味着可以用它们根据之前命令的成功或失败来有条件地运行命令，而成功与否由返回码是否为零来判定。一些例子：

```shell
# echo will only run if grep succeeds (finds a match)
grep -q "pattern" file.txt && echo "Pattern found"

# echo will only run if grep fails (no match)
grep -q "pattern" file.txt || echo "Pattern not found"

# true is a shell program that always succeeds
true && echo "This will always print"

# and false is a shell program that always fails
false || echo "This will always print"
```

同样的原则适用于 `if` 和 `while` 语句，它们都使用返回码来做判断：

```shell
# if uses the return code of the condition command (0 = true, nonzero = false)
if grep -q "pattern" file.txt; then
    echo "Found"
fi

# while loops continue as long as the command returns 0
while read line; do
    echo "$line"
done < file.txt
```

## 信号

在某些情况下，你需要在程序执行过程中中断它，例如某个命令花了太长时间才完成。中断程序最简单的方式是按下 `Ctrl-C`，命令很可能会停止。但这究竟是怎么做到的？为什么它有时无法停止进程？

```console
$ sleep 100
^C
$
```

> 注意，这里 `^C` 是 `Ctrl-C` 在终端中键入时的显示方式。

在底层，这里发生的是如下过程：

1. 我们按下 `Ctrl-C`
2. shell 识别出这个特殊的字符组合
3. shell 进程向 `sleep` 进程发送了一个 SIGINT 信号
4. 该信号中断了 `sleep` 进程的执行

信号（signal）是一种特殊的通信机制。当进程收到一个信号时，它会停止自己的执行、处理该信号，并可能根据信号传递的信息改变执行流程。正因如此，信号是*软件中断（software interrupt）*。

在我们的例子中，键入 `Ctrl-C` 会让 shell 向进程传递一个 `SIGINT` 信号。下面是一个 Python 程序的最小示例，它捕获 `SIGINT` 并忽略它，不再停止。要杀掉这个程序，我们现在可以改用 `SIGQUIT` 信号，即键入 `Ctrl-\`。

```python
#!/usr/bin/env python
import signal, time

def handler(signum, time):
    print("\nI got a SIGINT, but I am not stopping")

signal.signal(signal.SIGINT, handler)
i = 0
while True:
    time.sleep(.1)
    print("\r{}".format(i), end="")
    i += 1
```

下面是我们向该程序发送两次 `SIGINT`、随后发送 `SIGQUIT` 时发生的情况。注意 `^` 是 `Ctrl` 在终端中键入时的显示方式。

```console
$ python sigint.py
24^C
I got a SIGINT, but I am not stopping
26^C
I got a SIGINT, but I am not stopping
30^\[1]    39913 quit       python sigint.py
```

虽然 `SIGINT` 和 `SIGQUIT` 通常都与终端相关的请求联系在一起，但要请求进程优雅地退出，更通用的信号是 `SIGTERM` 信号。要发送这个信号，我们可以使用 [`kill`](https://www.man7.org/linux/man-pages/man1/kill.1.html) 命令，语法为 `kill -TERM <PID>`。

信号除了杀掉进程之外还能做其他事情。例如，`SIGSTOP` 会暂停一个进程。在终端中键入 `Ctrl-Z` 会让 shell 发送一个 `SIGTSTP` 信号，它是 Terminal Stop 的缩写（即终端版的 `SIGSTOP`）。

然后我们可以使用 [`fg`](https://www.man7.org/linux/man-pages/man1/fg.1p.html) 或 [`bg`](https://man7.org/linux/man-pages/man1/bg.1p.html) 分别在前台或后台继续被暂停的作业。

[`jobs`](https://www.man7.org/linux/man-pages/man1/jobs.1p.html) 命令列出与当前终端会话关联的未完成作业。你可以用这些作业的 pid 来指代它们（可以用 [`pgrep`](https://www.man7.org/linux/man-pages/man1/pgrep.1.html) 查出 pid）。更直观地，你也可以用百分号加上作业号（由 `jobs` 显示）来指代某个进程。要指代最近一个被放到后台的作业，可以使用 `$!` 特殊参数。

还有一点需要知道：命令后面的 `&` 后缀会让命令在后台运行，把提示符还给你，不过它仍然会使用 shell 的 STDOUT，这可能会很烦人（这种情况下请使用 shell 重定向）。同样地，要把一个已经在运行的程序放到后台，可以先按 `Ctrl-Z` 再输入 `bg`。

注意，放到后台的进程仍然是你的终端的子进程，如果你关闭终端，它们也会死掉（这会再发送一个信号 `SIGHUP`）。要防止这种情况发生，你可以用 [`nohup`](https://www.man7.org/linux/man-pages/man1/nohup.1.html)（一个忽略 `SIGHUP` 的包装器）来运行程序，或者如果进程已经启动，则使用 `disown`。另外，你也可以使用终端复用器（terminal multiplexer），我们将在下一节看到它。

下面是一个示例会话，展示其中一些概念。

```
$ sleep 1000
^Z
[1]  + 18653 suspended  sleep 1000

$ nohup sleep 2000 &
[2] 18745
appending output to nohup.out

$ jobs
[1]  + suspended  sleep 1000
[2]  - running    nohup sleep 2000

$ kill -SIGHUP %1
[1]  + 18653 hangup     sleep 1000

$ kill -SIGHUP %2   # nohup protects from SIGHUP

$ jobs
[2]  + running    nohup sleep 2000

$ kill %2
[2]  + 18745 terminated  nohup sleep 2000
```

一个特殊的信号是 `SIGKILL`，因为它无法被进程捕获，且总会立即终止进程。不过它可能有不良副作用，比如留下孤立的子进程。

你可以在[这里](https://en.wikipedia.org/wiki/Signal_(IPC))了解更多关于这些及其他信号的信息，或者输入 [`man signal`](https://www.man7.org/linux/man-pages/man7/signal.7.html) 或 `kill -l`。

在 shell 脚本中，你可以使用 `trap` 内建命令在收到信号时执行命令。这对清理操作很有用：

```shell
#!/usr/bin/env bash
cleanup() {
    echo "Cleaning up temporary files..."
    rm -f /tmp/mytemp.*
}
trap cleanup EXIT  # Run cleanup when script exits
trap cleanup SIGINT SIGTERM  # Also on Ctrl-C or kill
```

# 远程机器

程序员在日常工作中与远程服务器打交道越来越普遍。这项工作最常见的工具是 SSH（Secure Shell，安全外壳），它帮助我们连接到远程服务器，并提供大家现在已经熟悉的 shell 界面。我们用类似下面的命令连接到服务器：

```bash
ssh alice@server.mit.edu
```

这里我们尝试以用户 `alice` 的身份 ssh 登录服务器 `server.mit.edu`。

`ssh` 一个常被忽视的特性是以非交互方式运行命令的能力。`ssh` 会正确处理发送命令的 stdin 和接收命令的 stdout，所以我们可以把它和其他命令组合起来

```shell
# here ls runs in the remote, and wc runs locally
ssh alice@server ls | wc -l

# here both ls and wc run in the server
ssh alice@server 'ls | wc -l'

```

> 试试安装 [Mosh](https://mosh.org/) 作为 SSH 的替代品，它能应对断连、进入/退出睡眠、切换网络以及高延迟链路。

要让 `ssh` 允许我们在远程服务器上运行命令，我们需要证明自己有权这样做。这可以通过密码或 ssh 密钥来完成。基于密钥的认证利用公钥密码学（public-key cryptography）向服务器证明客户端拥有秘密的私钥，而无需透露密钥本身。基于密钥的认证既更方便也更安全，所以你应该优先使用它。注意私钥（常见的是 `~/.ssh/id_rsa`，较新的是 `~/.ssh/id_ed25519`）实际上就是你的密码，因此要像对待密码一样对待它，绝不要分享它的内容。

要生成一对密钥，可以运行 [`ssh-keygen`](https://www.man7.org/linux/man-pages/man1/ssh-keygen.1.html)。

```bash
ssh-keygen -a 100 -t ed25519 -f ~/.ssh/id_ed25519
```

如果你曾经配置过使用 SSH 密钥向 GitHub 推送，那么你大概已经完成了[这里](https://help.github.com/articles/connecting-to-github-with-ssh/)列出的步骤，并且已经拥有一个有效的密钥对。要检查你的密钥是否有口令短语（passphrase）并验证它，可以运行 `ssh-keygen -y -f /path/to/key`。

在服务器端，`ssh` 会查看 `.ssh/authorized_keys` 来决定应该放行哪些客户端。要把公钥复制过去，你可以使用：

```bash
cat .ssh/id_ed25519.pub | ssh alice@remote 'cat >> ~/.ssh/authorized_keys'

# or more simply (if ssh-copy-id is available)

ssh-copy-id -i .ssh/id_ed25519 alice@remote
```

除了运行命令之外，ssh 建立的连接还可以用来在服务器和本地之间安全地传输文件。[`scp`](https://www.man7.org/linux/man-pages/man1/scp.1.html) 是最传统的工具，语法是 `scp path/to/local_file remote_host:path/to/remote_file`。[`rsync`](https://www.man7.org/linux/man-pages/man1/rsync.1.html) 在 `scp` 的基础上做了改进：它能检测本地和远端相同的文件，防止再次复制它们。它还提供对符号链接（symlink）、权限更细粒度的控制，并有像 `--partial` 标志这样的额外功能，可以从之前中断的复制处继续。`rsync` 的语法与 `scp` 类似。

SSH 客户端配置位于 `~/.ssh/config`，它让我们可以声明主机并为它们设置默认选项。这个配置文件不仅被 `ssh` 读取，`scp`、`rsync`、`mosh` 等程序也会读取它。

```bash
Host vm
    User alice
    HostName 172.16.174.141
    Port 2222
    IdentityFile ~/.ssh/id_ed25519

# Configs can also take wildcards
Host *.mit.edu
    User alice
```

# 终端复用器

使用命令行界面时，你经常会想同时运行多个东西。例如，你可能想并排运行你的编辑器和你的程序。虽然这可以通过打开新的终端窗口来实现，但使用终端复用器是更通用的解决方案。

像 [`tmux`](https://www.man7.org/linux/man-pages/man1/tmux.1.html) 这样的终端复用器允许你用窗格（pane）和标签页（tab）来复用终端窗口，从而高效地与多个 shell 会话交互。此外，终端复用器让你可以分离（detach）当前的终端会话，并在之后的某个时间重新连接（attach）。正因如此，终端复用器在使用远程机器时非常方便，因为它避免了使用 `nohup` 之类的技巧。

如今最流行的终端复用器是 [`tmux`](https://www.man7.org/linux/man-pages/man1/tmux.1.html)。`tmux` 的可配置性很强，通过使用相应的按键绑定，你可以创建多个标签页和窗格，并在它们之间快速导航。

`tmux` 要求你熟悉它的按键绑定，它们都采用 `<C-b> x` 的形式，意思是 (1) 按下 `Ctrl+b`，(2) 松开 `Ctrl+b`，然后 (3) 按下 `x`。`tmux` 有如下对象层次结构：

* **会话（Sessions）** - 会话是一个独立的工作空间，包含一个或多个窗口
  + `tmux` 启动一个新会话。
  + `tmux new -s NAME` 以该名字启动会话。
  + `tmux ls` 列出当前的会话
  + 在 `tmux` 中键入 `<C-b> d` 分离当前会话
  + `tmux a` 连接最近的会话。你可以用 `-t` 标志指定连接哪一个
* **窗口（Windows）** - 相当于编辑器或浏览器中的标签页，是同一会话中视觉上相互独立的部分
  + `<C-b> c` 创建一个新窗口。要关闭它，只需按 `<C-d>` 终止其中的 shell 即可
  + `<C-b> N` 跳转到第 *N* 个窗口。注意它们是按数字编号的
  + `<C-b> p` 跳转到上一个窗口
  + `<C-b> n` 跳转到下一个窗口
  + `<C-b> ,` 重命名当前窗口
  + `<C-b> w` 列出当前的窗口
* **窗格（Panes）** - 类似 vim 的分屏，窗格让你在同一个显示画面中拥有多个 shell。
  + `<C-b> "` 水平分割当前窗格
  + `<C-b> %` 垂直分割当前窗格
  + `<C-b> <direction>` 移动到指定*方向*的窗格。这里的方向指方向键。
  + `<C-b> z` 切换当前窗格的缩放（最大化）
  + `<C-b> [` 进入回滚浏览。然后可以按 `<space>` 开始一个选区，并按 `<enter>` 复制该选区。
  + `<C-b> <space>` 在不同的窗格布局之间循环切换。

> 想进一步了解 tmux，可以阅读[这篇](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/)快速教程和[这篇](https://linuxcommand.org/lc3_adv_termmux.php)更详细的讲解。

有了 tmux 和 SSH，你会希望在任何机器上都能让环境像家一样顺手。这就轮到 shell 定制登场了。

# 自定义 Shell

大量的命令行程序都使用称为*点文件（dotfiles）*的纯文本文件来配置（之所以这么叫，是因为文件名以 `.` 开头，例如 `~/.vimrc`，这样它们在 `ls` 的目录列表中默认是隐藏的）。

> 点文件是又一个 shell 约定。前面的点是为了在列目录时把它们“隐藏”起来（是的，又是一个约定）。

shell 就是用这类文件配置的程序的一个例子。启动时，你的 shell 会读取许多文件来加载其配置。根据所用的 shell 以及你启动的是登录会话还是交互式会话（或两者），整个过程可能相当复杂。[这里](https://web.archive.org/web/20260329133158/https://blog.flowblok.id.au/2013-02/shell-startup-scripts.html)有一份关于该主题的优秀资料。

对 `bash` 来说，在大多数系统上编辑你的 `.bashrc` 或 `.bash_profile` 就可以。其他一些可以通过点文件配置的工具有：

* `bash` - `~/.bashrc`、`~/.bash_profile`
* `git` - `~/.gitconfig`
* `vim` - `~/.vimrc` 和 `~/.vim` 文件夹
* `ssh` - `~/.ssh/config`
* `tmux` - `~/.tmux.conf`

一个常见的配置改动是添加新的位置供 shell 查找程序。你在安装软件时会遇到这种模式：

```shell
export PATH="$PATH:path/to/append"
```

这里，我们让 shell 把 $PATH 变量的值设置为其当前值再加上一个新路径，并让所有子进程继承这个新的 PATH 值。这将使子进程能够找到位于 `path/to/append` 下的程序。

定制 shell 往往意味着安装新的命令行工具。包管理器让这变得容易。它们负责下载、安装和更新软件。不同的操作系统有不同的包管理器：macOS 使用 [Homebrew](https://brew.sh/)，Ubuntu/Debian 使用 `apt`，Fedora 使用 `dnf`，Arch 使用 `pacman`。我们将在代码发布（shipping code）一课中更深入地介绍包管理器。

下面是在 macOS 上用 Homebrew 安装两个实用工具的方法：

```shell
# ripgrep: a faster grep with better defaults
brew install ripgrep

# fd: a faster, user-friendly find
brew install fd
```

装好之后，你就可以用 `rg` 代替 `grep`，用 `fd` 代替 `find`。

> **关于 `curl | bash` 的警告**：你经常会看到类似 `curl -fsSL https://example.com/install.sh | bash` 的安装说明。这种模式下载一个脚本并立即执行它，很方便但有风险；你运行的是自己没有检查过的代码。更安全的做法是先下载、审查，然后再执行：
>
> ```shell
> curl -fsSL https://example.com/install.sh -o install.sh
> less install.sh  # review the script
> bash install.sh
> ```
>
> 有些安装程序使用一种稍安全的变体：`/bin/bash -c "$(curl -fsSL https://url)"`，它至少能确保由 bash 而不是你当前的 shell 来解释脚本。

当你尝试运行一个没有安装的命令时，你的 shell 会显示 `command not found`。网站 [command-not-found.com](https://command-not-found.com) 是一份有用的资源，你可以用它搜索任何命令，了解如何在不同的包管理器和发行版上安装它。

另一个实用工具是 [`tldr`](https://tldr.sh/)，它提供简化的、以示例为主的 man 手册页。不必读完冗长的文档，你就能快速看到常见的用法模式：

```console
$ tldr fd
  An alternative to find.
  Aims to be faster and easier to use than find.

  Recursively find files matching a pattern in the current directory:
      fd "pattern"

  Find files that begin with "foo":
      fd "^foo"

  Find files with a specific extension:
      fd --extension txt
```

有时候你需要的并不是一整个新程序，而只是给现有命令加上特定标志的快捷方式。这时别名（alias）就派上用场了。

我们也可以使用 `alias` shell 内建命令创建自己的命令别名。shell 别名是另一个命令的简写形式，你的 shell 会在对表达式求值之前自动替换它。例如，bash 中的一个别名有如下结构：

```bash
alias alias_name="command_to_alias arg1 arg2"
```

> 注意等号 `=` 两边没有空格，因为 [`alias`](https://www.man7.org/linux/man-pages/man1/alias.1p.html) 是一个只接受单个参数的 shell 命令。

别名有许多方便的特性：

```bash
# Make shorthands for common flags
alias ll="ls -lh"

# Save a lot of typing for common commands
alias gs="git status"
alias gc="git commit"

# Save you from mistyping
alias sl=ls

# Overwrite existing commands for better defaults
alias mv="mv -i"           # -i prompts before overwrite
alias mkdir="mkdir -p"     # -p make parent dirs as needed
alias df="df -h"           # -h prints human readable format

# Alias can be composed
alias la="ls -A"
alias lla="la -l"

# To ignore an alias run it prepended with \
\ls
# Or disable an alias altogether with unalias
unalias la

# To get an alias definition just call it with alias
alias ll
# Will print ll='ls -lh'
```

别名有局限性：它们无法在命令中间接受参数。要实现更复杂的行为，你应该改用 shell 函数。

大多数 shell 支持用 `Ctrl-R` 进行反向历史搜索。按下 `Ctrl-R` 然后开始输入，即可搜索之前的命令。前面我们把 `fzf` 作为模糊查找器介绍过；配置好 fzf 的 shell 集成后，`Ctrl-R` 会变成对你整个历史记录的交互式模糊搜索，比默认实现强大得多。

你应该如何组织你的点文件？它们应该放在自己的文件夹里、纳入版本控制（version control），并用脚本**符号链接（symlink）**到位。这样做的好处有：

* **易于安装**：如果你登录到一台新机器，应用你的定制只需一分钟。
* **可移植性**：你的工具在任何地方都以同样的方式工作。
* **同步**：你可以在任何地方更新你的点文件，并让它们全部保持同步。
* **变更追踪**：你可能会在整个编程生涯中维护你的点文件，而对于长寿项目来说拥有版本历史是很好的。

你该往点文件里放些什么？你可以通过阅读在线文档或 [man 页面](https://en.wikipedia.org/wiki/Man_page)来了解工具的设置。另一个好方法是在网上搜索关于特定程序的博客文章，作者们会在其中介绍他们偏爱的定制方式。还有一种了解定制的方式是翻看别人的点文件：你可以在 GitHub 上找到大量 [dotfiles 仓库](https://github.com/search?o=desc&q=dotfiles&s=stars&type=Repositories)—— 最受欢迎的一个见[这里](https://github.com/mathiasbynens/dotfiles)（不过我们建议你不要盲目照抄配置）。[这里](https://dotfiles.github.io/)是关于该主题的另一份好资源。

本课程的所有授课教师都把自己的点文件公开在 GitHub 上：[Anish](https://github.com/anishathalye/dotfiles)、[Jon](https://github.com/jonhoo/configs)、[Jose](https://github.com/jjgo/dotfiles)。

**框架和插件**也能改进你的 shell。一些流行的通用框架有 [prezto](https://github.com/sorin-ionescu/prezto) 或 [oh-my-zsh](https://ohmyz.sh/)，还有一些专注于特定功能的小插件：

* [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) - 在你输入时给有效/无效的命令着色
* [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) - 在你输入时从历史记录中建议命令
* [zsh-completions](https://github.com/zsh-users/zsh-completions) - 额外的补全定义
* [zsh-history-substring-search](https://github.com/zsh-users/zsh-history-substring-search) - 类似 fish 的历史搜索
* [powerlevel10k](https://github.com/romkatv/powerlevel10k) - 快速、可定制的提示符主题

像 [fish](https://fishshell.com/) 这样的 shell 默认就包含了许多这些功能。

> 你并不需要 oh-my-zsh 这样的庞大框架来获得这些功能。单独安装插件往往更快，也给你更多控制权。大型框架会显著拖慢 shell 的启动时间，所以请考虑只安装你真正会用的东西。

# Shell 中的 AI

把 AI 工具融入 shell 有很多种方式。下面是几个集成深度不同的例子：

**命令生成**：像 [`simonw/llm`](https://github.com/simonw/llm) 这样的工具可以帮助从自然语言描述生成 shell 命令：

```console
$ llm cmd "find all python files modified in the last week"
find . -name "*.py" -mtime -7
```

**管道集成**：大语言模型（LLM）可以集成到 shell 管道中来处理和转换数据。当你需要从不一致的格式中提取信息、而用正则表达式会很痛苦时，它们特别有用：

```console
$ cat users.txt
Contact: john.doe@example.com
User 'alice_smith' logged in at 3pm
Posted by: @bob_jones on Twitter
Author: Jane Doe (jdoe)
Message from mike_wilson yesterday
Submitted by user: sarah.connor
$ INSTRUCTIONS="Extract just the username from each line, one per line, nothing else"
$ llm "$INSTRUCTIONS" < users.txt
john.doe
alice_smith
bob_jones
jdoe
mike_wilson
sarah.connor
```

注意我们使用了 `"$INSTRUCTIONS"`（加了引号），因为变量中包含空格，还用 `< users.txt` 把文件的内容重定向到 stdin。

**AI shell**：像 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 这样的工具充当一种元 shell（meta-shell），它接受英语命令并把它们翻译成 shell 操作、文件编辑以及更复杂的多步骤任务。

# 终端模拟器

在定制 shell 的同时，值得花些时间斟酌你对**终端模拟器（terminal emulator）**的选择及其设置。终端模拟器是一个 GUI 程序，它提供你的 shell 运行所在的那个文本界面。终端模拟器有很多种。

由于你可能会在终端里度过数百到数千小时，研究一下它的设置是值得的。你可能想要修改的终端设置包括：

* 字体选择
* 配色方案
* 键盘快捷键
* 标签页/窗格支持
* 回滚（scrollback）配置
* 性能（一些较新的终端如 [Alacritty](https://github.com/alacritty/alacritty) 或 [Ghostty](https://ghostty.org/) 提供 GPU 加速）。

# 练习

## 参数与通配符展开

1. 你可能会看到类似 `cmd --flag -- --notaflag` 的命令。`--` 是一个特殊参数，它告诉程序停止解析标志。`--` 之后的一切都被当作位置参数（positional argument）。这为什么可能有用？试着运行 `touch -- -myfile`，然后在不带 `--` 的情况下把它删掉。
2. 阅读 [`man ls`](https://www.man7.org/linux/man-pages/man1/ls.1.html) 并写出一条 `ls` 命令，按以下方式列出文件：
   * 包括所有文件，包括隐藏文件
   * 大小以人类可读的格式列出（例如 454M 而不是 454279954）
   * 文件按新近程度排序
   * 输出带有颜色

   一个示例输出大致是这样的：

   ```
    -rw-r--r--   1 user group 1.1M Jan 14 09:53 baz
    drwxr-xr-x   5 user group  160 Jan 14 09:53 .
    -rw-r--r--   1 user group  514 Jan 14 06:42 bar
    -rw-r--r--   1 user group 106M Jan 13 12:12 foo
    drwx------+ 47 user group 1.5K Jan 12 18:08 ..
    ```
3. 进程替换 `<(command)` 让你可以把命令的输出当作文件来使用。用 `diff` 配合进程替换来比较 `printenv` 和 `export` 的输出。它们为什么不同？（提示：试试 `diff <(printenv | sort) <(export | sort)`）。

## 环境变量

1. 编写 bash 函数 `marco` 和 `polo`，实现如下功能：每当你执行 `marco` 时，应以某种方式保存当前工作目录；然后当你执行 `polo` 时，无论你身处哪个目录，`polo` 都应该把你 `cd` 回执行 `marco` 时所在的目录。为了便于调试，你可以把代码写在文件 `marco.sh` 中，并通过执行 `source marco.sh` 把这些定义（重新）加载到你的 shell 中。

## 返回码

1. 假设你有一条很少失败的命令。为了调试它，你需要捕获它的输出，但要等到一次失败的运行可能很费时间。编写一个 bash 脚本，反复运行下面的脚本直到它失败，把它的标准输出流和错误流捕获到文件中，并在最后打印所有内容。如果你还能报告脚本失败前运行了多少次，可以得到额外加分。

   ```bash
    #!/usr/bin/env bash

    n=$(( RANDOM % 100 ))

    if [[ $n -eq 42 ]]; then
       echo "Something went wrong"
       >&2 echo "The error was using magic numbers"
       exit 1
    fi

    echo "Everything went according to plan"
    ```

## 信号与作业控制

1. 在终端中启动一个 `sleep 10000` 作业，用 `Ctrl-Z` 把它放到后台，再用 `bg` 让它继续执行。现在用 [`pgrep`](https://www.man7.org/linux/man-pages/man1/pgrep.1.html) 找到它的 pid，并用 [`pkill`](https://man7.org/linux/man-pages/man1/pgrep.1.html) 杀掉它，全程不输入 pid 本身。（提示：使用 `-lf` 标志）。
2. 假设你希望在一个进程完成之后才启动另一个进程。你会怎么做？在这个练习中，我们的限制进程始终是 `sleep 60 &`。一种实现方式是使用 [`wait`](https://www.man7.org/linux/man-pages/man1/wait.1p.html) 命令。试着启动 sleep 命令并让一个 `ls` 等待后台进程结束。

   然而，如果我们是在另一个 bash 会话中开始的，这个策略就会失败，因为 `wait` 只对子进程有效。讲义中没有讨论的一个特性是：`kill` 命令的退出状态（exit status）成功时为零，否则为非零。`kill -0` 不会发送信号，但如果进程不存在，它会给出非零的退出状态。编写一个名为 `pidwait` 的 bash 函数，它接受一个 pid 并等待给定进程完成。你应该使用 `sleep`，以避免不必要地浪费 CPU。

## 文件与权限

1. （进阶）编写一个命令或脚本，递归地找出一个目录中最近修改的文件。更一般地，你能按新近程度列出所有文件吗？

## 终端复用器

1. 跟随这个 `tmux` [教程](https://www.hamvocke.com/blog/a-quick-and-easy-guide-to-tmux/)，然后按照[这些步骤](https://www.hamvocke.com/blog/a-guide-to-customizing-your-tmux-conf/)学习如何做一些基本的定制。

## 别名与点文件

1. 创建一个别名 `dc`，在你打错字时把它解析为 `cd`。
2. 运行 `history | awk '{$1="";print substr($0,2)}' | sort | uniq -c | sort -n | tail -n 10` 来得到你最常用的 10 个命令，并考虑为它们编写更短的别名。注意：这适用于 Bash；如果你用的是 ZSH，请用 `history 1` 代替 `history`。
3. 为你的点文件创建一个文件夹并设置好版本控制。
4. 为至少一个程序（例如你的 shell）添加一项配置，做一些定制（刚开始时，可以简单到通过设置 `$PS1` 来定制你的 shell 提示符）。
5. 搭建一种在新机器上快速（无需手动操作）安装你的点文件的方法。这可以简单到一个为每个文件调用 `ln -s` 的 shell 脚本，你也可以使用一个[专用工具](https://dotfiles.github.io/utilities/)。
6. 在一台全新的虚拟机上测试你的安装脚本。
7. 把你当前所有的工具配置迁移到你的点文件仓库。
8. 在 GitHub 上发布你的点文件。

## 远程机器（SSH）

为这些练习安装一台 Linux 虚拟机（或使用已有的虚拟机）。如果你不熟悉虚拟机，请看[这](https://hibbard.eu/install-ubuntu-virtual-box/)篇安装教程。

1. 进入 `~/.ssh/` 并检查那里是否有一对 SSH 密钥。如果没有，用 `ssh-keygen -a 100 -t ed25519` 生成它们。建议你使用口令短语并使用 `ssh-agent`，更多信息见[这里](https://www.ssh.com/ssh/agent)。
2. 编辑 `.ssh/config`，加入如下条目：

   ```bash
    Host vm
        User username_goes_here
        HostName ip_goes_here
        IdentityFile ~/.ssh/id_ed25519
        LocalForward 9999 localhost:8888
    ```
3. 用 `ssh-copy-id vm` 把你的 ssh 密钥复制到服务器上。
4. 在你的虚拟机中执行 `python -m http.server 8888` 启动一个 web 服务器。在你的机器上导航到 `http://localhost:9999` 来访问虚拟机上的 web 服务器。
5. 通过执行 `sudo vim /etc/ssh/sshd_config` 编辑你的 SSH 服务器配置，修改 `PasswordAuthentication` 的值来禁用密码认证。修改 `PermitRootLogin` 的值来禁用 root 登录。用 `sudo service sshd restart` 重启 `ssh` 服务。再次尝试 ssh 登录。
6. （挑战）在虚拟机中安装 [`mosh`](https://mosh.org/) 并建立一个连接。然后断开服务器/虚拟机的网络适配器。mosh 能正确地从中恢复吗？
7. （挑战）研究 `ssh` 的 `-N` 和 `-f` 标志是做什么的，并找出一条实现后台端口转发的命令。

---

原文：https://missing.csail.mit.edu/2026/command-line-environment/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
