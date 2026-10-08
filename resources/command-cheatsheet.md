# 命令速查表：Linux + SQL 高频命令

一份可以当工具书用的密集速查表：左侧命令、右侧一句话说明。按用途分组，遇到具体问题按分组查。

## 使用说明

- 命令与代码用**等英文原文**，说明用中文。
- 标注 **[GNU]** 的是 GNU 扩展，BSD/macOS 可能不支持；标注 **[Debian/Ubuntu]** 或 **[RHEL 系]** 的是发行版特有。
- 方括号 `[...]` 表示可选参数，`<...>` 表示必填占位符。
- 危险命令用 ⚠️ 标出。

---

## 一、Linux —— 文件与目录

| 命令 | 说明 |
| --- | --- |
| `pwd` | 显示当前工作目录的绝对路径 |
| `cd <dir>` | 切换目录；`cd -` 回到上一个目录；`cd` 回家目录 |
| `ls -l` | 长格式列表：权限、链接数、属主、属组、大小、时间 |
| `ls -la` | 含隐藏文件（以 `.` 开头） |
| `ls -lh` | 大小用人类可读单位（K/M/G） |
| `ls -lt` | 按修改时间排序（最新在前）；`-ltr` 反转 |
| `ls -lS` | 按大小排序（最大在前） |
| `ls -li` | 显示 inode 号（排查硬链接） |
| `ls -ld <dir>` | 看目录本身的信息，而不是目录内容 |
| `tree -L 2` | 以树形显示，深度 2 层（需安装 `tree`） |
| `cp <src> <dst>` | 复制文件 |
| `cp -r <dir1> <dir2>` | 递归复制目录 |
| `cp -a <src> <dst>` | 归档式复制（保留权限、属主、时间戳、软链接） |
| `cp -i` / `mv -i` | 覆盖前询问（建议设成默认别名） |
| `cp -u` | 仅在源更新时复制 |
| `mv <src> <dst>` | 移动或重命名；同一文件系统内是瞬时的 |
| `rm <file>` | 删除文件；⚠️ **没有回收站** |
| `rm -r <dir>` | 递归删除目录 |
| `rm -rf <dir>` | 强制递归删除，不询问 ⚠️ |
| `rm -i` | 删除前逐个询问 |
| `mkdir -p a/b/c` | 递归创建多级目录 |
| `rmdir <dir>` | 只能删空目录（比 `rm -r` 安全） |
| `touch <file>` | 创建空文件，或更新已有文件的时间戳 |
| `file <file>` | 猜文件类型（看 magic number，不看扩展名） |
| `stat <file>` | 详细元数据：inode、权限、atime/mtime/ctime |
| `ln <target> <link>` | 创建硬链接（共享 inode，不能跨文件系统，不能指向目录） |
| `ln -s <target> <link>` | 创建符号链接 |
| `readlink -f <path>` | 解析成规范化的绝对路径 |
| `basename <path>` / `dirname <path>` | 取文件名 / 取目录部分 |
| `realpath <path>` | 同 `readlink -f` |
| `du -sh <dir>` | 目录总大小（summary + human） |
| `du -h --max-depth=1 <dir>` | 只看第一层各子目录大小 |
| `df -h` | 各文件系统使用情况 |
| `df -h .` | 只看当前目录所在分区 |
| `lsof +L1` | 已删除但仍被进程打开的文件（磁盘空间不释放的排查）㊙ |
| `sync` | 把内存中的脏页刷到磁盘 |

### 查找文件

| 命令 | 说明 |
| --- | --- |
| `find <path> -name "*.log"` | 按名字查找（`-iname` 忽略大小写） |
| `find . -type f` | 只要普通文件（`d` 目录、`l` 符号链接） |
| `find . -type f -size +100M` | 大于 100M（`-1k` 小于 1k） |
| `find . -mtime -7` | 7 天内修改过（`+30` 是 30 天前） |
| `find . -mmin -10` | 10 分钟内修改过 |
| `find . -maxdepth 1` | 限制搜索深度 |
| `find / -perm -4000 -type f 2>/dev/null` | 找 SUID 文件（安全审计） |
| `find . -name "*.tmp" -exec rm -v {} \;` | 找到后执行命令 |
| `find . -name "*.log" -exec gzip {} +` | 用 `+` 让多个文件一次传参，更高效 |
| `find . -print0 \| xargs -0 rm` | 安全处理含空格的文件名 ⭐ |
| `locate <name>` | 从数据库查（极快但可能过期，需 `updatedb` 更新） |

### 归档与压缩

| 命令 | 说明 |
| --- | --- |
| `tar -czvf a.tar.gz dir/` | 创建 gzip 归档（c=创建 z=gzip v=详细 f=文件） |
| `tar -xzvf a.tar.gz` | 解压（x=解压） |
| `tar -xzvf a.tar.gz -C /tmp/` | 解压到指定目录 |
| `tar -tzvf a.tar.gz` | 只列出内容不解压（t=列表） |
| `tar -cjvf a.tar.bz2 dir/` | 用 bzip2（更小更慢） |
| `tar -cJvf a.tar.xz dir/` | 用 xz（最小最慢） |
| `tar -czf a.tar.gz --files-from list.txt` | 从文件列表打包 |
| `gzip -k <file>` | 压缩并保留原文件 |
| `unzip <file.zip> -d <dir>` | 解压 zip 到目录 |
| `zstd <file>` | 现代压缩工具（也可 `tar --zstd`） |

### 查看文件

| 命令 | 说明 |
| --- | --- |
| `cat <file>` | 输出整个文件 |
| `less <file>` | 分页查看（`/` 搜索、`q` 退出）⭐ |
| `head -n 20 <file>` | 前 20 行 |
| `tail -n 20 <file>` | 后 20 行 |
| `tail -f <file>` | 持续跟踪新写入的行 |
| `tail -F <file>` | 同上，日志轮转后仍能跟踪 ⭐ |
| `diff <a> <b>` | 逐行比较；`-u` 输出统一格式（同 `patch` 格式） |
| `vimdiff <a> <b>` | 并排比较 |

---

## 二、Linux —— 重定向与管道

| 语法 | 说明 |
| --- | --- |
| `cmd > file` | stdout 覆盖写入（**会先清空文件**） |
| `cmd >> file` | stdout 追加 |
| `cmd 2> err.log` | stderr 写入 |
| `cmd > out.log 2>&1` | stdout 和 stderr 都进文件 ⭐ **正确顺序** |
| `cmd 2>&1 > out.log` | ⚠️ **错误**：只有 stdout 进文件，stderr 仍到终端 |
| `cmd &> file` | 上面第一种的 bash 简写 |
| `cmd < input.txt` | 从文件读 stdin |
| `cmd <<< "string"` | here-string，字符串作为 stdin |
| `cmd1 \| cmd2` | 管道：cmd1 的 stdout 接到 cmd2 |
| `cmd \| tee file` | 同时输出到屏幕和文件（`-a` 追加） |
| `cmd > /dev/null 2>&1` | 彻底静默 |
| `cmd > >(tee log) 2>&1` | 持续同时输出到屏幕与日志（bash 进程替换） |
| `{ cmd1; cmd2; } > log 2>&1` | 一组命令的输出统一重定向 |

---

## 三、Linux —— 文本处理

### 筛选与查看

