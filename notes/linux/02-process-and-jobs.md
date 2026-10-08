# Linux 进程、作业控制与信号

本篇梳理进程与线程的区别、进程状态的判定、ps/top/kill 等观察与控制工具、作业控制 (job control) 的机制，以及信号 (signal) 的语义与常见陷阱。

## 1. 进程与线程

先建立两个层次的概念：

| 概念 | 定义 | 拥有什么 |
| --- | --- | --- |
| 进程 (process) | 资源分配的基本单位 | 独立的虚拟地址空间、文件描述符表、信号处理表、PID |
| 线程 (thread) | CPU 调度的基本单位 | 独立的栈与寄存器上下文；**共享**地址空间、文件描述符、大部分信号设置 |

**Linux 的实现细节值得知道**：内核并不区分"进程"和"线程"，两者都由同一个结构 `task_struct` 表示，统称 **task**。区别来自创建时 `clone()` 系统调用传入的 flags——共享哪些资源：

| `clone()` flags | 效果 |
| --- | --- |
| 都不共享 | 相当于 `fork()`，得到独立进程 |
| `CLONE_VM \| CLONE_FS \| CLONE_FILES \| CLONE_SIGHAND \| CLONE_THREAD` | 相当于 `pthread_create()`，得到线程 |

这正是 02 篇在操作系统板块里提到的"线程是共享地址空间的 task"的具体含义。实践上的两个推论：

- **线程能互相破坏对方的内存**（共享地址空间），因此同一进程内的内存错误影响面更大；进程之间则被 MMU 隔离。
- **`ps` 默认只显示进程（进程组首领）**，要看线程需要 `-L` 或 `-T`：

```bash
ps -eLf                    # -L 显示线程 (LWP)，一个进程多行
ps -eLf | grep -c ''       # 数一下系统里总共有多少 task
top -H -p 1234             # top 里按线程显示指定进程
```

**PID 与 PPID**：每个进程有 PID，以及父进程 PID (PPID)。所有进程构成一棵树，根是 PID 1（现代发行版上是 systemd）。用 `pstree` 直观查看：

```bash
pstree -p                  # -p 显示 PID
pstree -p 1234             # 只看某个进程的子树
ps -ef --forest            # ps 也能画树（--forest 是 GNU 扩展）
```

如果父进程先退出，子进程会被 **re-parent** 给 PID 1（在引入 subreaper 之前），这也解释了为什么孤儿进程不会失控。

## 2. 进程状态

`ps` 的 `STAT` 列（`ps aux`）或 `S` 列（`ps -ef`）用单字母表示状态。这是必须能一眼读懂的。

| 状态码 | 名称 | 含义 | 可否被 kill |
| --- | --- | --- | --- |
| `R` | Running / Runnable | 正在 CPU 上执行，或在运行队列里等着被调度 | ✅ |
| `S` | Interruptible Sleep | 可中断睡眠，等待事件（网络、键盘、定时器）。**绝大多数空闲进程都在这个状态** | ✅ |
| `D` | Uninterruptible Sleep | 不可中断睡眠，通常在等磁盘 I/O。**不响应信号，连 `SIGKILL` 也不行** | ❌ |
| `T` | Stopped | 被作业控制停止（`Ctrl+Z` 或 `SIGSTOP`） | 只能 `SIGCONT` 唤醒 |
| `Z` | Zombie | 已终止但父进程还没 `wait()` 回收 | ❌（已死） |
| `I` | Idle (Linux 4.14+) | 内核线程的空闲状态 | — |
| `X` | Dead | 不应被看到（只在极短瞬间存在） | — |

附加修饰符（`ps aux` 的 STAT 列常带后缀）：

| 修饰 | 含义 |
| --- | --- |
| `s` | session leader（会话首领） |
| `l` | 多线程 (multi-threaded) |
| `+` | 位于前台进程组 |
| `<` | 高优先级 (high priority) |
| `N` | 低优先级 (nice) |
| `L` | 有页面被锁在内存中 (locked) |

