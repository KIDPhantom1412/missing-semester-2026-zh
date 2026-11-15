# 版本控制与 Git

版本控制系统（VCS）是用于跟踪源代码（或其他文件与文件夹集合）变更的工具。顾名思义，这些工具帮助维护一份变更历史；此外，它们还便于协作。从逻辑上看，VCS 通过一系列*快照*（snapshot）来跟踪某个文件夹及其内容的变更，每个快照封装了某个顶层目录下文件/文件夹的完整状态。VCS 还会维护一些元数据，比如每个快照是谁创建的、与每个快照关联的消息，等等。

为什么版本控制很有用？即使你独自工作，它也能让你查看项目的历史快照、记录某些变更的原因、在并行的开发分支上工作，等等。在与他人协作时，它是一件不可或缺的工具，既能让你看到其他人改了什么，也能帮助解决并发开发中的冲突。

现代 VCS 还能让你轻松（而且往往是自动地）回答诸如以下问题：

* 这个模块是谁写的？
* 这个文件的这一行是什么时候编辑的？由谁编辑？为什么编辑？
* 在最近 1000 个修订版本中，某个单元测试是何时/为何失效的？

尽管还有其他 VCS，**Git** 是事实上的（de facto）版本控制标准。这幅 [XKCD 漫画](https://xkcd.com/1597/)刻画了 Git 的名声：

由于 Git 的接口是一种存在泄漏的抽象（leaky abstraction），自顶向下地学习 Git（从它的接口/命令行界面开始）可能会带来很多困惑。人们可以 memorize 少数几条命令并把它们当作魔法咒语，然后每当出问题时就按照上面漫画中的方法来处理。

虽然 Git 的接口确实丑陋，但其底层的设计和理念却是优美的。丑陋的接口只能靠*死记硬背*，而优美的设计是可以被*理解*的。正因如此，我们将自底向上地讲解 Git，先从它的数据模型开始，之后再介绍命令行界面。一旦理解了数据模型，就能更好地理解各个命令是如何操纵底层数据模型的。

# Git 的数据模型

Git 的巧妙之处在于它精心设计的数据模型，正是这个模型支撑了版本控制的各种出色特性，比如维护历史、支持分支以及实现协作。

## 快照

Git 将某个顶层目录下的一组文件和文件夹的历史建模为一系列快照。在 Git 的术语中，一个文件被称为“blob”，它就是一堆字节。一个目录被称为“tree”，它将名字映射到 blob 或 tree（因此目录可以包含其他目录）。快照就是被跟踪的顶层 tree。例如，我们可能有如下的一棵树：

```
 (tree) | +- foo (tree) | | | + bar.txt (blob, contents = \"hello world\") | +- baz.txt (blob, contents = \"git is wonderful\") 
```

顶层 tree 包含两个元素：一个 tree “foo”（它本身包含一个元素，即 blob “bar.txt”），以及一个 blob “baz.txt”。

## 为历史建模：关联快照

版本控制系统应该如何关联各个快照？一种简单的模型是线性历史。历史就是按时间顺序排列的快照列表。由于种种原因，Git 没有采用这样简单的模型。

在 Git 中，历史是一个由快照构成的有向无环图（DAG，directed acyclic graph）。这听起来像是一个花哨的数学术语，但不必被吓到。它的全部含义是：Git 中的每个快照都指向一组“父快照”（parents），也就是在它之前产生的快照。之所以是“一组”父快照而不是单个父快照（线性历史就是单个父快照），是因为一个快照可能有多个父快照，例如，当合并（merging）两条并行的开发分支时就会如此。

Git 把这些快照称为“提交”（commit）。把提交历史可视化出来大概长这样：

```
o <-- o <-- o <-- o ^ \\ --- o <-- o 
```

在上面的 ASCII 图中，每个 `o` 对应一个单独的提交（快照）。箭头指向每个提交的父提交（这是一种“在……之前”的关系，而不是“在……之后”）。在第三个提交之后，历史分叉成两条独立的分支。这可能对应于，比如两个相互独立、并行开发的特性。将来，这些分支可以被合并，创建出一个同时包含这两个特性的新快照，产生如下所示的新历史，其中新创建的合并提交以粗体显示：

```
 o <-- o <-- o <-- o <---- o ^ / \\ v --- o <-- o  
```

Git 中的提交是不可变的。但这并不意味着错误无法修正；只不过对提交历史的“编辑”实际上是创建了全新的提交，并将引用（reference，见下文）更新为指向这些新提交。

## 用伪代码表示数据模型

用伪代码把 Git 的数据模型写出来也许颇有启发：

```
// a file is a bunch of bytes type blob = array // a directory contains named files and directories type tree = map // a commit has parents, metadata, and the top-level tree type commit = struct { parents: array author: string message: string snapshot: tree } 
```

这是一个干净、简单的历史模型。

## 对象与内容寻址

一个“对象”（object）是 blob、tree 或 commit：

```
type object = blob | tree | commit 
```

在 Git 的数据存储中，所有对象都通过其 [SHA-1 哈希](https://en.wikipedia.org/wiki/SHA-1)进行内容寻址（content-addressed）。

```
objects = map def store(object): id = sha1(object) objects[id] = object def load(id): return objects[id] 
```

Blob、tree 和 commit 以这种方式被统一起来：它们全都是对象。当它们引用其他对象时，在磁盘上的表示中并不真正*包含*这些对象，而是通过哈希持有一个指向它们的引用。

例如，[上文](#snapshots)示例目录结构对应的 tree（使用 `git cat-file -p 698281bc680d1995c5f4caaf3359721a5a58d48d` 可视化）如下所示：

```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85 baz.txt 040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87 foo 
```

这个 tree 本身包含指向其内容的指针：`baz.txt`（一个 blob）和 `foo`（一个 tree）。如果我们用 `git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85` 查看 baz.txt 对应哈希所寻址的内容，会得到：

```
git is wonderful 
```

## 引用

现在，所有的快照都可以用它们的 SHA-1 哈希来标识。这很不方便，因为人类并不擅长记住 40 个十六进制字符组成的字符串。

Git 解决这个问题的办法是为 SHA-1 哈希起人类可读的名字，称为“引用”（reference）。引用是指向提交的指针。与不可变的对象不同，引用是可变的（可以更新为指向新的提交）。例如，`master` 引用通常指向开发主分支上的最新提交。

```
references = map def update_reference(name, id): references[name] = id def read_reference(name): return references[name] def load_reference(name_or_id): if name_or_id in references: return load(references[name_or_id]) else: return load(name_or_id) 
```

有了引用，Git 就可以用像 “master” 这样人类可读的名字来指代历史中的某个特定快照，而不必使用冗长的十六进制字符串。

有一个细节是，我们经常需要知道“当前处于历史中的什么位置”，这样当我们创建新快照时，才知道它是相对于哪个快照的（即如何设置提交的 `parents` 字段）。在 Git 中，这个“当前所在位置”是一个名为 “HEAD” 的特殊引用。

## 仓库

最后，我们可以定义（大致上）什么是一个 Git *仓库*（repository）：它就是数据 `objects` 和 `references`。

在磁盘上，Git 存储的全部内容就是对象和引用：Git 的数据模型仅此而已。所有 `git` 命令都对应于对提交 DAG 的某种操纵，即添加对象以及添加/更新引用。

无论何时输入任何命令，都请想一想这个命令对底层的图数据结构做了怎样的操纵。反过来，如果你想对提交 DAG 做某种特定的更改，比如“丢弃未提交的更改，并让 ‘master’ 引用指向提交 `5d83f9e`”，那么很可能有一条命令可以做到（例如在这个例子中是 `git checkout master; git reset --hard 5d83f9e`）。

# 暂存区

这是另一个与数据模型正交的概念，但它是创建提交的接口的一部分。

要实现上文所述的快照功能，你可能会设想这样一种方式：提供一个“创建快照”命令，基于工作目录的*当前状态*创建新快照。有些版本控制工具就是这样工作的，但 Git 不是。我们想要干净的快照，而根据当前状态来创建快照并不总是理想的。例如，设想这样一个场景：你实现了两个独立的特性，而你想创建两个独立的提交，第一个提交引入第一个特性，下一个提交引入第二个特性。或者设想这样一个场景：你在代码各处加了很多调试用的 print 语句，同时还做了一个漏洞修复；你想提交这个修复，但丢弃所有的 print 语句。

Git 通过一种叫做“暂存区”（staging area）的机制来适应这些场景，它允许你指定哪些修改应该被纳入下一个快照。

# Git 命令行界面

为避免重复信息，我们不打算在这份讲义中详细解释下列命令。更多信息请参见强烈推荐的 [Pro Git](https://git-scm.com/book/en/v2)，或者观看课程视频。

## 基础

* `git help` ：获取某个 git 命令的帮助
* `git init`：创建一个新的 git 仓库，数据存储在 `.git` 目录中
* `git status`：告诉你当前的状态
* `git add` ：将文件添加到暂存区
* `git commit`：创建一个新的提交
  + 写[好的提交信息](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)！
  + 还有更多理由去写[好的提交信息](https://chris.beams.io/posts/git-commit/)！
* `git log`：显示扁平化的历史日志
* `git log --all --graph --decorate`：以 DAG 的形式可视化历史
* `git diff` ：显示你相对于暂存区所做的更改
* `git diff` ：显示某个文件在快照之间的差异
* `git checkout` ：更新 HEAD（如果检出的是一个分支，还会更新当前分支）

## 分支与合并

* `git branch`：显示分支
* `git branch` ：创建分支
* `git switch` ：切换到某个分支
* `git checkout -b` ：创建分支并切换到它
  + 等同于 `git branch ; git switch`
* `git merge` ：合并到当前分支
* `git mergetool`：使用一个好用的工具来帮助解决合并冲突
* `git rebase`：将一组补丁变基（rebase）到新的基底上

## 远程

* `git remote`：列出远程
* `git remote add` ：添加一个远程
* `git push  :`：将对象发送到远程，并更新远程引用
* `git branch --set-upstream-to=/`：在本地分支与远程分支之间建立对应关系
* `git fetch`：从远程获取对象/引用
* `git pull`：等同于 `git fetch; git merge`
* `git clone`：从远程下载仓库

## 撤销

* `git commit --amend`：编辑某个提交的内容/信息
* `git reset` ：将某个文件移出暂存区
* `git restore`：丢弃更改

# Git 进阶

* `git config`：Git [高度可定制](https://git-scm.com/docs/git-config)
* `git clone --depth=1`：浅克隆，不包含完整的版本历史
* `git add -p`：交互式暂存
* `git rebase -i`：交互式变基
* `git blame`：显示每一行最后是由谁编辑的
* `git stash`：临时移除对工作目录的修改
* `git bisect`：对历史进行二分查找（例如查找引入回归的提交）
* `git revert`：创建一个新提交，用来撤销某个早期提交的效果
* `git worktree`：同时检出多个分支
* `.gitignore`：[指定](https://git-scm.com/docs/gitignore)故意不跟踪、要忽略的文件

# 杂项

* **GUI**：Git 有很多 [GUI 客户端](https://git-scm.com/downloads/guis)。我们个人并不使用它们，而是使用命令行界面。
* **Shell 集成**：把 Git 状态作为 shell 提示符的一部分非常方便（[zsh](https://github.com/olivierverdier/zsh-git-prompt)、[bash](https://github.com/magicmonty/bash-git-prompt)）。这通常包含在 [Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh) 之类的框架中。
* **编辑器集成**：与上一条类似，是带有很多功能的便捷集成。[fugitive.vim](https://github.com/tpope/vim-fugitive) 是 Vim 的标准选择。
* **工作流**：我们教了你数据模型和一些基本命令；但我们没有告诉你在大型项目中工作时应遵循哪些实践（而且有[很多](https://nvie.com/posts/a-successful-git-branching-model/)[不同](https://www.endoflineblog.com/gitflow-considered-harmful)[的](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)[方法](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow)）。
* **GitHub**：Git 不是 GitHub。GitHub 有一种向其他项目贡献代码的特定方式，叫做[拉取请求](https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests)（pull request）。
* **其他 Git 服务商**：GitHub 并不特殊：有很多 Git 仓库托管服务，比如 [GitLab](https://about.gitlab.com/) 和 [BitBucket](https://bitbucket.org/)。

# 资源

* [Pro Git](https://git-scm.com/book/en/v2) 是**强烈推荐阅读**的书。既然你已经理解了数据模型，读完第 1–5 章应该就能学到熟练使用 Git 所需的大部分知识。后面的章节有一些有趣的进阶内容。
* [Oh Shit, Git!?!](https://ohshitgit.com/) 是一份简短的指南，讲如何从一些常见的 Git 错误中恢复。
* [Git for Computer Scientists](https://eagain.net/articles/git-for-computer-scientists/) 是对 Git 数据模型的简短讲解，与这份讲义相比，伪代码更少，精美图表更多。
* [Git from the Bottom Up](https://jwiegley.github.io/git-from-the-bottom-up/) 详细解释了数据模型之外的 Git 实现细节，适合好奇的读者。
* [How to explain git in simple words](https://smusamashah.github.io/blog/2017/10/14/explain-git-in-simple-words)（如何用简单的话解释 git）
* [Learn Git Branching](https://learngitbranching.js.org/) 是一个基于浏览器的游戏，教你学习 Git。

# 练习

1. 如果你之前没有任何 Git 经验，可以试着读一读 [Pro Git](https://git-scm.com/book/en/v2) 的前几章，或者走一遍 [Learn Git Branching](https://learngitbranching.js.org/) 这样的教程。在学习的过程中，把 Git 命令与数据模型联系起来。
2. 克隆[课程网站的仓库](https://github.com/missing-semester/missing-semester)。
   1. 将版本历史可视化为图，探索它。
   2. 最后一个修改 `README.md` 的人是谁？（提示：给 `git log` 加一个参数。）
   3. `_config.yml` 中 `collections:` 这一行最后一次修改对应的提交信息是什么？（提示：使用 `git blame` 和 `git show`。）
3. 学习 Git 时一个常见的错误是提交不该由 Git 管理的大文件，或者添加敏感信息。试着往仓库中添加一个文件，做几次提交，然后把该文件从*历史*中删除（而不仅仅是最后一个提交）。你可能需要看看[这篇文档](https://help.github.com/articles/removing-sensitive-data-from-a-repository/)。
4. 从 GitHub 上克隆任意一个仓库，修改其中一个已有文件。执行 `git stash` 时会发生什么？运行 `git log --all --oneline` 时你会看到什么？运行 `git stash pop` 来撤销你用 `git stash` 所做的操作。这在什么场景下会有用？
5. 和许多命令行工具一样，Git 提供一个名为 `~/.gitconfig` 的配置文件（也叫点文件，dotfile）。在 `~/.gitconfig` 中创建一个别名，使得运行 `git graph` 时输出 `git log --all --graph --decorate --oneline` 的结果。你可以直接[编辑](https://git-scm.com/docs/git-config#Documentation/git-config.txt-alias) `~/.gitconfig` 文件，也可以用 `git config` 命令来添加别名。关于 git 别名的信息可以在[这里](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases)找到。
6. 在运行 `git config --global core.excludesfile ~/.gitignore_global` 之后，你可以在 `~/.gitignore_global` 中定义全局忽略模式。这条命令设置了 Git 将使用的全局忽略文件的位置，但你仍然需要在该路径下手动创建这个文件。设置好你的全局 gitignore 文件，忽略操作系统相关或编辑器相关的临时文件，比如 `.DS_Store`。
7. Fork [课程网站的仓库](https://github.com/missing-semester/missing-semester)，找一个错别字或其他你可以做的改进，并在 GitHub 上提交一个拉取请求（你可能需要看看[这个项目](https://github.com/firstcontributions/first-contributions)）。请只提交有用的 PR（请不要刷屏打扰我们！）。如果你找不到可做的改进，可以跳过这道练习。
8. 通过模拟一个协作场景来练习解决合并冲突：
   1. 用 `git init` 创建一个新仓库，并创建一个名为 `recipe.txt` 的文件，写上几行内容（例如一个简单的食谱）。
   2. 提交它，然后创建两个分支：`git branch salty` 和 `git branch sweet`。
   3. 在 `salty` 分支上，修改某一行（例如把 “1 cup sugar” 改成 “1 cup salt”）并提交。
   4. 在 `sweet` 分支上，以不同的方式修改同一行（例如把 “1 cup sugar” 改成 “2 cups sugar”）并提交。
   5. 现在切换到 `master`，先尝试 `git merge salty`，再 `git merge sweet`。发生了什么？查看 `recipe.txt` 的内容——`<<<<<<<`、`=======` 和 `>>>>>>>` 这些标记是什么意思？
   6. 通过编辑文件保留你想要的内容、删除冲突标记，并用 `git add` 和 `git commit`（或 `git merge --continue`）完成合并，来解决冲突。或者，试着用 `git mergetool` 配合图形化或终端的合并工具来解决冲突。
   7. 用 `git log --graph --oneline` 可视化你刚刚创建的合并历史。

---

原文：https://missing.csail.mit.edu/2026/version-control/ （The Missing Semester of Your CS Education, MIT）

翻译自原文。原文以 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 授权发布，本译文遵循相同许可。