| 命令 | 说明 |
| --- | --- |
| `grep "pat" <file>` | 找出匹配的行 |
| `grep -i` | 忽略大小写 |
| `grep -v` | 反向匹配（输出**不**匹配的行） |
| `grep -n` | 显示行号 |
| `grep -r "pat" <dir>` | 递归搜索目录 |
| `grep -l` | 只输出含匹配的**文件名** |
| `grep -c` | 只输出匹配的**行数**（不是出现次数） |
| `grep -w` | 全词匹配（`-w cat` 不匹配 `category`） |
| `grep -E` | 扩展正则（ERE），等于 `egrep` |
| `grep -F` | 固定字符串，不做正则解释（最快，等于 `fgrep`） |
| `grep -o` | 只输出匹配到的部分（不是整行） |
| `grep -q` | 静默，只用退出码表示是否匹配（脚本里用） |
| `grep -A3 -B2 -C1` | 显示匹配行的后 3 行 / 前 2 行 / 前后各 1 行 |
| `grep -rn --include="*.conf" "listen" /etc/` | 限定文件模式递归搜索 |
| `grep -v -E '^\s*(#\|$)' file` | 排除空行与注释行 ⭐ |

**`grep` 退出码**：`0` = 有匹配，`1` = 无匹配，`2` = 出错。脚本里用 `grep -q`。

### 替换与编辑

| 命令 | 说明 |
| --- | --- |
| `sed 's/old/new/' file` | 每行替换第一次出现 |
| `sed 's/old/new/g' file` | 每行替换所有出现 |
| `sed -i 's/old/new/g' file` | 直接改文件 **[GNU]** |
| `sed -i.bak 's/a/b/g' file` | 改文件并留备份 **[GNU]** |
| `sed -i '' 's/a/b/g' file` | **macOS/BSD** 的写法（`-i` 后必须有参数）⚠️ |
| `sed -n '10,20p' file` | 打印第 10-20 行（`-n` 抑制默认输出） |
| `sed -n '/BEGIN/,/END/p' file` | 打印两个标记之间的内容 |
| `sed '1d' file` | 删除第 1 行（跳过表头） |
| `sed '/^$/d' file` | 删除空行 |
| `sed '/^#/d' file` | 删除注释行 |
| `sed '3i\text' file` | 在第 3 行前插入 |
| `sed '3a\text' file` | 在第 3 行后追加 |
| `sed 's/$/ SUFFIX/' file` | 每行行尾加内容 |
| `sed 's/^/PREFIX: /' file` | 每行行首加内容 |
| `sed -i 's/\r$//' file` | 把 CRLF 转成 LF ⭐ |
| `sed -E 's/([a-z]+)@([a-z.]+)/\2:\1/'` | 扩展正则 + 反向引用 |

### 按字段处理

| 命令 | 说明 |
| --- | --- |
| `awk '{print $1, $3}' file` | 打印第 1、3 个字段 |
| `awk '{print $NF}' file` | 打印最后一个字段 |
| `awk -F: '{print $1}' /etc/passwd` | 指定分隔符 |
| `awk '{print NR": "$0}' file` | 加行号 |
| `awk '$3 > 100' file` | 按条件筛选 |
| `awk 'NR>=10 && NR<=20' file` | 按行号范围 |
| `awk '{sum += $3} END {print sum}' file` | 求和 |
| `awk '{sum+=$3; n++} END {print sum/n}' file` | 求平均 |
| `awk '{c[$1]++} END {for (k in c) print k, c[k]}'` | 按第一列分组计数 |
| `awk '!seen[$0]++' file` | **保持顺序去重** ⭐ |
| `awk -v t="$var" '$3 > t' file` | 把 shell 变量传进 awk（必须用 `-v`） |
| `awk 'BEGIN{OFS=","} {print $1,$2}' file` | 输出改成 CSV |
| `awk '{printf "%-10s %6.2f\n", $1, $2}' file` | 格式化输出 |
| `awk '$9 ~ /^[45]/' file` | 正则匹配字段 |
| `cut -d: -f1 /etc/passwd` | 按单字符分隔取列 |
| `cut -d: -f1,7` | 取第 1 和第 7 列 |
| `cut -d: -f2-5` | 取第 2 到第 5 列 |
| `cut -d: -f1 --complement` | 取除第 1 列外的所有列 |
| `cut -c1-10` | 按字符位置取 |
| `awk '{print $2}'` | **多空格分隔时必须用 awk，不能用 `cut -d' '`** ⚠️ |

### 排序、去重、计数、字符处理

| 命令 | 说明 |
| --- | --- |
| `sort file` | 字典序排序 |
| `sort -n` | 按数值排序（否则 10 会排在 9 前面）⚠️ |
| `sort -r` | 逆序 |
| `sort -u` | 排序并去重 |
| `sort -k2` | 按第 2 列（**到行尾**）排序 |
| `sort -k2,2n` | 只按第 2 列、数值排序 ⭐ |
| `sort -t: -k3,3n /etc/passwd` | 指定分隔符与列 |
| `sort -h` | 人类可读数值（1K < 1M < 1G） |
| `sort -V` | 版本号排序（v1.10 在 v1.9 之后） |
| `LC_ALL=C sort file` | 按字节序排序（确定性 + 更快）⭐ |
| `uniq` | 去掉**相邻**重复（必须先 sort）⚠️ |
| `uniq -c` | 加上出现次数 ⭐ |
| `uniq -d` | 只输出重复出现的行 |
| `uniq -u` | 只输出只出现一次的行 |
| `sort \| uniq -c \| sort -rn \| head` | **Top N 万能模板** ⭐⭐ |
| `wc -l file` | 行数（实际数的是换行符个数） |
| `wc -l < file` | 只输出数字（脚本里用）⭐ |
| `wc -w` / `wc -c` / `wc -m` | 词数 / 字节数 / 字符数（UTF-8 下 `-c` 与 `-m` 不同） |
| `tr 'a-z' 'A-Z' < file` | 小写转大写（`tr` 必须用 `<` 或管道） |
| `tr -d '\r' < file` | 删除回车 |
| `tr -s ' '` | 把连续多个空格压成一个 |
| `tr -s '\n'` | 把连续多个换行压成一个（去空行） |
| `tr -cs '[:alpha:]' '\n' < f` | 把非字母字符都换成换行（拆词） |
| `comm -12 a b` | 交集（要求两边都已排序） |
| `comm -23 a b` | 只在 a 中的行 |
| `paste -sd+ file \| bc` | 把一列数字用 `+` 连接后求和 |

---

## 四、Linux —— 进程与作业