### 2.1 `D` 状态为什么危险

`D` 状态的进程处于**不可中断睡眠**，典型原因是在等磁盘或网络文件系统的 I/O，而且这段等待在**内核态**且不检查待处理信号。后果：

- 它**不响应任何信号**，包括 `kill -9`；
- 如果底层 I/O 永远不返回（硬盘故障、NFS 服务器挂了、内核驱动 bug），这个进程会**永久卡住**；
- 它还可能导致系统负载 (load average) 持续升高，因为 load average 把 `D` 状态也计入（这是一个重要的、容易被误判的点：**load 高不一定是 CPU 忙**）。

**处理方法**：不能靠 `kill`。要么修复底层 I/O（恢复 NFS、检查磁盘），要么重启系统。如果大量进程堆积在 `D`，通常意味着存储层出了问题，应该先看 `dmesg` 和 `iostat`。

### 2.2 `Z` 状态（僵尸进程）与孤儿进程

**僵尸进程 (zombie)**：子进程已经退出，但父进程还没有调用 `wait()` 读取它的退出状态。此时子进程的**资源（内存、文件描述符）都已经释放**，只剩内核里的一个 `task_struct` 记着退出码和 PID。

- 僵尸**不占内存**（除了那个极小的结构体），所以少量僵尸无害。
- 僵尸**不能被 `kill`**——它已经死了。这一点必须记住。
- **清理僵尸的唯一方式是让它的父进程调用 `wait()`**。实践中的做法：
  - 修好父进程让它正确回收子进程（这是正解）；
  - 或者 `kill` 父进程，僵尸被 re-parent 给 PID 1，PID 1 会自动 `wait()` 回收它（这是权宜之计，因为父进程也被杀掉了）。

```bash
# 找僵尸进程，以及它们的父进程
ps aux | awk '$8 ~ /^Z/ {print $2, $11}'
ps -eo pid,ppid,stat,comm | awk '$3 ~ /^Z/'
```

**孤儿进程 (orphan)**：父进程先退出，子进程还活着。它被 re-parent 给 PID 1（或最近的 subreaper），继续正常运行。孤儿进程**不是问题**，与僵尸是两件不同的事——不要混为一谈。

## 3. 观察进程：ps / top / 相关工具

### 3.1 `ps` 的两种语法风格

`ps` 支持三种风格，混用会让人困惑：

| 风格 | 写法 | 特点 |
| --- | --- | --- |
| UNIX (POSIX) | `ps -ef` | 选项前有 `-` |
| BSD | `ps aux` | 选项前**没有** `-` |
| GNU 长选项 | `ps --sort=-%mem` | 长选项 |

```bash
ps aux                # 所有用户的全部进程，BSD 风格
ps -ef                # 同上，UNIX 风格
ps -ef --forest       # 加上进程树
ps -u alice           # 只看某个用户的进程
ps -p 1234            # 只看指定 PID
ps -eo pid,ppid,user,%cpu,%mem,stat,etime,cmd --sort=-%cpu | head -20
```

**`ps aux` 各列含义**（这是必须记住的）：

| 列 | 含义 |
| --- | --- |
| `USER` | 进程的有效用户 (euid) |
| `PID` | 进程 ID |
| `%CPU` | CPU 占用百分比（注意：是**进程整个生命周期的平均**，不是瞬时值，这是常见误读） |
| `%MEM` | 占物理内存百分比（基于 RSS） |
| `VSZ` | 虚拟内存大小 (Virtual Set Size)，包含已申请未使用的部分 |
| `RSS` | 常驻内存集 (Resident Set Size)，实际占用的物理内存 |
| `TTY` | 关联的终端；`?` 表示没有控制终端（守护进程常见） |
| `STAT` | 进程状态（见第 2 节） |
| `START` | 启动时间（超过 24 小时显示日期） |
| `TIME` | 累计消耗的 CPU 时间（不是墙钟时间！） |
| `COMMAND` | 命令行 |

