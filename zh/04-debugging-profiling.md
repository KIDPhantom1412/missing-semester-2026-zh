# 调试与性能分析

编程中的一条黄金法则是：代码做的不是你期望它做的事，而是你告诉它要做的事。弥合这一差距有时是相当困难的挑战。在本讲中，我们将介绍处理有 bug 且资源消耗巨大的代码的实用技术：调试（debugging）和性能分析（profiling）。

# 调试

## Printf 调试与日志

> “最有效的调试工具仍然是缜密的思考，再加上恰到好处放置的打印语句” —— Brian Kernighan，《Unix for Beginners》。

调试程序的第一种方法是在你检测到问题的位置附近添加打印语句，并不断迭代，直到提取出足够的信息来理解是什么导致了该问题。

第二种方法是在程序中使用日志记录（logging），而不是临时的打印语句。日志本质上是“更讲究的打印”，通常通过日志框架来完成，这类框架内置了对以下功能的支持：

* 将日志（或日志的子集）重定向到其他输出位置的能力；
* 设置严重级别（如 INFO、DEBUG、WARN、ERROR 等），并允许你据此过滤输出；以及
* 对与日志条目相关的数据进行结构化日志记录的支持，这些数据事后可以更轻松地提取出来。

你通常还会在编程时主动写好日志语句，这样你需要用来调试的数据可能已经在那里了！确实如此，一旦你用打印语句找到并修复了问题，在删除它们之前，通常值得把这些打印语句转换成规范的日志语句。这样，如果将来出现类似的 bug，你无需修改代码就已经拥有了所需的诊断信息。

> **第三方日志**：许多程序支持 `-v` 或 `--verbose` 标志，以便在运行时打印更多信息。这对于发现某个命令为何失败很有用。有些程序甚至允许重复使用该标志以获取更多细节。在调试服务（数据库、Web 服务器等）的问题时，请查看它们的日志——在 Linux 上通常位于 `/var/log/`。使用 `journalctl -u` 查看 systemd 服务的日志。对于第三方库，检查它们是否支持通过环境变量或配置开启调试日志。

## 调试器

当你知道该打印什么、并且能轻松修改和重新运行代码时，打印调试效果很好。而当你不确定需要哪些信息、bug 只在难以复现的条件下显现，或者修改并重启程序代价高昂（启动时间长、需要重建复杂状态等）时，调试器（debugger）就变得非常有价值。

调试器是让你在程序执行的同时与之交互的程序，它允许你：

* 当执行到达某一行时暂停执行。
* 一次单步执行一条指令。
* 在崩溃后检查变量的值。
* 当满足给定条件时有条件地暂停执行。
* 以及更多高级功能。