| 命令 | 说明 |
| --- | --- |
| `ps aux` | 所有进程（BSD 风格）；`%CPU` 与 `TIME` 是**累计值** ⚠️ |
| `ps -ef` | 同上（UNIX 风格） |
| `ps -ef --forest` | 显示进程树 |
| `ps -eo pid,ppid,user,%cpu,%mem,etime,cmd` | 自定义输出列 |
| `ps -eo ... --sort=-%cpu \| head` | 按 CPU 排序取前几 ⭐ |
| `ps -L -p <PID>` | 显示某进程的线程 |
| `ps -p <PID> -o lstart,cmd` | 确认 PID 是否被复用 |
| `pstree -p` | 进程树（带 PID） |
| `pgrep -a <name>` | 按名字找 PID（`-a` 显示完整命令行） |
| `pgrep -u <user> -f <pat>` | 按用户 + 完整命令行匹配 |
| `top` | 实时进程/系统概览 |
| `top -b -n 1` | 批处理输出一次（脚本用） |
| `top -H -p <PID>` | 显示线程 |
| `htop` | 更友好的交互式查看器（需安装） |
| `uptime` | 在线时间与 load average |
| `free -h` | 内存概览（**看 `available` 列判断是否吃紧**）⭐ |
| `vmstat 1 5` | 每秒采样 5 次：进程/内存/swap/IO/CPU |
| `iostat -xz 1` | 磁盘 I/O（需 sysstat） |
| `ss -lntp` | 监听中的 TCP 端口及进程 ⭐ |
| `ss -tnp` | 已建立的 TCP 连接 |
| `netstat -lntp` | 旧的等价命令（需 net-tools） |
| `lsof -p <PID>` | 某进程打开的文件 |
| `kill <PID>` | 发 SIGTERM（**首选**，进程可清理） |
| `kill -9 <PID>` | 发 SIGKILL（不可捕获，最后手段） |
| `kill -HUP <PID>` | 常用于让服务重读配置 |
| `kill -0 <PID>` | 不发信号，只检查进程是否存在 |
| `kill -- -<PGID>` | 给整个进程组发信号 |
| `killall <name>` | 按进程名发信号 |
| `pkill -f "<pattern>"` | 按完整命令行匹配后发信号 ⚠️ 可能匹配到自己 |
| `nohup cmd > log 2>&1 &` | 后台运行且忽略 SIGHUP ⭐ |
| `setsid cmd` | 脱离控制终端运行 |
| `disown %1` | 从 shell 作业表移除 |
| `nice -n 10 cmd` | 以低优先级启动 |
| `renice 15 -p <PID>` | 调整运行中进程的优先级 |
| `jobs -l` | 列出作业（含 PID） |
| `fg %1` | 把作业调到前台 |
| `bg %1` | 让停止的作业在后台继续 |
| `Ctrl+C` / `Ctrl+Z` / `Ctrl+D` | SIGINT / SIGTSTP / **EOF（不是信号！）** ⚠️ |

### 进程状态码（`ps` 的 STAT 列）

| 码 | 含义 | 能否 kill |
| --- | --- | --- |
| `R` | 运行/可运行 | ✅ |
| `S` | 可中断睡眠（等待事件） | ✅ |
| `D` | **不可中断睡眠（等 I/O）** | ❌ 连 SIGKILL 也不行 |
| `T` | 已停止（作业控制） | 需 `SIGCONT` |
| `Z` | **僵尸**（已终止，父进程未 wait） | ❌ 已经死了 |

### 常用信号

| 编号 | 名称 | 说明 |
| --- | --- | --- |
| 1 | `SIGHUP` | 终端挂断；daemon 约定用它表示"重读配置" |
| 2 | `SIGINT` | `Ctrl+C` |
| 9 | `SIGKILL` | **不可捕获、不可忽略、不可阻塞** |
| 11 | `SIGSEGV` | 段错误（访问非法内存） |
| 13 | `SIGPIPE` | 向已关闭的管道写数据 |
| 15 | `SIGTERM` | **默认 kill 信号**，可捕获，用于优雅退出 |
| 17 | `SIGCHLD` | 子进程状态变化 |
| 18 | `SIGCONT` | 继续运行 |
| 19 | `SIGSTOP` | 停止（不可捕获） |
| 20 | `SIGTSTP` | `Ctrl+Z`（可捕获） |

> 脚本里应写信号**名字**（`kill -TERM`）而不是数字，因为编号在不同架构上可能不同。

---

## 五、Linux —— 权限与用户

### 权限

| 命令 | 说明 |
| --- | --- |
| `chmod 755 <file>` | `rwxr-xr-x`：目录、可执行程序 |
| `chmod 644 <file>` | `rw-r--r--`：普通文件 |
| `chmod 600 <file>` | `rw-------`：私密文件、密钥 |
| `chmod 700 <dir>` | `rwx------`：私有目录 |
| `chmod 664` / `775` | 组内协作的文件 / 目录 |
| `chmod u+x <file>` | 给属主加执行权 |
| `chmod g-w <file>` | 移除属组的写权 |
| `chmod o=r <file>` | **精确设为** other 只读（`=` 是赋值，会清掉原有位）⚠️ |
| `chmod -R 755 <dir>` | 递归应用 |
| `chmod -R g+rwX <dir>` | 大写 `X`：只给目录和已可执行的文件加 x ⭐ |
| `chmod 4755` / `u+s` | 设 SUID（执行时 euid 变成文件属主） |
| `chmod 2755` / `g+s` | 设 SGID（对目录：新建对象继承目录属组） |
| `chmod 1777` / `+t` | 设 sticky（只能删自己的文件，如 `/tmp`） |
| `chown <user>:<group> <file>` | 改属主与属组 |
| `chown -R <user> <dir>` | 递归改属主（修 `sudo` 产生的文件） |
| `chgrp <group> <file>` | 只改属组 |
| `umask` | 查看掩码；`umask 027` 设置 |
| `umask -S` | 以符号显示 |

**权限数字速记**：`r=4, w=2, x=1`，三组依次是属主/属组/其他。`755`=目录/程序，`644`=文件，`600`=密钥。

**特殊位速记**：`4`=SUID，`2`=SGID，`1`=sticky，写在最前面（`4755`、`2775`、`1777`）。

**`ls -l` 特殊位显示**：小写 `s`/`t` = 特殊位与执行位都设了；**大写 `S`/`T` = 特殊位设了但执行位没设（该位实际不生效）** ⚠️。

### 用户与组

| 命令 | 说明 |
| --- | --- |
| `id [user]` | 查看 UID/GID 与所有组 ⭐ |
| `groups [user]` | 简化的组列表 |
| `whoami` | 当前用户名 |
| `getent passwd <user>` | 通过 NSS 查用户（含 LDAP 等外部源） |
| `getent group <group>` | 查组及其成员 |
| `useradd -m -s /bin/bash -G sudo,devs <u>` | 创建用户（**`-m` 才会建家目录**）⚠️ |
| `adduser <u>` | 更友好的交互式封装 **[Debian/Ubuntu]** |
| `usermod -aG <group> <u>` | **追加**到附加组（`-a` 不能省！）⚠️⚠️ |
| `usermod -G <groups> <u>` | ⚠️ **替换**整个附加组列表（会丢掉原有的组） |
| `usermod -L / -U <u>` | 锁定 / 解锁账户 |
| `userdel -r <u>` | 删除用户及家目录 |
| `groupadd` / `groupdel` | 创建 / 删除组 |
| `gpasswd -a <u> <g>` | 添加组成员 |
| `gpasswd -d <u> <g>` | 移除组成员 |
| `passwd [u]` | 改密码（root 改别人不需旧密码） |
| `passwd -l / -u` | 锁定 / 解锁 |
| `chage -l <u>` | 查看密码过期策略 |
| `chage -M 90 -W 7 <u>` | 设最长 90 天、提前 7 天警告 |

**关键提醒**：改完组必须**重新登录**才生效（进程的组列表在登录时确定）。

### `su` 与 `sudo`

| 命令 | 说明 |
| --- | --- |
| `su -` | 切成 root 并模拟完整登录（重置环境、切家目录）⭐ |
| `su` | 切成 root 但**保留当前环境与目录** ⚠️ |
| `su - <user>` | 切成指定用户 |
| `sudo <cmd>` | 以 root 执行单条命令（要**自己的**密码） |
| `sudo -i` | 启动 root 的登录 shell |
| `sudo -s` | 启动 root 的非登录 shell（保留环境） |
| `sudo -u <user> <cmd>` | 以指定用户执行 |
| `sudo -l` | **列出自己有哪些 sudo 权限** ⭐ |
| `sudo -k` | 立即失效时间戳（下次需重新输密码） |
| `sudo -n <cmd>` | 非交互（需要密码就失败） |
| `sudo visudo` | 编辑 sudoers 的**唯一正确方式**（有语法检查）⭐ |
| `sudo visudo -f /etc/sudoers.d/myapp` | 编辑 drop-in 文件 |
| `sudo !!` | 用 sudo 重跑上一条命令 |