**`%CPU` 与 `TIME` 是累计量**这一点极其重要：一个 `%CPU` 显示 200% 的进程（多核）可能是长期累积的结果，要判断"现在忙不忙"应该用 `top` 或看 `/proc/<pid>/stat` 的差值。

`ps -eo` 支持很多有用的格式符：

```bash
ps -eo pid,ppid,user,lstart,etime,cmd     # lstart: 启动时刻  etime: 已运行时长
ps -eo pid,stat,wchan:32,cmd              # wchan: 内核里正等待什么（排查 D 状态很有用）
```

**`etime` 与 `TIME` 的区别**：`etime` 是墙钟已运行时间（ELAPSED），`TIME` 是消耗的 CPU 时间。一个进程 `etime` 显示 `10-00:00:00`（10 天）但 `TIME` 只有 `00:00:03`，说明它几乎一直闲着。

### 3.2 `top` —— 实时的进程与系统概览

```bash
top                    # 默认每 3 秒刷新
top -d 1               # 每 1 秒刷新
top -u alice           # 只看某用户
top -p 1234,5678       # 只看指定 PID
top -b -n 1            # 批处理模式，输出一次就退出（用于脚本/日志）
```

`top` 的头部信息（这几行信息量很大）：

```text
top - 10:31:02 up 3 days,  4:12,  2 users,  load average: 0.52, 0.48, 0.45
Tasks: 287 total,   1 running, 286 sleeping,   0 stopped,   0 zombie
%Cpu(s):  3.2 us,  1.1 sy,  0.0 ni, 95.4 id,  0.2 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem :  15872.0 total,   9214.3 free,   3118.5 used,   3539.2 buff/cache
MiB Swap:   2048.0 total,   2048.0 free,      0.0 used.  11980.6 avail Mem
```

逐项解读：

| 项 | 含义 |
| --- | --- |
| `load average: 0.52, 0.48, 0.45` | 过去 1/5/15 分钟的**运行队列平均长度**（含 `R` 和 `D` 状态的任务）。**不是 CPU 百分比** |
| `us` | user 态 CPU 时间占比 |
| `sy` | 内核 (system) 态占比 |
| `ni` | nice 过的进程占比 |
| `id` | 空闲 |
| `wa` | 等 I/O (iowait) 占比 |
| `hi` / `si` | 硬中断 / 软中断占比 |
| `st` | 被虚拟化偷走的时间 (steal)，虚拟机里很关键 |
| `buff/cache` | 文件缓存占用的内存，**可回收**，不算真正的"已用" |
| `avail Mem` | 估算的可用内存（含可回收的缓存），**判断内存是否吃紧应该看这个而不是 free** |

**load average 怎么读**：要除以 CPU 核心数才有意义。8 核机器上 load 8.0 表示刚好跑满；load 0.5 表示很闲。**如果 load 高但 `id` 也高，说明压力来自 `D` 状态**（等 I/O），而不是 CPU 不够——这是最容易被误判的场景。

`top` 的交互键（在 top 运行时按）：

| 键 | 作用 |
| --- | --- |
| `P` | 按 CPU 排序（默认） |
| `M` | 按内存排序 |
| `T` | 按运行时间排序 |
| `k` | 输入 PID 并发送信号（最常用的"在 top 里杀进程"） |
| `r` | renice，调整优先级 |
| `1` | 展开/合并多核 CPU 的显示 |
| `c` | 切换显示完整命令行还是仅程序名 |
| `H` | 切换显示线程 |
| `u` | 按用户过滤 |
| `q` | 退出（`Ctrl+C` 也可以） |

`htop` 是更友好的替代品（彩色、可用鼠标、可横向滚动），但需要单独安装（`sudo apt install htop`），且**默认没有**——不要假设任何机器上都有它。

### 3.3 其他有用的观察工具