大多数编程语言都支持（或自带）某种形式的调试器。最通用的是**通用调试器（general-purpose debuggers）**，如 [`gdb`](https://www.gnu.org/software/gdb/)（GNU 调试器）和 [`lldb`](https://lldb.llvm.org/)（LLVM 调试器），它们可以调试任何本机二进制程序。许多语言还有**特定语言的调试器（language-specific debuggers）**，与运行时集成得更紧密（如 Python 的 pdb 或 Java 的 jdb）。

`gdb` 是 C、C++、Rust 及其他编译型语言的事实标准调试器。它让你能探测几乎任何进程并获取其当前的机器状态：寄存器、栈、程序计数器等等。

一些实用的 GDB 命令：

* `run` - 启动程序
* `b {function}` 或 `b {file}:{line}` - 设置断点（breakpoint）
* `c` - 继续执行
* `step` / `next` / `finish` - 单步进入 / 单步跳过 / 单步跳出
* `p {variable}` - 打印变量的值
* `bt` - 显示回溯（调用栈，backtrace）
* `watch {expression}` - 当值发生变化时中断

> 考虑使用 GDB 的 TUI 模式（`gdb -tui` 或在 GDB 中按 `Ctrl-x a`），以分屏视图同时显示源代码和命令提示符。

### 录制-回放调试

最令人抓狂的一类 bug 是*海森堡 bug（Heisenbugs）*：当你试图观察它们时，它们似乎会消失或改变行为。竞态条件（race condition）、时序相关的 bug，以及只在特定系统条件下出现的问题都属于这一类。传统调试在这里往往无济于事，因为再次运行程序会产生不同的行为（例如，打印语句可能会让代码慢到竞态不再发生）。

**录制-回放调试（record-replay debugging）**通过录制程序的执行过程并允许你按需确定性地回放任意多次来解决这个问题。更棒的是，你可以*反向*回溯执行过程，准确找到出问题的位置。

[rr](https://rr-project.org/) 是 Linux 上一个强大的工具，它录制程序执行并允许带完整调试能力的确定性回放。它与 GDB 协同工作，所以你已经熟悉它的界面。

基本用法：

```
# Record a program execution# Replay the recording (opens GDB)
```

神奇之处发生在回放期间。由于执行是确定性的，你可以使用**反向调试（reverse debugging）**命令：

* `reverse-continue` (`rc`) - 反向运行，直到命中一个断点
* `reverse-step` (`rs`) - 反向单步一行
* `reverse-next` (`rn`) - 反向单步，跳过函数调用
* `reverse-finish` - 反向运行，直到进入当前函数

这对调试来说威力无比。假设你遇到了一次崩溃——与其猜测 bug 在哪里并设置断点，你可以：

1. 运行到崩溃处
2. 检查被破坏的状态
3. 在被破坏的变量上设置一个观察点（watchpoint）
4. 用 `reverse-continue` 准确找出它是在哪里被破坏的

**何时使用 rr：**

* 间歇性失败的不稳定测试（flaky tests）
* 竞态条件和多线程 bug
* 难以复现的崩溃
* 任何让你希望能让“时间倒流”的 bug

> 注意：rr 仅在 Linux 上工作，并且需要硬件性能计数器。在不暴露这些计数器的虚拟机中（例如大多数 AWS EC2 实例）它无法工作，而且它不支持 GPU 访问。对于 macOS，请看看 [Warpspeed](https://warpspeed.dev/)。

> **rr 与并发**：由于 rr 以确定性方式录制执行，它会将线程调度串行化。这意味着某些竞态条件在 rr 下可能不会显现（如果它们依赖特定的时序）。rr 对调试竞态仍然有用——一旦你捕获到一次失败的运行，就可以可靠地回放它——但要捕捉一个间歇性 bug 可能需要多次录制尝试。对于不涉及并发的 bug，rr 的表现最为出色：你总能复现精确的执行过程，并用反向调试来追查破坏发生的位置。

## 系统调用追踪

有时你需要理解你的程序如何与操作系统交互。程序通过[系统调用](https://en.wikipedia.org/wiki/System_call)（system calls）向内核请求服务——打开文件、分配内存、创建进程等等。追踪这些调用可以揭示程序为何挂起、它正试图访问哪些文件，或者它把时间花在等待什么上。

### strace (Linux) 和 dtruss (macOS)

[`strace`](https://www.man7.org/linux/man-pages/man1/strace.1.html) 让你观察程序发出的每一个系统调用：

```
# Trace all system calls# Trace only file-related calls -e trace =# Follow child processes (important for programs that start other programs) -f # Trace a running process -p # Show timing information -T
```

> 在 macOS 和 BSD 上，使用 [`dtruss`](https://www.manpagez.com/man/1/dtruss/)（它封装了 `dtrace`）来获得类似的功能：
> 想更深入地了解 `strace`，请看 Julia Evans 出色的 [strace 小册子](https://jvns.ca/strace-zine-unfolded.pdf)。

### bpftrace 与 eBPF

[eBPF](https://ebpf.io/)（extended Berkeley Packet Filter，扩展伯克利数据包过滤器）是一项强大的 Linux 技术，允许在内核中运行沙箱化程序。[`bpftrace`](https://github.com/iovisor/bpftrace) 为编写 eBPF 程序提供了高层次的语法。这些是运行在内核中的任意程序，因而具有巨大的表达能力（尽管语法有点像 awk，略显笨拙）。它们最常见的用例是调查正在调用哪些系统调用，包括做聚合（如计数或延迟统计），或内省（甚至基于其过滤）系统调用参数。

```
# Trace file opens system-wide (prints immediately) sudo -e'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'# Count system calls by name (prints summary on Ctrl-C) sudo -e'tracepoint:syscalls:sys_enter_* { @[probe] = count(); }'
```

不过，你也可以使用 [`bcc`](https://github.com/iovisor/bcc) 这样的工具链直接用 C 编写 eBPF 程序，它还附带[许多实用工具](https://www.brendangregg.com/blog/2015-09-22/bcc-linux-4.3-tracing.html)，比如用于打印磁盘操作延迟分布的 `biosnoop`，或用于打印所有打开文件的 `opensnoop`。

`strace` 的好处是“开箱即用”非常容易，而当你需要更低的开销、想要追踪内核函数、需要做任何类型的聚合等等时，就应该选用 `bpftrace`。注意 `bpftrace` 必须以 `root` 身份运行，而且它通常监控整个内核，而不仅仅是某个特定进程。要针对特定程序，你可以按命令名或 PID 进行过滤：

```
# Filter by command name (prints summary on Ctrl-C) sudo -e'tracepoint:syscalls:sys_enter_* /comm == "bash"/ { @[probe] = count(); }'# Trace a specific command from startup using -c (cpid = child PID) sudo -e'tracepoint:syscalls:sys_enter_* /pid == cpid/ { @[probe] = count(); }' -c'ls -la'
```

`-c` 标志会运行指定的命令并将 `cpid` 设置为该命令的 PID，这对于从程序启动那一刻起开始追踪非常有用。当被追踪的命令退出时，bpftrace 会打印聚合结果。

### 网络调试

对于网络问题，[`tcpdump`](https://www.man7.org/linux/man-pages/man1/tcpdump.1.html) 和 [Wireshark](https://www.wireshark.org/) 让你捕获并分析网络数据包：

```
# Capture packets on port 80 sudo -i # Capture and save to file for Wireshark analysis sudo -i -w
```

对于 HTTPS 流量，加密使得 tcpdump 不那么好用。像 [mitmproxy](https://mitmproxy.org/) 这样的工具可以作为拦截代理来检查加密流量。浏览器开发者工具（Network/网络 标签页）通常是调试来自 Web 应用的 HTTPS 请求最简单的方式——它们会显示解密后的请求/响应数据、头部和时序信息。

## 内存调试

内存 bug——缓冲区溢出、释放后使用（use-after-free）、内存泄漏——是最危险且最难调试的问题之一。它们往往不会立即崩溃，而是以导致很久之后才出现问题的方式破坏内存。

### 检测器（Sanitizers）

查找内存 bug 的一种方法是使用**检测器（sanitizers）**，它们是编译器特性，会对你的代码进行插桩以便在运行时检测错误。例如，广泛使用的 **AddressSanitizer (ASan)** 可以检测：

* 缓冲区溢出（栈、堆和全局区）
* 释放后使用（use-after-free）
* 返回后使用（use-after-return）
* 内存泄漏

```
# Compile with AddressSanitizer -fsanitize = -g -o
```

有多种实用的检测器：

* **ThreadSanitizer (TSan)**：检测多线程代码中的数据竞争（`-fsanitize=thread`）
* **MemorySanitizer (MSan)**：检测对未初始化内存的读取（`-fsanitize=memory`）
* **UndefinedBehaviorSanitizer (UBSan)**：检测未定义行为，如整数溢出（`-fsanitize=undefined`）

检测器需要重新编译，但速度足够快，可以在 CI 流水线和日常开发中使用。

### Valgrind：当你无法重新编译时

[Valgrind](https://valgrind.org/) 则是把你的程序运行在一个类似虚拟机的环境中来检测内存错误。它比检测器慢，但不需要重新编译：

```
--leak-check =
```

在以下情况下使用 Valgrind：

* 你没有源代码
* 你无法重新编译（第三方库）
* 你需要检测器无法提供的特定工具

Valgrind 实际上是一个非常强大的受控执行环境，等我们讲到性能分析时会更多地看到它！

## 用 AI 调试

大型语言模型已经成为出人意料有用的调试助手。它们擅长某些能与传统工具互补的调试任务。

**LLM 擅长的方面：**

* **解释晦涩的错误信息**：编译器错误，尤其是来自 C++ 模板或 Rust 借用检查器的错误，可能出了名地晦涩。LLM 可以把它们翻译成平实的语言并建议修复方法。
* **跨越语言和抽象边界**：如果你在调试一个横跨多种语言的问题（比如说，一个 C 库中的 bug 通过 Python 绑定显现出来），LLM 可以帮助你理清不同的层次。它们尤其擅长理解 FFI 边界、构建系统问题和跨语言调试（例如：我的程序报错了，但我认为这是我的某个依赖项中的 bug 导致的）。
* **将症状与根因关联起来**：“我的程序工作正常，但使用的内存比预期多 10 倍”就是那种 LLM 可以帮助调查的模糊症状，它们会建议可能的原因以及该查找什么。
* **分析崩溃转储和栈回溯**：粘贴一段栈回溯，然后询问可能是什么导致了它。

> **关于调试符号的说明**：为了获得有意义的栈回溯和调试体验，请确保你的二进制文件（以及任何链接的库）都带有调试符号编译（`-g` 标志）。调试信息通常以 DWARF 格式存储。此外，编译时保留帧指针（`-fno-omit-frame-pointer`）会让栈回溯更加可靠，对性能分析工具尤其如此。没有这些，栈回溯可能只会显示内存地址或不完整。这一点对原生编译的程序（C++、Rust）比对 Python 或 Java 更重要。

**需要牢记的局限：**

* LLM 可能会编造听起来合理但错误的解释
* 它们可能提出的修复方案会掩盖 bug 而不是修复它
* 务必用实际的调试工具验证这些建议
* 它们最好作为理解代码的补充，而不是替代品

> 这与开发环境一讲中介绍的[通用 AI 编码能力](/2026/development-environment/#ai-powered-development)不同。这里我们专门讨论将 LLM 用作调试辅助工具。

# 性能分析

即使你的代码在功能上表现符合预期，如果它在此过程中占用了你全部的 CPU 或内存，那可能还不够好。算法课往往只教大 *O* 表示法，却不教如何找到程序中的热点。由于[过早优化是万恶之源](https://wiki.c2.com/?PrematureOptimization)，你应该了解性能分析器（profilers）和监控工具。它们会帮助你理解程序的哪些部分占用了最多的时间和/或资源，从而让你可以专注于优化这些部分。

## 计时

测量性能最简单的方法是计时。在许多场景下，只需打印你的代码在两个点之间所花的时间就足够了。

不过，墙钟时间可能具有误导性，因为你的计算机可能同时在运行其他进程或在等待事件发生。`time` 命令区分 *Real*、*User* 和 *Sys* 时间：

* **Real** - 从开始到结束的墙钟时间，包括花在等待上的时间
* **User** - CPU 花在运行用户代码上的时间
* **Sys** - CPU 花在运行内核代码上的时间

```
$ time 
```

这里请求花了将近 300 毫秒（真实时间），但 CPU 时间只有 107 毫秒（user + sys）。其余时间都在等待网络。

## 资源监控

有时，分析程序性能的第一步是了解它的实际资源消耗是多少。程序在资源受限时常常运行缓慢。

* **通用监控**：[`htop`](https://htop.dev/) 是 `top` 的改进版本，它展示当前正在运行的进程的各种统计信息。实用快捷键： 对进程排序，`t` 显示树状层级，`h` 切换线程显示。还有 [`btop`](https://github.com/aristocratos/btop)，它监控的东西*多*得多。
* **I/O 操作**：[`iotop`](https://www.man7.org/linux/man-pages/man8/iotop.8.html) 显示实时的 I/O 使用信息。
* **内存使用**：[`free`](https://www.man7.org/linux/man-pages/man1/free.1.html) 显示空闲和已用内存的总量。
* **打开的文件**：[`lsof`](https://www.man7.org/linux/man-pages/man8/lsof.8.html) 列出进程所打开文件的相关信息。适合用来查看哪个进程打开了某个特定文件。
* **网络连接**：[`ss`](https://www.man7.org/linux/man-pages/man8/ss.8.html) 让你监控网络连接。一个常见用例是弄清哪个进程在使用某个给定端口：`ss -tlnp | grep :8080`。
* **网络使用**：[`nethogs`](https://github.com/raboof/nethogs) 和 [`iftop`](https://pdw.ex-parrot.com/iftop/) 是按进程监控网络使用情况的优秀交互式命令行工具。

## 可视化性能数据

人类在图表中发现模式的速度比在数字表格中快得多。分析性能时，把数据绘制成图往往能揭示在原始数字中看不见的趋势、尖峰和异常。

**让数据变得可绘图**：在为调试添加打印或日志语句时，不妨把输出格式化成日后易于绘图的形式。CSV 格式的简单时间戳和数值（`1705012345,42.5`）比一句散文好画得多。JSON 结构化的日志也可以用极少的精力解析并绘图。换句话说，[以整饬的方式](https://vita.had.co.nz/papers/tidy-data.pdf)记录你的数据。

**用 gnuplot 快速绘图**：对于简单的命令行绘图，[`gnuplot`](http://www.gnuplot.info/) 可以直接从数据文件生成图表：

```
# Plot a simple CSV with timestamp,value -e"set datafile separator ','; plot 'latency.csv' using 1:2 with lines"
```

**用 matplotlib 和 ggplot2 进行迭代式探索**：对于更深入的分析，Python 的 [`matplotlib`](https://matplotlib.org/) 和 R 的 [`ggplot2`](https://ggplot2.tidyverse.org/) 支持迭代式探索。与一次性绘图不同，这些工具让你可以快速对数据进行切片和转换以验证假设。ggplot2 的分面图（facet plots）尤其强大——你可以按类别把一个数据集拆分到多个子图中（例如，按端点或一天中的时段对请求延迟进行分面），从而揭示原本会被掩盖的模式。

**示例用例：**

* 绘制请求延迟随时间变化的图，可以揭示周期性的变慢（垃圾回收、cron 任务、流量模式），而原始的百分位数会掩盖这些现象
* 可视化一个不断增长的数据结构的插入耗时，可以暴露算法复杂度问题——向量（vector）插入的图会在底层数组倍增扩容时显示出典型的尖峰
* 按不同维度（请求类型、用户群组、服务器）对指标进行分面，往往能揭示所谓“全局性”的问题其实只局限于某一个类别

## CPU 性能分析器

大多数时候人们说*性能分析器（profilers）*时，指的都是 *CPU 性能分析器*。主要有两种类型：

* **追踪式分析器（tracing profilers）**记录你的程序发出的每一次函数调用
* **采样式分析器（sampling profilers）**定期（通常每毫秒一次）探测你的程序并记录程序的栈

采样式分析器开销更低，通常更适合在生产环境中使用。

### perf：采样式分析器

[`perf`](https://www.man7.org/linux/man-pages/man1/perf.1.html) 是 Linux 的标准性能分析器。它无需重新编译即可对任何程序做性能分析：

`perf stat` 让你快速了解时间花在了哪里：

```
$ stat for'./slow_program'# 0.998 CPUs utilized# 3.738 /sec# 0.000 /sec# 48.587 /sec# 3.845 GHz# 0.80 insn per cycle# 384.532 M/sec# 1.00% of all branches
```

对真实世界程序的性能分析输出会包含大量信息。人类是视觉动物，很不擅长阅读大量数字。[火焰图（flame graphs）](https://www.brendangregg.com/flamegraphs.html)是一种能让性能分析数据更容易理解的可视化方式。

火焰图在 Y 轴上显示函数调用的层级结构，X 轴上占用的宽度与所耗时间成正比。它们是交互式的——你可以点击以放大到程序的特定部分。

要从 `perf` 数据生成火焰图：

```
# Record profile -g# Generate flame graph (requires flamegraph scripts)>
```

> 考虑使用 [Speedscope](https://www.speedscope.app/) 作为基于 Web 的交互式火焰图查看器，或使用 [Perfetto](https://perfetto.dev/) 进行全面的系统级分析。

### Valgrind 的 Callgrind：追踪式分析器

[`callgrind`](https://valgrind.org/docs/manual/cl-manual.html) 是一个性能分析工具，它记录你程序的调用历史和指令计数。与采样式分析器不同，它提供精确的调用次数，并能显示调用方与被调用方之间的关系：

```
# Run with callgrind --tool =# Analyze with callgrind_annotate (text) or kcachegrind (GUI)
```

Callgrind 比采样式分析器慢，但能提供精确的调用次数，而且如果你需要相关信息，它还可以选择性地模拟缓存行为（使用 `--cache-sim=yes`）。

> 如果你使用的是某种特定的语言，可能会有更专业的性能分析器。例如，Python 有 [`cProfile`](https://docs.python.org/3/library/profile.html) 和 [`py-spy`](https://github.com/benfred/py-spy)，Go 有 [`go tool pprof`](https://pkg.go.dev/cmd/pprof)，Rust 有 [`cargo-flamegraph`](https://github.com/flamegraph-rs/flamegraph)（它实际上适用于任何编译型程序！）。

## 内存性能分析器

内存性能分析器帮助你理解程序随时间如何使用内存，并找出内存泄漏。

### Valgrind 的 Massif

[`massif`](https://valgrind.org/docs/manual/ms-manual.html) 对堆内存使用情况进行性能分析：

```
--tool =
```

它会向你展示堆使用量随时间的变化，帮助识别内存泄漏和过度分配。

> 对于 Python，[`memory-profiler`](https://pypi.org/project/memory-profiler/) 提供逐行的内存使用信息。

## 基准测试

当你需要比较不同实现或工具的性能时，[`hyperfine`](https://github.com/sharkdp/hyperfine) 是对命令行程序进行基准测试（benchmarking）的绝佳工具：

```
$ --warmup'fd -e jpg''find . -iname "*.jpg"'#1: fd -e jpg()[()#2: find . -iname "*.jpg"()[()'fd -e jpg' times 'find . -iname "*.jpg"'
```

> 对于 Web 开发，浏览器开发者工具自带出色的性能分析器。参见 [Firefox Profiler](https://profiler.firefox.com/docs/) 和 [Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/rendering-tools) 的文档。

# 练习

## 调试

1. **调试一个排序算法**：以下伪代码实现了归并排序（merge sort），但包含一个 bug。用你选择的语言实现它，然后使用调试器（gdb、lldb、pdb 或你 IDE 的调试器）找到并修复这个 bug。

   ```
   function merge_sort(arr): if length(arr) <= 1: return arr mid = length(arr) / 2 left = merge_sort(arr[0..mid]) right = merge_sort(arr[mid..end]) return merge(left, right) function merge(left, right): result = [] i = 0, j = 0 while i < length(left) AND j < length(right): if left[i] <= right[j]: append result, left[i] i = i + 1 else: append result, right[i] j = j + 1 append remaining elements from left and right return result 
   ```

   测试向量：`merge_sort([3, 1, 4, 1, 5, 9, 2, 6])` 应该返回 `[1, 1, 2, 3, 4, 5, 6, 9]`。使用断点并单步跟踪 merge 函数，找出错误的元素是在哪里被选中的。
2. 安装 [`rr`](https://rr-project.org/) 并使用反向调试来找出一个内存破坏 bug。将以下程序保存为 `corruption.c`：

   ```
   #include  typedef struct{int id; int scores[3];} Student; Student students[2]; void init(){students[0]. id = 1001; students[0]. scores[0] = 85; students[0]. scores[1] = 92; students[0]. scores[2] = 78; students[1]. id = 1002; students[1]. scores[0] = 90; students[1]. scores[1] = 88; students[1]. scores[2] = 95;} void curve_scores(int student_idx, int curve){for(int i = 0; i< 4; i ++){students[student_idx]. scores[i] += curve;}} int main(){init(); printf("=== Initial state === \n "); printf("Student 0: id=%d \n ", students[0]. id); printf("Student 1: id=%d \n ", students[1]. id); curve_scores(0, 5); printf(" \n === After curving === \n "); printf("Student 0: id=%d \n ", students[0]. id); printf("Student 1: id=%d \n ", students[1]. id); if(students[1]. id!= 1002){printf(" \nERROR: Student 1's ID was corrupted! Expected 1002, got %d \n ", students[1]. id); return 1;} return 0;}
   ```

   用 `gcc -g corruption.c -o corruption` 编译并运行它。学生 1 的 ID 被破坏了，但破坏发生在一个只操作学生 0 的函数中。使用 `rr record ./corruption` 和 `rr replay` 找出罪魁祸首。在 `students[1].id` 上设置一个观察点，并在破坏发生后使用 `reverse-continue`，准确找出是哪一行代码覆写了它。
3. 用 AddressSanitizer 调试一个内存错误。将以下内容保存为 `uaf.c`：

   ```
   #include  #include  #include  int main(){char* greeting = malloc(32); strcpy(greeting,"Hello, world!"); printf("%s \n ", greeting); free(greeting); greeting[0] = 'J'; printf("%s \n ", greeting); return 0;}
   ```

   首先在不使用检测器的情况下编译并运行：`gcc uaf.c -o uaf && ./uaf`。它可能看起来工作正常。现在用 AddressSanitizer 编译：`gcc -fsanitize=address -g uaf.c -o uaf && ./uaf`。阅读错误报告。ASan 发现了什么 bug？修复它所指出的问题。
4. 使用 `strace`（Linux）或 `dtruss`（macOS）追踪 `ls -l` 这样的命令发出的系统调用。它做了哪些系统调用？试着追踪一个更复杂的程序，看看它打开了哪些文件。
5. 用 LLM 来帮助调试一条晦涩的错误信息。试着把一条编译器错误（尤其是来自 C++ 模板或 Rust 的）复制给它，请求解释和修复。也可以试着把 `strace` 或地址检测器的部分输出放进去。

## 性能分析

1. 用 `perf stat` 获取你选择的某个程序的基本性能统计。各个计数器都是什么意思？
2. 用 `perf record` 做性能分析。将以下内容保存为 `slow.c`：

   ```
   #include  #include  double slow_computation(int n){double result = 0; for(int i = 0; i< n; i ++){for(int j = 0; j< 1000; j ++){result += sin(i* j)* cos(i + j);}} return result;} int main(){double r = 0; for(int i = 0; i< 100; i ++){r += slow_computation(1000);} printf("Result: %f \n ", r); return 0;}
   ```

   带调试符号编译：`gcc -g -O2 slow.c -o slow -lm`。运行 `perf record -g ./slow`，然后用 `perf report` 查看时间花在哪里。试着用 flamegraph 脚本生成一个火焰图。
3. 用 `hyperfine` 对同一任务的两种不同实现进行基准测试（例如 `find` 与 `fd`、`grep` 与 `ripgrep`，或者你自己代码的两个版本）。
4. 在运行一个资源密集型程序时用 `htop` 监控你的系统。试着用 `taskset` 限制一个进程可以使用哪些 CPU：`taskset --cpu-list 0,2 stress -c 3`。为什么 `stress` 没有用到三个 CPU？
5. 一个常见的问题是你想监听的端口已经被另一个进程占用。学会如何发现该进程：首先执行 `python -m http.server 4444`，在端口 4444 上启动一个最小的 Web 服务器。在另一个终端里运行 `ss -tlnp | grep 4444` 找到该进程。用 `kill` 终止它。

---

原文：https://missing.csail.mit.edu/2026/debugging-profiling/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