**sudoers 一行结构**：`谁 = (可变成哪些用户:哪些组) [NOPASSWD:] 可执行哪些命令`

```text
%sudo  ALL=(ALL:ALL) ALL                                    # 组内成员全权限
alice  ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx # 免密执行特定命令
```

> ⚠️ 命令规格要写**绝对路径并限定参数**。允许 `sudo vim` 或 `sudo systemctl`（不限子命令）等价于给了完整 root 权限。

### 排障

| 命令 | 说明 |
| --- | --- |
| `namei -l <path>` | 显示路径上每级目录的权限（排查穿越失败）⭐ |
| `lsattr <file>` | 查看文件属性（`i` = immutable，连 root 都不能改） |
| `chattr -i <file>` | 去掉 immutable 属性 |
| `mount \| grep noexec` | 检查挂载选项（`noexec`/`nosuid`/`ro` 会否决权限配置） |
| `getenforce` / `aa-status` | SELinux / AppArmor 状态 |
| `getfacl` / `setfacl -m u:bob:rwx <f>` | 查看 / 设置 ACL |

---

## 六、Linux —— 包管理与服务

### Debian / Ubuntu（apt / dpkg）

| 命令 | 说明 |
| --- | --- |
| `sudo apt update` | **只刷新索引**（不安装任何东西）⭐ |
| `sudo apt upgrade` | 升级（**绝不增删包**） |
| `sudo apt full-upgrade` | 升级（允许为依赖增删包）⚠️ |
| `sudo apt install <pkg>` | 安装（`-y` 免交互） |
| `sudo apt install --dry-run <pkg>` | 只显示会做什么 ⭐ |
| `sudo apt install <pkg>=<ver>` | 安装指定版本 |
| `sudo apt install ./x.deb` | 装本地 deb 并自动解决依赖 |
| `sudo apt remove <pkg>` | 卸载但**保留配置** |
| `sudo apt purge <pkg>` | 连配置一起删 ⭐ |
| `sudo apt autoremove --purge` | 删除孤儿包及其配置 |
| `sudo apt clean` / `autoclean` | 清理 `.deb` 缓存 |
| `apt search <kw>` | 搜索 |
| `apt show <pkg>` | 包详情 |
| `apt list --installed` | 已安装列表 |
| `apt list --upgradable` | 可升级的包 |
| `apt-cache policy <pkg>` | **可用版本与实际会装的版本** ⭐ |
| `apt-cache rdepends <pkg>` | 反向依赖（删包前必查） |
| `sudo apt-mark hold <pkg>` | 锁定版本不升级 |
| `dpkg -l \| grep <pkg>` | 已安装的包及状态（`ii` = 正常） |
| `dpkg -L <pkg>` | **列出包安装了哪些文件** ⭐ |
| `dpkg -S <path>` | **查某个文件属于哪个包** ⭐⭐ |
| `sudo dpkg -i x.deb` | 安装本地 deb（**不解决依赖**）⚠️ |
| `sudo apt --fix-broken install` | 修复依赖损坏 |
| `sudo dpkg --configure -a` | 完成被中断的配置 |

> ⚠️ **`Could not get lock /var/lib/dpkg/lock-frontend`**：先 `ps aux | grep -E '[a]pt|[d]pkg'` 确认持有者（可能是 `unattended-upgrades` 在正常工作），**等它完成**。**绝不要删锁文件。**

### RHEL 系（dnf / yum / rpm）

| 命令 | 说明 |
| --- | --- |
| `sudo dnf check-update` | 检查可更新 |
| `sudo dnf upgrade` | 升级（**会为依赖移除包**，与 apt 不同）⚠️ |
| `sudo dnf install / remove <pkg>` | 安装 / 卸载 |
| `sudo dnf search <kw>` / `info` | 搜索 / 详情 |
| `sudo dnf list installed` | 已安装列表 |
| `sudo dnf provides */<file>` | **反查哪个包提供这个文件** ⭐ |
| `sudo dnf repoquery -l <pkg>` | 列出包内文件 |
| `sudo dnf clean all` / `makecache` | 清理 / 重建缓存 |
| `sudo dnf history` | 操作历史 |
| `sudo dnf history undo <id>` | **回滚某次操作**（apt 没有此能力）⭐ |
| `sudo dnf config-manager --set-enabled/--set-disabled <repo>` | 启用/禁用仓库 |
| `rpm -qa \| grep <pkg>` | 已安装的包 |
| `rpm -qf <path>` | 查文件属于哪个包 |
| `rpm -ql <pkg>` | 列出包内文件 |
| `rpm -V <pkg>` | **校验包内文件是否被篡改**（安全审计）⭐ |
| `rpm -q --scripts <pkg>` | 查看包带的安装脚本 |
| `sudo rpm --import <key>` | 导入 GPG 公钥 |

> ⚠️ 不要关闭 `gpgcheck`。签名验证失败应导入正确的公钥，而不是跳过校验。

### systemd

| 命令 | 说明 |
| --- | --- |
| `systemctl start / stop / restart <svc>` | 启动 / 停止 / 重启 |
| `systemctl reload <svc>` | 重载配置（不断连接，需服务支持） |
| `systemctl reload-or-restart <svc>` | 支持就 reload，否则 restart ⭐ |
| `systemctl status <svc>` | 状态 + 最近日志 + PID |
| `systemctl enable <svc>` | **设为开机自启（不启动！）** ⚠️ |
| `systemctl disable <svc>` | 取消自启（**不停止！**） ⚠️ |
| `systemctl enable --now <svc>` | 启用**并**立即启动 ⭐ |
| `systemctl disable --now <svc>` | 禁用**并**立即停止 |
| `systemctl mask <svc>` | **彻底屏蔽**（连依赖都拉不起来） |
| `systemctl unmask <svc>` | 解除屏蔽 |
| `systemctl is-active / is-enabled <svc>` | 只输出状态（脚本用） |
| `systemctl list-units --type=service` | 当前服务 |
| `systemctl list-unit-files` | 所有 unit 及自启状态 |
| `systemctl --failed` | **只看失败的**（排障第一步）⭐⭐ |
| `sudo systemctl daemon-reload` | **改完 unit 文件必须执行** ⚠️⚠️ |
| `systemctl cat <svc>` | 显示 unit 文件内容（含覆盖） |
| `systemctl show <svc>` | 显示**实际生效**的所有属性 ⭐ |
| `systemctl edit <svc>` | 创建 drop-in 覆盖（推荐方式） |
| `systemctl revert <svc>` | 撤销本地修改 |
| `systemd-analyze verify <unit>` | 校验 unit 文件语法并给出建议 ⭐ |
| `systemctl get-default` | 当前默认 target |
| `sudo systemctl set-default multi-user.target` | 设为无图形界面启动 |
| `sudo systemctl isolate multi-user.target` | 立即切换 target（不重启） |

### journalctl