```bash
pgrep -a nginx              # 按名字找 PID，-a 显示完整命令行
pgrep -u alice -l bash      # 限定用户
pstree -p 1                 # 从 init 开始的进程树
free -h                     # 内存概览（看 available 列）
uptime                      # 只看负载与在线时间
vmstat 1 5                  # 每秒采样 5 次：进程、内存、swap、I/O、CPU
iostat -xz 1                # 磁盘 I/O（可能需要 sysstat 包）
ss -lntp                    # 监听中的 TCP 端口及对应进程（-p 需要权限）
lsof -p 1234                # 某进程打开的所有文件
```

### 3.4 `/proc` —— 一切事实的来源

前面那些工具**本质上都在读 `/proc`**。知道这一点，就不会被工具的显示差异迷惑。

```bash
cat /proc/1234/status       # Name, State, Pid, PPid, Uid, VmRSS, Threads...
cat /proc/1234/cmdline | tr '\0' ' '; echo    # 命令行，参数之间是 NUL
cat /proc/1234/environ | tr '\0' '\n'         # 环境变量（需要权限）
ls -l /proc/1234/fd/        # 打开的文件描述符，链接指向实际文件
cat /proc/1234/maps         # 内存映射（哪些库映射到哪些地址）
cat /proc/1234/limits       # 资源限制（文件描述符上限等）
cat /proc/1234/cwd          # 当前工作目录（符号链接）
readlink /proc/1234/exe     # 可执行文件的真实路径（可判断是否已升级）
cat /proc/loadavg           # load average 的原始值
```

其中两个特别有用：

- **`/proc/<pid>/fd/`** 配合 `lsof` 可以找出"已删除但仍被打开"的文件（磁盘空间不释放的问题）。
- **`readlink /proc/<pid>/exe`** 会显示 `(deleted)` 后缀，说明二进制被替换过（升级了软件但没重启服务）——这是"重启后就好了"这类问题的根因。

## 4. 信号 (signal)

信号是**异步通知机制**：内核或其他进程向某个进程发送一个编号，进程有三种处理方式——**默认动作**、**忽略**、**自定义处理函数 (handler)**。

### 4.1 常用信号

```bash
kill -l                    # 列出所有信号名与编号
```

