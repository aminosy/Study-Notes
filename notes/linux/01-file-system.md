# Linux 文件系统与常用命令

本篇梳理 Linux 的目录结构 (FHS)、路径与文件类型的基本概念，以及 ls/cd/cp/mv/rm/find/ln 等最常用命令，最后讲清权限 rwx 与 chmod/chown 的数字表示法。

## 1. Linux 目录结构 (FHS)

Linux 的文件系统是一棵以 `/` 为根的**单根树 (single-rooted tree)**，没有 Windows 那种 `C:\` `D:\` 盘符的概念。新增的磁盘被**挂载 (mount)** 到树的某个目录上，从而成为这棵树的一部分。定义这套布局的标准叫 **FHS (Filesystem Hierarchy Standard)**。

| 目录 | 存放内容 | 举个具体例子 |
| --- | --- | --- |
| `/` | 根目录，一切路径的起点 | `/etc`、`/home` |
| `/bin` | 所有用户可用的基本命令 | `/bin/ls`、`/bin/cp` |
| `/sbin` | 系统管理命令，传统上主要给 root 用 | `/sbin/fdisk`、`/sbin/reboot` |
| `/lib`、`/lib64` | 共享库与内核模块，供 `/bin`、`/sbin` 使用 | `/lib/x86_64-linux-gnu/libc.so.6` |
| `/boot` | 启动所需文件：内核、initramfs、引导器配置 | `/boot/vmlinuz-*`、`/boot/grub/grub.cfg` |
| `/etc` | **系统级配置文件**（"etcetera"），几乎全是纯文本 | `/etc/passwd`、`/etc/ssh/sshd_config`、`/etc/fstab` |
| `/home` | 普通用户的家目录 | `/home/alice/.bashrc` |
| `/root` | root 用户的家目录（注意不在 `/home` 下） | `/root/.bashrc` |
| `/dev` | 设备文件，由 udev 动态生成 | `/dev/sda`、`/dev/null`、`/dev/tty0` |
| `/proc` | **虚拟文件系统**，内核暴露的进程与系统信息，不占磁盘 | `/proc/cpuinfo`、`/proc/1234/status` |
| `/sys` | 另一个虚拟文件系统，暴露设备与内核对象模型 | `/sys/class/net/eth0/` |
| `/run` | 运行期可变数据（PID 文件、socket），**基于内存，重启即清空** | `/run/systemd/`、`/run/sshd.pid` |
| `/tmp` | 临时文件，通常**任何用户都可写**（配合 sticky bit） | `/tmp/build-cache/` |
| `/var` | **variable**，运行中不断变化的数据 | `/var/log/syslog`、`/var/lib/dpkg/`、`/var/cache/apt/` |
| `/usr` | 用户空间的程序与只读数据，占据系统体积的绝大部分 | `/usr/bin/python3`、`/usr/share/doc/`、`/usr/local/` |
| `/opt` | 第三方独立软件包，整体放在一个目录里 | `/opt/google/chrome/` |
| `/srv` | 本机作为服务器对外提供的数据 | `/srv/ftp/`、`/srv/www/` |
| `/mnt` | 临时挂载点，管理员手动挂载用 | `/mnt/usb/` |
| `/media` | 可移动介质的自动挂载点 | `/media/alice/USB-DRIVE/` |

### 1.1 关于 `/bin`、`/sbin`、`/lib` 的现代变化

在**现代主流发行版**上（包括近年的 Debian/Ubuntu、Fedora、Arch 等），`/bin`、`/sbin`、`/lib` 实际上是指向 `/usr` 下对应目录的**符号链接**：

```bash
$ ls -ld /bin /sbin /lib
lrwxrwxrwx 1 root root 7 ... /bin -> usr/bin
lrwxrwxrwx 1 root root 8 ... /sbin -> usr/sbin
lrwxrwxrwx 1 root root 7 ... /lib -> usr/lib
```

这个变化叫 **usrmerge**：历史上 `/bin` 是为了在 `/usr` 尚未挂载时也能提供基本命令（早期 Unix 里 `/usr` 常常是单独的分区），但现代系统的启动流程不再需要这个区分，于是合并到 `/usr` 简化维护。**这意味着 `/bin/ls` 和 `/usr/bin/ls` 是同一个文件**——但写脚本时两者的写法都能用。

### 1.2 `/usr/local` 的用途

值得单独一提：`/usr/local/bin`、`/usr/local/lib` 是**给管理员手工安装的软件**用的位置，优先级通常高于 `/usr/bin`。自己编译安装的工具放在这里，好处是与发行版包管理器管理的文件互不干扰，不会被 `apt upgrade` 覆盖。这是"从源码编译安装"时的默认前缀 (`./configure --prefix=/usr/local`)。

## 2. 一切皆文件 (everything is a file)

Linux 把设备、管道、socket 都抽象成文件，因此可以用同一套 `open/read/write/close` 系统调用来操作它们。`ls -l` 输出的**第一个字符**就是文件类型：

| 首字符 | 类型 | 说明 | 例子 |
| --- | --- | --- | --- |
| `-` | 普通文件 (regular) | 文本、二进制、图片…… | `/etc/passwd` |
| `d` | 目录 (directory) | 本质是"名字 → inode 号"的映射表 | `/etc` |
| `l` | 符号链接 (symbolic link) | 存的是另一个路径的字符串 | `/bin` → `usr/bin` |
| `b` | 块设备 (block device) | 按块随机访问，有缓冲 | `/dev/sda` |
| `c` | 字符设备 (character device) | 按字节流访问 | `/dev/tty`、`/dev/null` |
| `p` | 命名管道 (FIFO) | 进程间通信，先进先出 | `mkfifo mypipe` 创建 |
| `s` | 套接字 (socket) | 进程间/网络通信 | `/run/docker.sock` |

## 3. 路径 (path) 概念

| 概念 | 含义 | 例子 |
| --- | --- | --- |
| 绝对路径 | 从 `/` 开始，与当前目录无关 | `/etc/nginx/nginx.conf` |
| 相对路径 | 从当前工作目录开始 | `../bin/run.sh` |
| `.` | 当前目录 | `./script.sh`（明确指当前目录里的脚本） |
| `..` | 上一级目录 | `cd ..` |
| `~` | 当前用户的家目录，由 shell 展开 | `~/notes` → `/home/alice/notes` |
| `~user` | 指定用户的家目录 | `~bob` |
| `-` | 上一个工作目录（仅 `cd` 与少数命令支持） | `cd -` |

一个容易被忽略的点：**`~`、`*`、`?` 是 shell 展开的，不是命令自己处理的**。所以 `cp ~/a.txt /tmp` 里，`~` 在命令执行前就已经被替换成了 `/home/alice/a.txt`；而给一个程序传 `"~/a.txt"`（加引号）时，`~` 不会被展开，程序收到的是字面量。这也是脚本里要小心引号的原因之一（详见 [06-shell-scripting.md](06-shell-scripting.md)）。

## 4. 浏览与操作文件

### 4.1 `ls` —— 列表

```bash
ls                  # 列出当前目录内容
ls -l               # 长格式：权限、链接数、属主、属组、大小、时间、名字
ls -a               # 显示以 . 开头的隐藏文件（以及 . 和 ..）
ls -lh              # 大小变成人类可读（K/M/G）
ls -lt              # 按修改时间排序，最新的在最前
ls -ltr             # 反转时间序，最旧的在前（看最近改了什么的常用写法）
ls -lS              # 按文件大小排序，最大在前
ls -R               # 递归列出子目录
ls -i               # 显示 inode 号（排查硬链接时必用）
ls -ld /etc         # 看目录自身的信息，而不是目录里的内容
```

**`ls -l` 一行怎么读**（这是必须掌握的）：

```text
-rwxr-xr--  1  alice  devs   4096  Mar 12 10:30  deploy.sh
│└┬┘└┬┘└┬┘  │   │      │      │        │          │
│ │  │  │   │   │      │      │        │          └─ 文件名
│ │  │  │   │   │      │      │        └─ 最后修改时间
│ │  │  │   │   │      │      └─ 文件大小（字节，或 -h 下的 4.0K）
│ │  │  │   │   │      └─ 属组 (group)
│ │  │  │   │   └─ 属主 (owner)
│ │  │  │   └─ 硬链接计数（目录则是子目录数）
│ │  │  └─ 其他用户 other 的权限 (r--)
│ │  └─ 属组 group 的权限 (r-x)
│ └─ 属主 user 的权限 (rwx)
└─ 文件类型：- 普通文件
```

### 4.2 `cd` 与 `pwd`

```bash
pwd                 # print working directory，打印当前绝对路径
cd /var/log         # 用绝对路径进入
cd log              # 用相对路径进入（假设当前在 /var）
cd ..               # 上一级
cd ~                # 回家目录
cd -                # 回到上一个目录（在两个目录间来回切换很有用）
cd                  # 不带参数也是回家目录
```

### 4.3 `cp` / `mv` / `rm`

```bash
cp src.txt dst.txt              # 复制文件
cp -r dir1 dir2                 # 复制目录（-r 递归，必须）
cp -i src.txt existing.txt      # 覆盖前询问（interactive）
cp -p src.txt dst.txt           # 保留权限、属主、时间戳（preserve）
cp -a dir1 dir2                 # 归档式复制 = -dR --preserve=all，备份常用
cp -u src.txt dst.txt           # 只在 src 更新时复制（update）
cp *.log /tmp/archive/          # 多个源文件复制到目标目录
```

```bash
mv old.txt new.txt              # 重命名
mv file.txt /tmp/               # 移动到目录
mv -i a.txt b.txt               # 覆盖前询问（建议给 mv 设别名加 -i）
mv dir/ /tmp/                   # 移动目录不需要 -r
```

`mv` 的一个实用细节：**同一文件系统内的 `mv` 是瞬时的**，因为它只改目录项（重命名 inode 的指向），数据一个字节都不搬；跨文件系统时 `mv` 退化为"复制 + 删除"，大文件会很慢。这也解释了为什么"同一个盘里 mv 一个 10 GB 文件"和"跨盘 mv"体验完全不同。

```bash
rm file.txt                     # 删除文件
rm -i file.txt                  # 删除前询问
rm -r dir/                      # 递归删除目录
rm -rf dir/                     # 强制递归删除，不询问
rm -f *.tmp                     # 强制删除（文件不存在也不报错）
```

> ⚠️ **`rm` 没有回收站**。删掉就是删掉了（严格说是断了目录项、inode 引用计数减到 0，数据块被标记为空闲）。三个必须记住的纪律：
> 1. **永远不要**在 `/` 或 `$HOME` 下随手执行 `rm -rf *`，尤其不要写 `rm -rf /`（现代 `rm` 对 `rm -rf /` 有 `--preserve-root` 保护，但 `rm -rf /*` 这类变形仍可能造成灾难）。
> 2. 在脚本里删除前，先 `echo` 出要删的路径，确认后再执行。写成**先 dry-run 后执行**的两段式。
> 3. 变量未定义时 `rm -rf "$DIR"/*` 若 `$DIR` 为空会变成 `rm -rf /*`。所以脚本开头要写 `set -u`，并在删除前校验 `[ -n "$DIR" ]`。

### 4.4 `mkdir` / `rmdir` / `touch` / `file` / `stat`

```bash
mkdir newdir                    # 创建目录
mkdir -p a/b/c                  # 一次创建多层，父目录不存在也自动创建
mkdir -p project/{src,test}     # 花括号展开，一次建两个（bash 特性）
rmdir emptydir                  # 只能删空目录，非空会报错（比 rm -r 安全）
touch newfile                   # 创建空文件，或更新已有文件的时间戳
file somefile                   # 猜文件类型（看 magic number，不靠扩展名）
stat somefile                   # 详细元数据：inode、权限、三个时间戳
```

`stat` 输出的三个时间戳值得记住：

| 时间戳 | 含义 | 什么时候变 |
| --- | --- | --- |
| Access (`atime`) | 最后访问时间 | 读文件时。很多系统用 `relatime` 降低更新频率 |
| Modify (`mtime`) | 最后**内容**修改时间 | 写文件内容时。`ls -l` 显示的就是它 |
| Change (`ctime`) | 最后**元数据**修改时间 | 改权限、改属主、改内容都会变（注意 ctime 无法被手动设置） |

"删不掉的文件"有时是因为目录本身权限不够，而不是文件权限——`rm` 改的是**目录**的目录项，所以需要**目录的写权限**，而不是文件的写权限。这是一个非常经典的分析点（详见 [04-permissions-and-users.md](04-permissions-and-users.md)）。

### 4.5 `find` —— 按条件查找

`find` 的通用形式是 `find 起始路径 条件 动作`。

```bash
# 1. 按名字（不区分大小写用 -iname）
find /var/log -name "*.log"

# 2. 按类型：f 普通文件, d 目录, l 符号链接
find . -type f
find /etc -type d -maxdepth 1        # 只看 /etc 这一层

# 3. 按大小：+100M 大于 100M，-1k 小于 1k
find / -type f -size +100M 2>/dev/null

# 4. 按修改时间：-mtime -7 表示 7 天内改过，+30 表示 30 天前改过
find /var/log -type f -mtime +30
find . -type f -mmin -10             # 10 分钟内修改过的（mmin 按分钟）

# 5. 按权限：找 SUID 文件（安全审计的标准动作）
find / -perm -4000 -type f 2>/dev/null

# 6. 找到后执行动作：-exec ... {} \;
find . -type f -name "*.tmp" -exec rm -v {} \;
# 更高效：用 + 代替 \; 让多个文件一次传给命令
find . -type f -name "*.log" -exec gzip -v {} +

# 7. 组合条件（-o 是 or，默认是 and）
find . -type f \( -name "*.c" -o -name "*.h" \) -exec wc -l {} +
```

**`find` vs `locate`**：`locate` 查的是一个预先建好的数据库（由 `updatedb` 维护），因此**极快**，但结果可能过期，且新建的文件可能查不到。需要实时准确的结果就用 `find`；只是"这个文件大概在哪"就用 `locate`。

**`2>/dev/null` 的作用**：`find /` 会因为权限不足产生大量报错到 stderr，`2>/dev/null` 把它们丢掉，只保留正常结果。这是 `find` 最常用的搭配。

### 4.6 `ln` —— 硬链接与符号链接

这是 Linux 文件系统里最容易混淆的概念之一，理解它需要先接受**inode** 这个概念：文件的实际内容 + 元数据存在 inode 里，**目录里存的是"文件名 → inode 号"的映射**。

| 对比项 | 硬链接 (hard link) | 符号链接 (symbolic link) |
| --- | --- | --- |
| 创建命令 | `ln target linkname` | `ln -s target linkname` |
| 本质 | 同一 inode 的**另一个目录项**（又一个名字） | 一个独立文件，内容是**目标路径的字符串** |
| inode | 与目标**相同** | 与目标**不同**，自己占一个 inode |
| 跨文件系统 | ❌ 不行（inode 号只在单个文件系统内唯一） | ✅ 可以 |
| 指向目录 | ❌ 一般不行（防止目录树成环） | ✅ 可以 |
| 目标被删除后 | 仍然可用（还有名字指向 inode，数据不丢） | **变成悬空链接 (dangling)**，访问会 `No such file` |
| 占用空间 | 不额外占用（只多一个目录项） | 占一个 inode 和路径字符串的长度 |
| 相对路径解析 | 不涉及 | **相对于链接所在目录**，不是相对于你的当前目录 |
| `ls -l` 显示 | 普通文件，链接计数 +1 | `l` 类型，显示 `link -> target` |

```bash
# 硬链接示例：注意 inode 号相同（1600），链接计数变成 2
$ echo hello > original.txt
$ ln original.txt hard.txt
$ ls -li original.txt hard.txt
1600 -rw-r--r-- 2 alice devs 6 Mar 12 10:30 hard.txt
1600 -rw-r--r-- 2 alice devs 6 Mar 12 10:30 original.txt

# 删掉 original.txt，数据仍然通过 hard.txt 可访问
$ rm original.txt
$ cat hard.txt
hello

# 符号链接示例：注意 inode 不同，类型是 l
$ ln -s /var/log/syslog mylog
$ ls -l mylog
lrwxrwxrwx 1 alice devs 14 Mar 12 10:31 mylog -> /var/log/syslog
```

**链接计数 (link count)** 的含义就此明确：它数的是"有多少个目录项指向这个 inode"。硬链接创建一个就 +1，`rm` 一个就 -1；**只有当计数归零，inode 和数据块才真正被回收**。所以"文件被删了但空间没释放"的常见原因就是：还有一个进程开着这个文件（进程持有一个引用），`lsof | grep deleted` 能看到这种情况。

**符号链接的相对路径陷阱**：

```bash
# 假设 /opt/app/run -> ../lib/run.sh
# 这个 ../lib 是相对 /opt/app 解析的，即 /opt/lib/run.sh
# 而不是相对于你当前所在的目录
```

## 5. 权限 rwx 与 chmod / chown

### 5.1 rwx 对文件与目录的不同含义

这是最常被搞错的地方：**同样的字母，对目录的意义完全不同**。

| 权限位 | 对文件 | 对目录 |
| --- | --- | --- |
| `r` (read) | 可以读取文件内容 | 可以**列出目录内的名字**（`ls`）。注意：知道名字但没有 `x` 的话，只能看名字不能访问内容 |
| `w` (write) | 可以修改文件内容 | 可以**在目录内创建、删除、重命名**条目（改的是目录项，不是文件本身） |
| `x` (execute) | 可以作为程序执行 | 可以**进入 (cd) 和穿越 (traverse)** 该目录，即路径解析可以通过它 |

由此可以推出几个实用结论：

- **"我可以写这个文件但删不掉它"**：删除需要的是**父目录的 `w`**，与文件自身权限无关。
- **"我能读文件却 `cat` 不了"**：路径上某一级目录缺 `x`，无法穿越。
- **目录的 `r` 而不给 `x`**：能 `ls` 看到文件名，但拿不到详细信息、也打不开。这是一个刻意的"只看名字"配置。

### 5.2 数字（八进制）表示法

每一位权限用一个八进制数字表示，由三个位相加：

```text
r = 4   (读)
w = 2   (写)
x = 1   (执行)
- = 0   (无)
```

三组数字依次是 **属主 (user) / 属组 (group) / 其他 (other)**。完整对照表：

| 八进制 | 二进制 | 符号 | 含义 |
| --- | --- | --- | --- |
| 0 | 000 | `---` | 无权限 |
| 1 | 001 | `--x` | 仅执行 |
| 2 | 010 | `-w-` | 仅写 |
| 3 | 011 | `-wx` | 写 + 执行 |
| 4 | 100 | `r--` | 仅读 |
| 5 | 101 | `r-x` | 读 + 执行 |
| 6 | 110 | `rw-` | 读 + 写 |
| 7 | 111 | `rwx` | 读 + 写 + 执行 |

三种典型用法（日常遇到的绝大多数场景都属于其中之一）：

| 数字 | 符号 | 典型用途 |
| --- | --- | --- |
| `755` | `rwxr-xr-x` | 目录、可执行程序、脚本；属主全权，其他人只读/执行 |
| `644` | `rw-r--r--` | 普通文件、网页、配置文件；属主可写，其他人只读 |
| `600` | `rw-------` | 私密文件、密钥、含凭据的配置；只有属主能碰 |

其他常用值：`700`（私有目录）、`640`（属组可读的配置）、`664`（组内可协作编辑的文件）、`775`（组内可协作的目录）、`777`（所有人全权，**除调试外不要用**）、`400`（只读，例如只读密钥）、`444`（所有人只读）。

```bash
chmod 755 deploy.sh        # -rwxr-xr-x
chmod 644 config.yml       # -rw-r--r--
chmod 600 ~/.ssh/id_ed25519   # SSH 私钥必须是这个权限，否则 ssh 会拒绝使用
chmod 700 ~/.ssh           # 私钥所在目录
chmod -R 755 public_html/  # 递归应用到整个目录树
```

> 关于 SSH 私钥权限：`ssh` 会检查私钥文件是否对其他用户可读，如果权限过于开放（如 `644`）它会**拒绝使用并直接报错**。这是"权限太松反而用不了"的一个真实例子，见 [07-vm-and-env-setup.md](07-vm-and-env-setup.md) 的密钥登录部分。

### 5.3 符号（字母）表示法

当只想改一两位、不想重算整个数字时，用符号表示法更方便。

格式：`chmod [ugoa][+-=][rwx] 文件`

| 部分 | 可选值 | 含义 |
| --- | --- | --- |
| 谁 | `u` `g` `o` `a` | user / group / other / all |
| 操作 | `+` `-` `=` | 增加 / 移除 / **精确设为** |
| 权限 | `r` `w` `x` | 以及 `s` `t`（特殊位，见 04 篇） |

```bash
chmod u+x script.sh          # 给属主加执行权（最常用的一条）
chmod g-w file.txt           # 移除属组的写权
chmod o=r file.txt           # 把 other 精确设为只读（= 是赋值，不是增加）
chmod a+r file.txt           # 所有人可读
chmod u+x,g+x,o+x script.sh  # 逗号分隔，一次多组
chmod -R g+w shared/         # 递归给属组加写权
chmod u=rwx,g=rx,o= file     # 等价于 chmod 750
```

**`=` 与 `+` 的关键区别**：`chmod o+w` 是"在现有基础上加写"，`chmod o=w` 是"把 other 设成只有写"（会**清掉**原有的读和执行）。不确定时用 `=` 更明确。

### 5.4 `chown` 与 `chgrp`

`chmod` 管权限，`chown` 管**归属**（属主与属组）。

```bash
chown alice file.txt               # 只改属主
chown alice:devs file.txt          # 同时改属主和属组
chown :devs file.txt               # 只改属组（等价于 chgrp devs file.txt）
chown -R alice:devs /srv/app       # 递归（常用于修 git clone 后的属主问题）
chown --reference=ref.txt target   # 把 target 的归属改成与 ref 一致
chgrp devs file.txt                # 单独改属组
```

`ls -l` 里属主和属组那两列就是这样来的。几个现实场景：

- **只有 root 能把文件给别的用户**。普通用户不能把自己的文件"送"给别人（否则可以伪造成别人的文件），但可以把属组改成自己所属的组。
- **USB 盘里的文件属主变成数字而不是用户名**：说明该 UID 在本机不存在，`ls -l` 会直接显示数字。
- **`sudo` 执行过的命令产生的文件归 root**：这是 `sudo` 最常见的副作用，需要 `chown -R $USER` 修回来（或改用 `sudo -u` 指定身份）。

### 5.5 一个完整的排查例子：为什么我的脚本不能执行

```bash
$ ./deploy.sh
bash: ./deploy.sh: Permission denied

# 第一步：看权限
$ ls -l deploy.sh
-rw-r--r-- 1 alice devs 412 Mar 12 10:30 deploy.sh
#  注意：没有 x

# 第二步：确认这是"缺执行位"而不是别的问题
$ head -1 deploy.sh
#!/bin/bash         # shebang 正常

# 第三步：加执行位
$ chmod u+x deploy.sh
$ ls -l deploy.sh
-rwxr--r-- 1 alice devs 412 Mar 12 10:30 deploy.sh
$ ./deploy.sh
# 正常执行
```

如果在第三步后仍然 `Permission denied`，就要继续查**路径上每一级目录是否有 `x`**（例如脚本放在一个 `700` 的目录里，或者所在分区被 `noexec` 挂载）。`mount | grep noexec` 可以检查挂载选项。这个排查顺序（先看文件权限 → 再看路径权限 → 再看挂载选项）是通用的。

## 6. `du` 与 `df`

```bash
df -h                      # 各文件系统用了多少（disk free）
df -h /home                # 只看 /home 所在的分区
du -sh /var/log            # 某目录总大小（s = summary, h = human readable）
du -h --max-depth=1 /var   # 只看 /var 下第一层各目录的大小
du -ah /etc | sort -rh | head -20   # 找出 /etc 下最大的 20 个东西
```

**"`df` 说磁盘满了，但 `du` 加起来对不上"** 的标准解释：有进程还开着已被删除的文件，空间没释放。排查方法：

```bash
lsof +L1 2>/dev/null | head        # 列出 link count 为 0 的打开文件
# 或者
lsof 2>/dev/null | grep -i deleted
```

重启该进程（或 `: > /proc/<pid>/fd/<n>` 截断）即可释放。

## 7. `tar` —— 归档与压缩

```bash
tar -czvf archive.tar.gz dir/       # 创建 gzip 压缩归档 (c=create, z=gzip, v=verbose, f=file)
tar -xzvf archive.tar.gz            # 解压到当前目录 (x=extract)
tar -xzvf archive.tar.gz -C /tmp/   # 解压到指定目录
tar -tzvf archive.tar.gz            # 只列出内容不解压 (t=list)
tar -cjvf archive.tar.bz2 dir/      # 用 bzip2（压缩率更高、更慢）
tar -cJvf archive.tar.xz  dir/      # 用 xz（压缩率最高、最慢）
tar -xzvf archive.tar.gz path/inside/archive    # 只解压其中一部分
```

记忆方法：**`c`/`x`/`t` 三选一（创建/解压/查看）**，`f` 一定放最后并紧跟文件名，中间的 `z`/`j`/`J` 是压缩方式。`-C` 必须在文件名之后。

`tar` 的现代替代是 `zstd`（`tar --zstd`，压缩速度与压缩率的平衡更好），但 `gzip` 的兼容性最好，是最通用的选择。

## 8. `head` / `tail` / `wc`（速览）

这三个在下一篇会详细讲，这里只记最常用的形态：

```bash
head -n 20 file.txt        # 前 20 行
tail -n 20 file.txt        # 后 20 行
tail -f /var/log/syslog    # 持续跟踪新写入的行（-f = follow），Ctrl+C 退出
tail -F /var/log/app.log   # 比 -f 更强：文件被轮转（重建）后仍能继续跟踪
wc -l file.txt             # 行数（实际数的是换行符个数）
wc -l < file.txt           # 只输出数字，不带文件名（脚本里常用）
```

## 小结

- Linux 是单根树结构，磁盘通过**挂载**接入；`/etc` 放配置、`/var` 放变化的数据、`/proc` 与 `/sys` 是内核的虚拟视图、`/tmp` 是所有人可写的临时区。现代发行版的 `/bin`、`/sbin`、`/lib` 一般是指向 `/usr` 的符号链接 (usrmerge)。
- `ls -l` 的第一个字符是文件类型，后面 9 位是三组 rwx；对**目录**来说 `x` 是"可穿越"，`w` 是"可增删目录项"。
- 删除文件需要的是**父目录的写权限**，不是文件自身的写权限；`rm` 没有回收站。
- `find` 是实时精确查找，`locate` 查数据库所以快但可能过期；`find /` 记得加 `2>/dev/null`。
- **硬链接**共享 inode（链接计数 +1，跨文件系统不行，不能指向目录）；**符号链接**是存路径字符串的独立文件（可跨文件系统、可指向目录、目标删除后会悬空）。
- 权限数字表示法：`r=4, w=2, x=1`，三组对应属主/属组/其他。常用 `755`（程序与目录）、`644`（普通文件）、`600`（私密文件与私钥）。
- `chmod` 改权限，`chown`/`chgrp` 改归属；符号表示法适合只改一两位，`=` 是赋值而 `+` 是累加。
- "脚本不能执行"的标准排查顺序：文件是否有 `x` → 路径上每级目录是否有 `x` → 所在分区是否 `noexec`。
- `df` 与 `du` 对不上，通常是"已删除但仍被进程打开的文件"占着空间，用 `lsof +L1` 查。