| 命令 | 说明 |
| --- | --- |
| `journalctl -u <svc>` | 某服务的日志 |
| `journalctl -u <svc> -f` | 持续跟踪 |
| `journalctl -u <svc> -n 100` | 最近 100 行 |
| `journalctl -u <svc> --since "1 hour ago"` | 按时间过滤 |
| `journalctl -u <svc> -p err` | 只看 err 及以上（**数字越小越严重**）⚠️ |
| `journalctl -b` | 本次启动的日志 |
| `journalctl -b -1` | 上次启动（排障利器）⭐ |
| `journalctl -k` | 只看内核日志（同 `dmesg`） |
| `journalctl --disk-usage` | 日志占用磁盘 |
| `journalctl --vacuum-size=500M` | 清理到 500M |
| `journalctl --vacuum-time=30d` | 只保留 30 天 |
| `journalctl --list-boots` | 列出所有启动记录 |
| `journalctl _PID=<pid>` | 按 PID 过滤 |

> ⚠️ **journald 默认可能不持久化**（日志在 `/run` 这个 tmpfs 里，重启即丢）。要保留历史日志需创建 `/var/log/journal/` 或设 `Storage=persistent`。

### 服务排障顺序

```text
1. systemctl status <svc>
2. journalctl -u <svc> -n 100 --no-pager
3. systemd-analyze verify /etc/systemd/system/<svc>.service
4. sudo -u <service-user> <ExecStart 的完整命令>     ← 最关键的一步
5. ss -lntp | grep <port>                            ← 端口占用
6. systemctl show <svc> > /tmp/eff.txt; less /tmp/eff.txt   ← 看实际生效配置
```

---

## 七、Linux —— Shell 脚本要点

| 语法 | 说明 |
| --- | --- |
| `#!/usr/bin/env bash` | shebang（比 `#!/bin/bash` 更可移植） |
| `set -euo pipefail` | 遇错退出 + 未定义变量报错 + 管道任一失败即失败 ⭐ |
| `"$VAR"` vs `$VAR` | **不确定就加双引号**（避免词分隔与通配符展开）⭐⭐ |
| `'单引号'` | 完全不展开（变量、命令替换、通配符都是字面量） |
| `"双引号"` | 展开变量与命令替换，但不做通配符展开 |
| `name="value"` | 赋值（**等号两边不能有空格**）⚠️ |
| `${VAR:-default}` | 未设置或为空时用默认值 |
| `${VAR:=default}` | 同上并**赋值**给 VAR |
| `${VAR:?msg}` | 未设置则报错退出 |
| `${#VAR}` | 字符串长度 |
| `${VAR:0:3}` | 取前 3 个字符 |
| `${path##*/}` | 取文件名（删最长前缀 `*/`） |
| `${path%/*}` | 取目录（删最短后缀 `/*`） |
| `${file%.*}` / `${file##*.}` | 去扩展名 / 取扩展名 |
| `${VAR//old/new}` | 全部替换 |
| `$(cmd)` | 命令替换（**优先于反引号**）⭐ |
| `$((n + 1))` | 算术（**只支持整数！**）⚠️ |
| `echo "scale=2; 10/3" \| bc` | 浮点运算（或用 `awk`） |
| `$0 $1 $# $@ $? $$ $!` | 脚本名 / 参数 / 参数个数 / 全部参数 / 上个退出码 / 当前 PID / 最后后台 PID |
| `"$@"` | **转发参数的正确方式**（保持每个参数独立）⭐⭐ |
| `[ ]` vs `[[ ]]` | POSIX 命令 vs bash 关键字（后者更安全，支持 `&&`/`\|\|`/`=~`） |
| `[ "$a" = "$b" ]` | 字符串相等 |
| `[ "$a" -gt 5 ]` | 整数比较（**不能用 `>`**！`>` 会被当重定向）⚠️⚠️ |
| `-z` / `-n` | 长度为零 / 非零 |
| `-e -f -d -r -w -x -s -L` | 存在 / 普通文件 / 目录 / 可读 / 可写 / 可执行 / 非空 / 符号链接 |
| `if cmd; then` | 用命令退出码作条件 |
| `case "$1" in ... esac` | 分支匹配（支持 `*` `?` `[...]` 与 `\|`） |
| `for i in {1..10..2}` | 范围循环（步长 2，bash 特性） |
| `while IFS= read -r line; do ...; done < file` | **读文件的标准写法** ⭐⭐ |
| `done < <(cmd)` | 进程替换（避免子 shell 丢变量）⭐ |
| `cat f \| while read` | ⚠️ **错误**：循环在子 shell 里，变量修改会丢失 |
| `shopt -s nullglob` | 通配符无匹配时展开成空（而不是字面量） |
| `f() { local x=1; }` | 函数定义；**`local` 必不可少** |
| `return 0` | 只返回 0-255 的整数状态码（**不是值**）⚠️ |
| `result=$(f)` | 用 echo + 命令替换返回值 |
| `printf '%s\n' "$var"` | 优先用 `printf`（`echo -e`/`-n` 不是 POSIX 标准）⭐ |
| `printf '%q\n' "$var"` | 转义成可安全复制回 shell 的形式 |
| `<<EOF` | here-doc（变量**会**展开） |
| `<<'EOF'` | here-doc（**引号阻止展开**，内容含 `$` 时必须用）⚠️ |
| `<<-EOF` | 去掉行首 **Tab**（不是空格） |
| `<<< "str"` | here-string |
| `read -r -p "提示: " var` | 读输入（`-r` 不解释反斜杠） |
| `read -r -s -p "密码: " p` | 静默读（不回显） |
| `read -r -t 5 var` | 5 秒超时 |
| `trap 'cleanup' EXIT` | 退出时清理（**脚本质量的重要标志**）⭐ |
| `trap '...' ERR INT TERM` | 其他 trap 目标 |
| `bash -x script.sh` | 打印每条执行的命令 |
| `set -x` / `set +x` | 局部开启/关闭 trace |
| `export PS4='+ ${LINENO}: '` | 让 trace 带行号 ⭐ |
| `shellcheck script.sh` | **必备的静态检查工具** ⭐⭐ |
| `command -v <cmd>` | 检查命令是否存在（比 `which` 可移植） |

**`set -e` 的例外**（这些情况下不触发退出）：`if` 条件里、`&&`/`||` 左侧、`!` 取反、函数在条件中被调用。

**`((var++))` 的陷阱**：`var` 为 0 时返回假（退出码 1），配合 `set -e` 会让脚本意外退出。用 `((var++)) || true` 或 `var=$((var+1))`。⚠️

---

## 八、Linux —— 网络与远程