| 编号 | 名称 | 默认动作 | 说明 |
| --- | --- | --- | --- |
| 1 | `SIGHUP` | 终止 | 终端挂断。**daemon 常约定用它表示"重读配置"** |
| 2 | `SIGINT` | 终止 | `Ctrl+C` 产生的中断 |
| 3 | `SIGQUIT` | 终止 + core dump | `Ctrl+\` |
| 9 | `SIGKILL` | 终止 | **不可捕获、不可忽略、不可阻塞** |
| 11 | `SIGSEGV` | 终止 + core dump | 段错误，访问非法内存 |
| 13 | `SIGPIPE` | 终止 | 向已关闭的管道写数据。脚本里 `head` 之后报的错常是它 |
| 15 | `SIGTERM` | 终止 | **默认的 `kill` 信号**，可被捕获，用于优雅退出 |
| 17 | `SIGCHLD` | 忽略 | 子进程状态变化，父进程靠它触发回收 |
| 18 | `SIGCONT` | 继续 | 让停止的进程继续运行 |
| 19 | `SIGSTOP` | 停止 | 不可捕获、不可忽略（与 `SIGTSTP` 的区别在此） |
| 20 | `SIGTSTP` | 停止 | `Ctrl+Z` 产生的，**可被捕获** |
| 23 | `SIGURG` | 忽略 | 带外数据到达（很少用） |
| 28 | `SIGWINCH` | 忽略 | 终端窗口大小变化 |

编号在不同架构上可能不同（例如 `SIGCHLD` 在 x86 上是 17，某些架构上是 20），所以**脚本里应该写名字而不是数字**（`kill -TERM` 而不是 `kill -15`），可移植性更好。

### 4.2 `SIGTERM` 与 `SIGKILL` 的本质区别

这是最需要理解清楚的一对：

| | `SIGTERM` (15) | `SIGKILL` (9) |
| --- | --- | --- |
| 可捕获 | ✅ 进程可以自定义 handler | ❌ 由内核直接终止 |
| 可忽略 | ✅ | ❌ |
| 可阻塞 | ✅ | ❌ |
| 使用场景 | **应当作为首选**：给进程机会保存数据、关闭连接、清理临时文件 | 仅在 `SIGTERM` 无效时作为最后手段 |
| 风险 | 进程可能"卡住"不退出 | 进程无机会清理：**临时文件残留、数据库事务未提交、日志缓冲丢失、锁未释放** |

**正确做法**：先 `kill PID`（发 SIGTERM），等几秒；如果还在，再 `kill -9 PID`。

```bash
kill 1234                  # 等价于 kill -TERM 1234
kill -TERM 1234
kill -9 1234               # 强制
kill -HUP 1234             # 让服务重读配置（约定俗成，不是所有服务都支持）
kill -0 1234               # 不发信号，只检查进程是否存在（脚本里常用！）
kill -l 1234               # 注意这里是列出信号，不是对 1234 操作
kill -- -1234              # 给整个进程组发信号（负号表示进程组 ID）
```

一个容易被忽略的点：**`kill` 是系统调用，能中断系统调用的信号才叫"可中断"**。处于 `D` 状态的进程不检查信号，所以即使 `SIGKILL` 也已经在内核里排队，它也要等 I/O 返回才会被处理——这是"`kill -9` 杀不掉"的第一种正确解释（第二种是僵尸进程）。

### 4.3 `killall`、`pkill`

```bash
killall nginx              # 按进程名发 SIGTERM（默认）
killall -9 nginx           # 强制
killall -u alice firefox   # 限定用户
killall -i nginx           # 交互式确认
pkill -f "python3 app.py"  # 按完整命令行匹配（-f 匹配整个命令行）
pkill -u alice -f pytest   # 按用户 + 模式
pkill -9 -f myscript       # 强制
```

⚠️ **`pkill -f` 的陷阱**：`-f` 匹配完整命令行，因此**它可能匹配到你自己**（例如你执行的 `pkill -f myscript` 这行命令的进程名里就含 `myscript`）。结果是把执行 pkill 的 shell 也一起杀掉。安全做法是用 `pkill -f -- "pattern"` 或者干脆用 `pgrep -f` 先看清会匹配谁：

```bash
pgrep -af "myscript"       # 先看会匹配到什么
pkill -f "myscript"        # 确认后再杀
```

### 4.4 发送信号给进程组

负号表示"进程组"：

```bash
kill -- -1234              # 向进程组 1234 的所有成员发 SIGTERM
```

这一点与作业控制直接相关：当你在终端里按 `Ctrl+C`，**内核把 `SIGINT` 发给整个前台进程组**，而不只是那一个进程。这解释了为什么"一条管道里的所有命令都停了"。

## 5. 作业控制 (job control)

作业控制是 **shell** 提供的功能，管理的是"这个 shell 启动的进程组"，与内核的进程/进程组概念配合。

### 5.1 前台与后台

| 概念 | 含义 | 能否读终端输入 |
| --- | --- | --- |
| 前台作业 (foreground) | 独占终端，可以读键盘输入 | ✅ |
| 后台作业 (background) | 在后台运行，**读终端输入会收到 `SIGTTIN` 而被暂停** | ❌ |

**这个"后台读终端会被暂停"的行为是理解 `fg`/`bg` 的关键**：一个后台进程一旦试图读终端，就被挂起（状态变成 `T`），而不是报错退出或静默卡死。

```bash
long_task &                # 后台启动
# [1] 4321                 # 输出作业号与 PID
```

### 5.2 三个关键按键 —— 必须区分清楚

| 按键 | 实际发生了什么 | 是不是信号 |
| --- | --- | --- |
| `Ctrl+C` | 终端驱动向前台进程组发送 `SIGINT` | ✅ 是信号 |
| `Ctrl+Z` | 终端驱动向前台进程组发送 `SIGTSTP`（默认动作是**停止**，不是终止） | ✅ 是信号 |
| `Ctrl+D` | 不是一个信号。它让终端驱动把"输入结束"通知给读到 EOF 的程序，通常表现为 `read()` 返回 0 | ❌ **不是信号** |
| `Ctrl+\` | 发送 `SIGQUIT`（终止 + core dump） | ✅ 是信号 |

`Ctrl+D` 不是信号这一点是高频误解。它的机制是：终端处于规范模式时，`Ctrl+D` 让当前输入缓冲区的内容立即交付（如果没有缓冲区内容，则读到 EOF）。所以：
- 在 `cat` 面前按 `Ctrl+D`，`cat` 读到 EOF 后正常退出；
- 在 `bash` 提示符下按 `Ctrl+D`，bash 读到 EOF 后退出登录 shell；
- 一个不读 stdin 的程序对 `Ctrl+D` 完全没有反应。

### 5.3 `jobs` / `fg` / `bg` / `disown`

```bash
jobs                       # 列出当前 shell 的作业
jobs -l                    # 加上 PID
jobs -r                    # 只看运行中的
jobs -s                    # 只看已停止的