| 命令 | 说明 |
| --- | --- |
| `ip a` | 查看网卡与 IP 地址 ⭐ |
| `ip route` | 查看路由表与默认网关 |
| `ip link set <dev> up/down` | 启用/禁用网卡 |
| `nmcli device status` | NetworkManager 设备状态 |
| `nmcli connection up "<name>"` | 启用连接 |
| `ping -c 3 <host>` | 发 3 个 ICMP 包 |
| `curl -s -o /dev/null -w '%{http_code}' <url>` | 只取 HTTP 状态码 ⭐ |
| `curl -I <url>` | 只看响应头 |
| `curl -m 5 <url>` | 5 秒超时（**健康检查必须设**） |
| `curl -X POST -H 'Content-Type: application/json' -d '{}' <url>` | 发 POST JSON |
| `wget -c <url>` | 断点续传下载 |
| `dig <domain>` / `nslookup` | DNS 查询（需 dnsutils） |
| `resolvectl status` | systemd-resolved 的 DNS 状态 |
| `cat /etc/resolv.conf` | DNS 配置 |
| `ss -lntp` | 监听端口 |
| `ssh <user>@<host>` | 登录（`-p` 指定端口，`-v` 调试） |
| `ssh-keygen -t ed25519 -C "comment"` | 生成密钥对（ed25519 是推荐算法） |
| `ssh-copy-id <user>@<host>` | 把公钥复制到远端 |
| `scp <file> <user>@<host>:<path>` | 复制文件（`-r` 递归） |
| `rsync -avz <src>/ <user>@<host>:<dst>/` | 增量同步（`-n` 演练）⭐ |
| `rsync -avz --delete <src>/ <dst>/` | 删除目标端多余文件 ⚠️ |
| `sudo sshd -t` | **校验 sshd 配置语法**（改完必做）⭐⭐ |
| `sudo systemctl restart ssh` | 重启 SSH（**Debian/Ubuntu 是 `ssh`，RHEL 系是 `sshd`**）⚠️ |
| `tmux new -s <name>` / `attach -t` / `ls` | 终端复用（长任务必备）⭐ |
| `timedatectl set-timezone Asia/Shanghai` | 设置时区 |
| `timedatectl` | 时区与 NTP 同步状态 |

**rsync 路径语义**：`rsync -av src/ dst/` 同步**内容**；`rsync -av src dst/` 同步**目录本身**。⚠️

**SSH 私钥权限**：`chmod 700 ~/.ssh`、`chmod 600 ~/.ssh/id_ed25519`、`chmod 600 ~/.ssh/authorized_keys`。**权限太开放 `ssh` 会拒绝使用。** ⚠️

---

## 九、SQL —— 查询

### 基本查询

| 语句 | 说明 |
| --- | --- |
| `SELECT col1, col2 FROM t;` | 投影（**避免 `SELECT *`**，但探索数据时可用） |
| `SELECT DISTINCT col FROM t;` | 去重（作用于**整行**） |
| `SELECT col AS alias FROM t;` | 列别名（`AS` 可省） |
| `SELECT COUNT(DISTINCT col) FROM t;` | 数不同值的个数 |
| `SELECT * FROM t WHERE cond;` | 行筛选 |
| `... WHERE a = 1 AND (b = 2 OR c = 3)` | **混用 AND/OR 必须加括号**（AND 优先级更高）⚠️ |
| `... WHERE col BETWEEN 10 AND 20` | 闭区间；**不匹配 NULL** |
| `... WHERE col IN (1, 2, 3)` | 在集合中 |
| `... WHERE col NOT IN (1, 2)` | ⚠️ 子查询含 NULL 时**结果为空集** |
| `... WHERE col IS NULL` / `IS NOT NULL` | **判断 NULL 的唯一正确方式**（`= NULL` 永远不成立）⚠️ |
| `... WHERE name LIKE 'A%'` | `%` 任意长度，`_` 单个字符 |
| `... WHERE name LIKE '%x%'` | ⚠️ **前置通配符用不上索引** |
| `... WHERE a LIKE '%\%%' ESCAPE '\'` | 转义字面量 `%`/`_` |
| `... ORDER BY col DESC` | 降序（默认 ASC） |
| `... ORDER BY a ASC, b DESC` | 多列排序（每列各自带方向） |
| `... ORDER BY col ASC NULLS LAST` | 显式控制 NULL 位置（**PostgreSQL / Oracle 支持**） |
| `... ORDER BY (col IS NULL), col` | ⭐ 可移植的 NULL 排序（MySQL / SQLite 写法） |
| `... LIMIT 10` | 取前 10 条（**必须有 ORDER BY 才有确定意义**）⚠️ |
| `... LIMIT 10 OFFSET 20` | 分页（跳过 20 条） |
| `... ORDER BY score DESC, id ASC LIMIT 10` | ⭐ **稳定的分页**（排序键必须唯一） |
| `... OFFSET 20 ROWS FETCH FIRST 10 ROWS ONLY` | 标准 SQL 写法 |

### 逻辑执行顺序 ⭐⭐⭐

```text
书写:  SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
逻辑:  FROM → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT
```

由此推出：`WHERE` 不能用聚合函数、通常不能用 `SELECT` 别名；`ORDER BY` 可以用别名；窗口函数不能在 `WHERE` 里用。

### 连接

| 语句 | 说明 |
| --- | --- |
| `A INNER JOIN B ON a.id = b.a_id` | 只保留匹配上的（= 笛卡尔积 + 过滤） |
| `A LEFT JOIN B ON ...` | 保留左表全部，右表缺失处补 NULL |
| `A RIGHT JOIN B ON ...` | 保留右表全部（= 交换顺序的 LEFT JOIN） |
| `A FULL OUTER JOIN B ON ...` | 两边都保留（**MySQL 不支持**，需 UNION 模拟） |
| `A CROSS JOIN B` | 笛卡尔积（**有意用它构造完整框架**） |
| `A JOIN B USING (col)` | 简写；**结果里连接列会合并成一列** |
| `A NATURAL JOIN B` | ⚠️ **不要用**：隐式按所有同名列连接，加列会悄悄改语义 |
| `A LEFT JOIN B ON ... AND <条件>` | ⭐ 条件在 `ON` 里：不匹配的行仍保留（补 NULL） |
| `A LEFT JOIN B ON ... WHERE b.col = x` | ⚠️ **退化成 INNER JOIN**（补 NULL 的行被过滤掉） |
| `A LEFT JOIN B ON ... WHERE b.id IS NULL` | ⭐ **反连接**：找"只在 A 中"的记录 |
| `A s1 JOIN A s2 ON s1.k = s2.k AND s1.id < s2.id` | 自连接配对（用 `<` 避免重复对与自配对）⭐ |

**MySQL 模拟 FULL OUTER JOIN**：

```sql
SELECT ... FROM a LEFT JOIN b ON ...
UNION                                    -- 必须用 UNION（不是 UNION ALL）
SELECT ... FROM a RIGHT JOIN b ON ...;
```

### 聚合与分组

| 语句 | 说明 |
| --- | --- |
| `COUNT(*)` | 数**所有行**（不检查任何列） |
| `COUNT(col)` | 数 `col` **非 NULL** 的行数 ⭐ |
| `COUNT(DISTINCT col)` | 数不同的非 NULL 值 |
| `SUM(col)` / `AVG(col)` | 求和 / 平均（**跳过 NULL**） |
| `AVG(col)` | = `SUM(col) / COUNT(col)`，**分母是非 NULL 行数** ⚠️ |
| `SUM(COALESCE(col,0)) / COUNT(*)` | 把 NULL 当 0 算的平均 ⭐ |
| `MIN(col)` / `MAX(col)` | 最小 / 最大（跳过 NULL，也可用于字符串和日期） |
| `COALESCE(SUM(x), 0)` | ⭐ **`SUM` 无行时返回 NULL，不是 0** |
| `GROUP BY col` | 分组（所有 NULL 归为**同一组**） |
| `GROUP BY col1, col2` | 多列分组（按组合分桶） |
| `GROUP BY ROLLUP (a, b)` | 加小计与总计行（SQLite 不支持） |
| `HAVING COUNT(*) >= 2` | 对**分组**过滤（此处可以用聚合函数） |
| `HAVING AVG(x) > (SELECT AVG(x) FROM t)` | 与全体平均比较（不能写 `AVG(x) > AVG(x)`）⭐ |
| `COUNT(*) - COUNT(col)` | **数 NULL 的个数** ⭐（`COUNT(col IS NULL)` 是错的）⚠️ |
| `SUM(CASE WHEN cond THEN 1 ELSE 0 END)` | 条件计数（可移植） |
| `NULLIF(COUNT(*), 0)` | 防除零 ⭐ |
| `ROUND(x, 2)` | 保留 2 位小数 |

**`WHERE` vs `HAVING`**：`WHERE` 过滤**行**（分组前，不能用聚合）；`HAVING` 过滤**组**（分组后，能用聚合）。

**`SELECT` 列表只能出现**：分组键 + 聚合函数。想要"组内某行的其他列"→ 用窗口函数或再连接回去。

### 窗口函数（MySQL 8.0+ / PostgreSQL / SQLite 3.25+）

| 语句 | 说明 |
| --- | --- |
| `ROW_NUMBER() OVER (PARTITION BY g ORDER BY x DESC)` | 唯一序号（并列也按顺序编号） |
| `RANK()` | 并列同名次并**跳跃**（1,1,3） |
| `DENSE_RANK()` | 并列同名次**不跳跃**（1,1,2） |
| `SUM(x) OVER (PARTITION BY g ORDER BY t)` | 组内累计和 |
| `AVG(x) OVER (PARTITION BY g)` | 组内平均（**行不折叠**） |
| `AVG(x) OVER ()` | 全体平均（不带 PARTITION 即整个结果集） |
| `LAG(x, 1) OVER (ORDER BY t)` / `LEAD(...)` | 前一行 / 后一行的值 |
| `FIRST_VALUE(x) OVER (...)` | 组内首值 |
| `LAST_VALUE(x) OVER (... ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)` | ⚠️ 组内末值**必须显式写窗口框架** |
| 取每组前 N | ⭐ 必须用 CTE/派生表包一层，**不能在 `WHERE` 里过滤窗口函数结果** |

### 子查询与 CTE

| 语句 | 说明 |
| --- | --- |
| `WHERE x > (SELECT AVG(x) FROM t)` | 标量子查询（**返回多行会报错**） |
| `WHERE x IN (SELECT ...)` | 集合成员（基本 NULL 安全） |
| `WHERE NOT IN (SELECT ...)` | ⚠️ 子查询含 NULL 时结果**恒为空** |
| `WHERE EXISTS (SELECT 1 FROM t2 WHERE t2.k = t1.k)` | 存在性判断（NULL 安全、找到即短路）⭐ |
| `WHERE NOT EXISTS (...)` | ⭐ **"排除"语义的推荐写法** |
| `WHERE x > ALL (SELECT ...)` | 等价于 `> MAX(...)`（**空集时恒为 TRUE**）⚠️ |
| `WHERE x > ANY (SELECT ...)` | 等价于 `> MIN(...)`；`= ANY` 等价于 `IN` |
| `FROM (SELECT ...) AS t` | 派生表（**必须写别名**，且不能在同一层引用两次） |
| `WITH t AS (SELECT ...) SELECT ... FROM t` | CTE（可复用、可单独调试）⭐ |
| `WITH a AS (...), b AS (SELECT ... FROM a) ...` | 多个 CTE，后面的可引用前面的 |
| `WITH RECURSIVE t AS (SELECT ... UNION ALL SELECT ... FROM t WHERE <终止>)` | ⭐ 递归 CTE（锚点 + UNION ALL + 递归成员） |
| `... WHERE level < 100` | ⭐ **递归 CTE 的深度保护**（防脏数据成环导致无限递归） |

### 数据定义（DDL）

| 语句 | 说明 |
| --- | --- |
| `CREATE TABLE t (id INT PRIMARY KEY, ...);` | 建表 |
| `CREATE TABLE IF NOT EXISTS t (...);` | 存在则不报错（Oracle 不支持） |
| `CONSTRAINT name PRIMARY KEY (a, b)` | 复合主键（保证**组合**唯一） |
| `col INT NOT NULL` | 非空 |
| `col INT DEFAULT 0` | 默认值（⚠️ **只在省略该列时生效**，显式写 `NULL` 不触发） |
| `col INT UNIQUE` | 唯一（✅ **允许多个 NULL 共存**） |
| `FOREIGN KEY (a) REFERENCES t2 (id) ON DELETE CASCADE` | 外键（`RESTRICT` 立即报错 / `NO ACTION` 可延迟到事务结束 / `SET NULL`） |
| `CHECK (score BETWEEN 0 AND 100)` | 检查约束（⚠️ **不阻止 NULL**，需配合 `NOT NULL`；MySQL 8.0.16 前被忽略） |
| `col NUMERIC(5,2)` | 精确定点数（⭐ **金额、成绩必须用它，不要用 `FLOAT`**） |
| `col VARCHAR(50)` | 变长字符串（PostgreSQL/MySQL 里是**字符数**，Oracle 默认是字节数） |
| `col BIGINT GENERATED BY DEFAULT AS IDENTITY` | 标准自增（MySQL 用 `AUTO_INCREMENT`，旧 PostgreSQL 用 `SERIAL`） |
| `col TIMESTAMPTZ DEFAULT CURRENT_TIMESTAMP` | 带时区时间戳（PostgreSQL） |
| `ALTER TABLE t ADD COLUMN c INT;` | 加列 |
| `ALTER TABLE t ALTER COLUMN c TYPE VARCHAR(100);` | 改类型（⚠️ 大表上可能长时间锁表） |
| `ALTER TABLE t ADD CONSTRAINT c CHECK (...);` | 加约束 |
| `ALTER TABLE t DROP CONSTRAINT c;` | 删约束（所以**约束要显式命名**） |
| `DROP TABLE t;` / `DROP TABLE IF EXISTS t;` | 删表 |
| `TRUNCATE TABLE t;` | 只清数据、保留结构（**自增计数器会重置**） |
| `CREATE INDEX idx_t_c ON t (c);` | 建索引 |
| `CREATE UNIQUE INDEX uq ON t (c);` | 唯一索引 |
| `CREATE INDEX idx ON t (a, b);` | 复合索引（⭐ **列顺序决定能否用上**） |
| `CREATE INDEX CONCURRENTLY idx ON t (c);` | 不长时间阻塞写入（PostgreSQL，**不能在事务块里**） |
| `CREATE INDEX idx ON t (c) WHERE cond;` | 部分索引（PostgreSQL / SQLite） |
| `CREATE INDEX idx ON t (LOWER(c));` | 表达式索引（PostgreSQL） |
| `DROP INDEX idx;` | 删索引 |
| `CREATE VIEW v AS SELECT ...;` | 创建视图（**不存数据、不提速**） |
| `CREATE MATERIALIZED VIEW mv AS SELECT ...;` | 物化视图（存数据，_MySQL 不支持_） |
| `CREATE OR REPLACE VIEW v AS ...;` | 修改视图定义 |
| `GRANT SELECT ON t TO u;` / `REVOKE ...` | 权限管理（DCL） |
| `BEGIN; ... COMMIT;` / `ROLLBACK;` | 事务控制 |

### 索引与优化