fg                         # 把最近的作业调到前台
fg %1                      # 指定作业号
fg %nginx                  # 按命令名开头匹配（%?nginx 是包含匹配）

bg                         # 让最近的停止作业在后台继续
bg %2

disown %1                  # 从 shell 的作业表里移除（进程继续跑，但 shell 不再管它）
disown -h %1               # 只移除 HUP 处理，进程仍留在作业表里
disown -a                  # 移除所有作业
```

**作业号 (job number) 与 PID 的区别**：作业号是 shell 内部的编号（`%1`），只对当前 shell 有效；PID 是系统的。在一个 shell 里 `fg %1`，到了另一个 shell 里就没有 `%1` 了。

作业号前的符号（`jobs` 输出里）：

| 符号 | 含义 |
| --- | --- |
| `+` | 当前作业 (current)，`fg`/`bg` 不带参数时的默认目标 |
| `-` | 前一个作业 (previous) |
| 空格 | 其他作业 |

### 5.4 一个完整的交互场景

```bash
$ vim bigfile.txt          # 打开编辑器
# 按 Ctrl+Z
[1]+  Stopped                 vim bigfile.txt
$ jobs
[1]+  Stopped                 vim bigfile.txt
$ make                     # 前台跑构建
# 按 Ctrl+Z 暂停构建
[2]+  Stopped                 make
$ bg %2                    # 让构建在后台继续
[2]+ make &
$ fg %1                    # 回到 vim
```

这是作业控制最实用的场景：**在编辑器与构建/测试命令之间来回切换，不用开多个终端**。

## 6. 让进程在终端关闭后继续运行

这是最常被问的问题之一："我 ssh 上去跑了个任务，断开连接后它就死了，怎么办？"

**根因**：终端关闭时，内核向该终端的**前台进程组**发送 `SIGHUP`，默认动作是终止进程。

四种应对方式，能力递增：

| 方式 | 命令 | 能否抵抗 SIGHUP | 能否抵抗标准输入关闭 | 能否事后重新接管 |
| --- | --- | --- | --- | --- |
| 仅后台 | `./run.sh &` | ❌ 仍然会收到 SIGHUP | ❌ | 否 |
| `disown` | `./run.sh &` 然后 `disown` | ✅（shell 不再向它转发 HUP） | 视情况 | 否 |
| `nohup` | `nohup ./run.sh &` | ✅（进程忽略 SIGHUP） | ✅（stdin 重定向为不可读） | 否 |
| `setsid` | `setsid ./run.sh` | ✅（脱离控制终端、新会话） | ✅ | 否 |
| `tmux` / `screen` | 在会话里跑 | ✅ | ✅ | ✅ **可以重新 attach** |
| systemd 服务 | 写成 unit | ✅ | ✅ | ✅ 有状态与日志 |

**`nohup` 的机制**：把进程对 `SIGHUP` 的处理设为**忽略 (SIG_IGN)**。注意这是一个"遗传"设置：如果程序自己重新注册了 SIGHUP handler，忽略设置会被覆盖，那时候 `nohup` 就失效了。另外 `nohup` 会在 stdin 是终端时把它重定向到不可读（避免后台读终端被暂停），并把 stdout 重定向到 `nohup.out`（如果 stdout 还是终端的话）。

**最常见的写法**：

```bash
nohup ./long_task.sh > /tmp/long_task.log 2>&1 &
```

逐段解释：
- `nohup` —— 忽略 SIGHUP；
- `./long_task.sh` —— 要跑的命令；
- `> /tmp/long_task.log` —— 明确指定 stdout 到文件（否则会写到 `nohup.out`，位置不好找）；
- `2>&1` —— **把 stderr 也指向同一个文件**。注意顺序：`2>&1` 必须写在 `> file` **之后**，否则 stderr 会指向"重定向前"的 stdout（还是终端）；
- `&` —— 放后台。

**判断命令是否还活着**：

```bash
jobs -l                    # 如果在同一个 shell 里
ps -p <PID>                # 直接查
pgrep -af long_task        # 按名字查
tail -f /tmp/long_task.log # 看输出有没有在增长（最直接的证据）
```

**更推荐的做法**：对任何需要长期运行的**服务**，直接写成 systemd unit（见 [05-package-and-service.md](05-package-and-service.md)）。它会处理重启、日志（journalctl）、开机自启、依赖顺序——这些用 `nohup` 都要自己实现。`nohup` 适合"一次性的长任务"，`tmux` 适合"需要交互的长任务"，systemd 适合"需要长期存在的服务"。

## 7. 优先级：nice 与 renice

Linux 用 **nice 值**表示优先级，范围 **-20（最高优先级）到 19（最低优先级）**，默认 0。注意方向：**数值越大优先级越低**（越"nice"，越谦让）。

```bash
nice -n 10 ./heavy.sh           # 以 nice 10 启动（降低优先级）
sudo nice -n -5 ./important.sh  # 提高优先级需要 root（负值）
renice 15 -p 1234               # 调整运行中进程的 nice
renice -5 -p 1234               # 提高（需要权限）
renice 10 -u alice              # 调整某用户所有进程
```

**`nice` 不是万能的速度调节器**：它只影响 CPU 时间片的分配倾向，对 I/O 等待、内存压力毫无作用。一个 nice 值很高的进程如果卡在 `D` 状态，照样拖慢系统负载。相反，如果系统里只有一个可运行任务，它的 nice 值再低也照样独占 CPU——因为**没有别人跟它抢**。这是一个重要的、常被误解的点。

## 8. 排查实践

### 8.1 "CPU 跑满了，怎么找出是谁"

```bash
# 第 1 步：看整体
top -b -n 1 | head -20
uptime                     # 看 load 与核心数对比
nproc                      # 有几个核心