| 语句 / 要点 | 说明 |
| --- | --- |
| `EXPLAIN <query>` | 查看预估执行计划 ⭐ |
| `EXPLAIN ANALYZE <query>` | 真正执行并给出实际耗时与行数（PostgreSQL / MySQL 8+） |
| `ANALYZE <table>;` | 更新统计信息（PostgreSQL） |
| `ANALYZE TABLE t;` | 同上（MySQL） |
| **最左前缀原则** | 索引 `(a,b,c)` 能服务 `a`、`a+b`、`a+b+c`；**不能**服务 `b` 或 `b+c` |
| **范围条件截断后续列** | `WHERE a=1 AND b>10 AND c=5` 中 `c` 用不上索引 |
| **列上做函数/运算** | ⚠️ 索引用不上。`WHERE YEAR(d)=2005` → 改成 `WHERE d >= '2005-01-01' AND d < '2006-01-01'` ⭐ |
| **前置通配符** | ⚠️ `LIKE '%x%'` 用不上索引；`LIKE 'x%'` 可以 |
| **隐式类型转换** | ⚠️ 类型不匹配可能让索引失效，让传入类型与列类型一致 |
| **`OR` 连接不同列** | ⚠️ 考虑改写成 `UNION` |
| **选择性太低** | 性别、布尔标志这类列上的单列索引意义不大（考虑部分索引） |
| **外键列索引** | ⭐ **PostgreSQL 不会自动建，MySQL InnoDB 会自动建** |
| **覆盖索引** | 查询所需列全在索引里 → 免回表（`INCLUDE` 可把列排除出索引键） |
| **聚簇索引** | 叶子存行数据（每表最多一个）；InnoDB 主键索引即聚簇索引，二级索引叶子存主键值（**回表的来源**） |
| **`EXPLAIN` 看什么** | 访问方式（Seq Scan/ALL = 全表扫描）、预估 vs 实际行数、连接算法（两个大表的 Nested Loop 是警报）、有无 Sort/Using temporary、是否回表 |

### 常用内置函数

| 函数 | 说明 |
| --- | --- |
| `COALESCE(a, b, c)` | 返回第一个非 NULL 的值 ⭐ |
| `NULLIF(a, b)` | 相等则返回 NULL（常用于防除零） |
| `CAST(x AS type)` / `x::type` | 类型转换 |
| `ROUND(x, n)` | 四舍五入到 n 位小数 |
| `ABS` / `CEIL` / `FLOOR` / `MOD` | 绝对值 / 上取整 / 下取整 / 取余 |
| `LENGTH(s)` / `CHAR_LENGTH(s)` | 长度（注意 PostgreSQL 的 `LENGTH` 返回字符数） |
| `LOWER(s)` / `UPPER(s)` | 转小写 / 大写 |
| `TRIM` / `LTRIM` / `RTRIM` | 去空白 |
| `SUBSTRING(s, start, len)` | 取子串 |
| `REPLACE(s, from, to)` | 替换 |
| `CONCAT(a, b)` / `a \|\| b` | 拼接（**任一为 NULL 结果为 NULL**，用 `COALESCE` 处理） |
| `SPLIT_PART(s, delim, n)` | 按分隔符取第 n 段（PostgreSQL） |
| `NOW()` / `CURRENT_TIMESTAMP` | 当前时间 |
| `CURRENT_DATE` | 当前日期 |
| `EXTRACT(YEAR FROM d)` / `DATE_PART('year', d)` | 提取日期部分 |
| `DATE_TRUNC('month', d)` | 截断到月（PostgreSQL） |
| `d + INTERVAL '1 day'` | 日期加减（PostgreSQL / MySQL 语法略有不同） |
| `ROW_NUMBER()` 等窗口函数 | 见上面的窗口函数表 |

### NULL 行为速查 ⭐

| 场景 | 行为 |
| --- | --- |
| `NULL = NULL` | `UNKNOWN`（不是 TRUE） |
| `WHERE col = NULL` | 永远不返回行 |
| `WHERE col <> 'x'` | **也排除 `col IS NULL` 的行** ⚠️ |
| `COUNT(*)` | 数所有行（含 NULL） |
| `COUNT(col)` | 跳过 NULL |
| `SUM/AVG/MIN/MAX(col)` | 跳过 NULL |
| `SUM/AVG/MIN/MAX`（无行时） | 返回 **`NULL`**（`COUNT` 返回 0）⚠️ |
| `AVG(col)` 的分母 | **非 NULL 的行数**（不是总行数） |
| `UNIQUE` 约束 | **多个 NULL 不冲突** |
| `PRIMARY KEY` | 不允许 NULL |
| `DISTINCT` | 多个 NULL 只保留一个 |
| `GROUP BY` | 所有 NULL 归为同一组 |
| `ORDER BY` 中 NULL 位置 | **标准未规定**（PostgreSQL 视为最大、MySQL/SQLite 视为最小）⚠️ |
| `NOT IN (含 NULL 的子查询)` | **结果恒为空集** ⚠️⚠️ |
| `IN (含 NULL 的列表)` | 基本安全（匹配到就 TRUE） |
| `CHECK (col <= 100)` | NULL 时表达式为 UNKNOWN，**约束不阻止** ⚠️ |
| `CASE WHEN` 中的 NULL | 数值比较都不命中，**会掉进 `ELSE`** ⚠️ |
| 字符串拼接含 NULL | 结果为 NULL |
| `'x' \|\| NULL` | `NULL` |

---

## 十、几个"救命"组合

| 需求 | 命令 / 语句 |
| --- | --- |
| **Top N 统计** | `sort \| uniq -c \| sort -rn \| head` |
| **日志里访问量前 10 的 IP** | `awk '{print $1}' access.log \| sort \| uniq -c \| sort -rn \| head` |
| **统计各状态码分布** | `awk '{print $9}' access.log \| sort \| uniq -c \| sort -rn` |
| **找出最大的 10 个文件** | `find . -type f -printf '%s\t%p\n' \| sort -rn \| head -10` |
| **保持顺序去重** | `awk '!seen[$0]++' file` |
| **找出没选课的学生** | `SELECT s.* FROM students s LEFT JOIN enrollments e ON e.student_id=s.student_id WHERE e.enrollment_id IS NULL;` |
| **找出平均分高于全体的组** | `... GROUP BY g HAVING AVG(x) > (SELECT AVG(x) FROM t)` |
| **每组前 N 名** | `WITH r AS (SELECT *, ROW_NUMBER() OVER (PARTITION BY g ORDER BY x DESC) rn FROM t) SELECT * FROM r WHERE rn <= 3;` |
| **统计 NULL 个数** | `COUNT(*) - COUNT(col)` |
| **防除零** | `SUM(x) * 100.0 / NULLIF(COUNT(*), 0)` |
| **`sudo !!`** | 用 sudo 重跑上一条命令 |
| **服务起不来** | `systemctl status X` → `journalctl -u X` → **手动以相同身份跑 ExecStart** → `systemctl show X` |
| **进程杀不掉** | 先看是不是僵尸（`Z`）或 `D` 状态（`ps -o stat= -p PID`） |
| **磁盘满了但 `du` 对不上** | `lsof +L1` 找"已删除但仍被打开"的文件 |
| **远程长任务** | `tmux new -s work`（比 `nohup` 好，可重新接管） |
| **换源** | `lsb_release -cs` 确认代号 → 备份 → 只改 URI → `apt update` 验证 |

---

## 说明

- 本速查表与仓库里的细节笔记配套：Linux 部分对应 [notes/linux/](../notes/linux/README.md)，SQL 部分对应 [notes/sql/](../notes/sql/README.md)。需要理解"为什么"时请查对应笔记。
- 标注的方言差异（`[GNU]`、`[Debian/Ubuntu]`、`[RHEL 系]`）只是最常见的归属；不同版本可能有变化，**关键操作请查你所用的版本的官方文档**。
- 文中不含任何编造的统计数字或漏洞编号。