# 第 2 步：按 CPU 排序取前几名
ps -eo pid,ppid,user,%cpu,%mem,etime,cmd --sort=-%cpu | head -10

# 第 3 步：确认是哪个线程（多线程程序）
top -H -p <PID>
ps -L -p <PID> -o pid,tid,pcpu,comm

# 第 4 步：看它在干什么
cat /proc/<PID>/status | head -5
cat /proc/<PID>/wchan; echo         # 内核里等待什么
strace -p <PID> -c                  # 统计系统调用（需要权限，会有性能影响）
```

**注意 `%CPU` 是累计平均**：一个刚启动就吃满 CPU 的进程与一个跑了 3 天偶尔吃 CPU 的进程，`ps` 的 `%CPU` 显示可能一样。要看瞬时值，用 `top` 的两帧差值，或读 `/proc/<PID>/stat` 的 `utime`/`stime` 两次算差。

### 8.2 "进程杀不掉，怎么办"

按这个顺序判断：

```text
1. 它是僵尸吗？   ps -o stat= -p PID  → 是 Z
   → 杀不掉，因为它已经死了。去处理它的父进程（或修好父进程的 wait）。
      找父进程：ps -o ppid= -p PID

2. 它在 D 状态吗？  ps -o stat= -p PID  → 含 D
   → 不响应信号，因为不可中断睡眠不检查信号。
      看它在等什么：cat /proc/PID/wchan; echo
      检查存储/网络文件系统：dmesg | tail, iostat -xz 1
      → 修底层 I/O，或重启。

3. 我发的信号对吗？   kill -9 试过了吗？
   → 有些进程会捕获 SIGTERM 并忽略；确认用 kill -9。

4. 我有权限吗？   ps -o user= -p PID
   → 非 root 只能杀自己的进程（同一个 UID 即可）。

5. PID 复用了吗？   ps -p PID -o lstart,cmd
   → 如果 PID 已被复用，你杀的是另一个进程。用启动时间确认。
```

第 5 点值得一提：PID 会被回收复用。脚本里"先 `pgrep` 拿 PID，过一会儿再 `kill` 那个 PID"存在杀错对象的风险。更稳妥的做法是用 `pgrep`/`pkill` 在**同一个瞬间**完成匹配与发送，或者用 PID 文件 + 启动时间去校验。

### 8.3 "服务/进程莫名消失了"

```bash
dmesg -T | tail -50            # 看内核有没有 OOM killer 的记录
journalctl -k --since "10 min ago" | grep -i -E 'oom|killed process'
journalctl --since "10 min ago" -p err       # 系统级错误
grep -i oom /var/log/syslog    # Debian/Ubuntu 上也可以直接翻 syslog
```

在容器或 `systemd` 的 `MemoryMax=` 限制下，进程被杀同样会留下 OOM 相关记录。**"进程无故消失"最常见的原因就是被 OOM killer 杀了**，而这一点从应用日志里是看不到的，必须查内核日志。

## 小结

- Linux 里进程与线程都是 `task_struct`，区别只在 `clone()` 共享了哪些资源；线程共享地址空间，所以线程间的内存错误影响面更大。
- 进程状态必须一眼看懂：`R` 运行、`S` 可中断睡眠、**`D` 不可中断睡眠（不响应任何信号，包括 SIGKILL）**、`T` 停止、`Z` 僵尸。
- 僵尸进程**不能被 kill**（它已经死了），只能靠父进程 `wait()` 回收；孤儿进程被 re-parent 给 PID 1，不是问题。
- 读 load average 要除以核心数；**load 高但 CPU 空闲**通常意味着有大量 `D` 状态任务在等 I/O。
- `ps aux` 的 `%CPU` 与 `TIME` 都是**累计量**，不是瞬时值；判断当前负载要用 `top` 或读两次 `/proc/PID/stat` 算差。
- `SIGTERM`(15) 可捕获、用于优雅退出，是首选；`SIGKILL`(9) 不可捕获，只在必要时使用，因为它会让进程失去清理机会。
- `Ctrl+C` 是 `SIGINT`、`Ctrl+Z` 是 `SIGTSTP`，但 **`Ctrl+D` 不是信号**，它是 EOF。
- 作业控制里"后台进程一读终端就被暂停"是设计行为（`SIGTTIN`），这是 `fg`/`bg` 存在的意义。
- 让进程活过终端关闭：`nohup`（忽略 SIGHUP）< `setsid`（脱离终端）< `tmux`/`screen`（还能重新接管）< systemd（有重启与日志管理）。
- 重定向写 `> file 2>&1` 时顺序不能反；`2>&1` 必须在 `> file` 之后。
- "杀不掉进程"的排查顺序：是不是僵尸 → 是不是 `D` 状态 → 信号对不对 → 有没有权限 → PID 是否已被复用。
- 进程无故消失先查内核日志里的 OOM killer 记录，应用日志里不会有。
