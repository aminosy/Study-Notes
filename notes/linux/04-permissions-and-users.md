# 用户、组、sudo 与特殊权限位

本篇深入 Linux 的用户与组模型：/etc/passwd 与 shadow 的分工、用户组管理命令、sudo 与 sudoers 的写法、umask 的计算，以及 SUID/SGID/sticky 三个特殊权限位的真实用途。

## 1. 用户与组的数据文件

Linux 的用户信息不放在某个数据库里，而是几个纯文本文件。理解它们的字段布局，就能读懂一切权限问题。

### 1.1 `/etc/passwd` —— 七个字段

```bash
$ head -3 /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
alice:x:1000:1000:Alice Wang,,,:/home/alice:/bin/bash
```

| 序号 | 字段 | 示例 | 含义 |
| --- | --- | --- | --- |
| 1 | 用户名 | `alice` | 登录名 |
| 2 | 密码占位符 | `x` | **`x` 表示口令哈希在 `/etc/shadow` 里** |
| 3 | UID | `1000` | 用户 ID，内核真正用的是它 |
| 4 | GID | `1000` | **主组 (primary group)** 的 GID |
| 5 | GECOS / 注释 | `Alice Wang,,,` | 全名、办公室、电话等，逗号分隔 |
| 6 | 家目录 | `/home/alice` | 登录后的起始目录 |
| 7 | 登录 shell | `/bin/bash` | 登录时启动的程序 |

几个必须读懂的细节：

- **`x` 不是密码**。它是一个占位符，意思是"真正的口令哈希在 `/etc/shadow` 里，而那个文件普通用户读不到"。这个设计叫 **shadow password**：早期的 Unix 把哈希直接放在 `/etc/passwd` 的第 2 字段，但 `/etc/passwd` 必须对所有人可读（很多程序要查用户名和 UID），于是哈希暴露在可读文件里，任何人都能拿去离线暴力破解。把哈希移到权限为 `640`（或 `000`）的 `/etc/shadow` 就解决了这个问题。
- **`/etc/passwd` 里第 2 字段为空**表示该用户**无密码即可登录**（很少见，通常是要修的问题）；写 `*` 或 `!` 表示该账户被锁定，无法用密码登录。
- **第 7 字段是 `/usr/sbin/nologin` 或 `/bin/false`**：这是系统账户的常见做法——账户存在（用于文件属主、服务运行身份）但不允许交互式登录。`daemon`、`www-data`、`sshd` 这类账户都是如此。
- **登录 shell 可以是任意程序**。把它设成 `/bin/bash` 之外的东西（如某个菜单程序）就形成了"受限登录"。

### 1.2 `/etc/shadow` —— 只有 root 能读

```bash
$ sudo head -2 /etc/shadow
root:$6$xxxx...:19700:0:99999:7:::
alice:$y$j9T$yyy...:19700:0:99999:7:::
```

| 序号 | 字段 | 含义 |
| --- | --- | --- |
| 1 | 用户名 | 与 `/etc/passwd` 对应 |
| 2 | 口令哈希 | 形如 `$id$salt$hash`。`$6$`=SHA-512 crypt，`$5$`=SHA-256 crypt，`$y$`=yescrypt，`$2b$`=bcrypt（不同发行版/工具的默认算法不同） |
| 3 | 最后修改日期 | 从 1970-01-01 起的天数 |
| 4 | 最小天数 | 两次改密码之间至少要隔多少天（0 = 随时可改） |
| 5 | 最大天数 | 密码多少天后必须改（99999 ≈ 永不过期） |
| 6 | 警告天数 | 到期前多少天开始提醒 |
| 7 | 宽限天数 | 到期后还允许登录多少天 |
| 8 | 账户失效日期 | 从 1970-01-01 起的天数，超过则账户不可用 |
| 9 | 保留字段 | 通常为空 |

**哈希前缀的正确读法不是"安全性排名"**：`$6$` 并不等于"比 `$y$` 更安全"，它们都是**加盐的单向密钥派生函数**，目的是让离线暴力破解变慢。现代发行版逐渐向 yescrypt（`$y$`）等更强的算法迁移，但具体默认值由发行版决定，不要凭前缀下结论。

> ⚠️ 不要手工编辑 `/etc/shadow`。改密码用 `passwd`，设过期策略用 `chage`。手工编辑极易破坏格式，可能导致**所有用户都无法登录**。

### 1.3 `/etc/group` —— 四个字段

```bash
$ head -3 /etc/group
root:x:0:
sudo:x:27:alice
devs:x:1001:alice,bob
```

| 序号 | 字段 | 含义 |
| --- | --- | --- |
| 1 | 组名 | `devs` |
| 2 | 组密码占位符 | `x`（组密码几乎不用；真正在 `/etc/gshadow`） |
| 3 | GID | `1001` |
| 4 | 成员列表 | 逗号分隔的**附加成员** |

**一个极其重要的细节**：`/etc/group` 的第 4 字段只列出**附加成员 (supplementary members)**，不列出"以该组为主组的用户"。例如 `alice` 的主组是 `alice`（GID 1000），那么 `alice` 这个组在 `/etc/group` 里的成员列表通常是**空的**，尽管 alice 属于它。要看到完整的组成员关系，用 `id` 或 `getent group`：

```bash
$ id alice
uid=1000(alice) gid=1000(alice) groups=1000(alice),27(sudo),1001(devs)
#                                    └─── 主组      └─── 附加组 ───┘

$ getent group devs       # 从 NSS（可能包含 LDAP 等非本地源）查询
$ groups alice            # 简化的组列表
$ groups                  # 当前用户自己的
```

`getent` 比直接 `cat /etc/group` 更正确，因为系统可能配置了 NSS（Name Service Switch），用户信息来自 LDAP/SSSD 等外部源，而这些都不会出现在本地文件里。

### 1.4 UID 的约定

| 范围 | 用途 | 说明 |
| --- | --- | --- |
| 0 | root | **UID 0 就是超级用户**。用户名是不是叫 root 无关紧要——这是"改名也改不掉权限"的原因 |
| 1 – 999 | 系统/服务账户 | 具体边界是**发行版策略**，不是内核规定。Debian/Ubuntu 传统上用 < 1000，RHEL 系传统上用 < 500 或 < 1000 |
| 1000 – 59999 | 普通用户 | 大多数发行版从 1000 开始分配（`/etc/login.defs` 里的 `UID_MIN`） |
| 65534 | `nobody` | 通常用于"最小权限"运行，映射无主 UID |

这里必须诚实：**具体数值范围是发行版与 `login.defs` 的配置，不是内核强制的**。唯一的硬规则是"UID 0 是特权用户"。

## 2. 主组与附加组

| 概念 | 含义 | 数量 |
| --- | --- | --- |
| 主组 (primary group) | 用户创建文件时**默认的属组**；记录在 `/etc/passwd` 第 4 字段 | 恰好 1 个 |
| 附加组 (supplementary groups) | 额外获得的组权限；记录在 `/etc/group` 的成员列表 | 0 到多个 |

**附加组的实际价值**：Linux 权限模型只有三档（属主/属组/其他），如果每个权限都要一个新组，就需要"一个用户属于多个组"的能力。附加组正是为此存在。典型用法：

- 把用户加入 `sudo` 组 → 获得 `sudo` 权限（Debian/Ubuntu）；
- 加入 `docker` 组 → 可以操作 Docker 守护进程（⚠️ 这等于给了 root 级别的权限，因为能挂载宿主根目录）；
- 加入 `devs` 组 → 能读写属组为 `devs` 的共享目录。

```bash
id                                    # 看自己的 UID/GID/所有组
id alice                              # 看别人
groups alice
getent group sudo                     # 看某个组有哪些成员
```

## 3. 用户与组的管理命令

### 3.1 创建用户

```bash
# 底层命令（所有发行版都有）
sudo useradd -m -s /bin/bash -G sudo,devs alice
sudo passwd alice                     # 设置密码（一定要单独做，useradd 不设密码）

# 参数说明
#   -m            创建家目录（不加 -m 就不会创建！这是最常见的疏漏）
#   -s /bin/bash  指定登录 shell
#   -G sudo,devs  加入附加组（逗号分隔，无空格）
#   -d /path      自定义家目录位置
#   -u 1500       手动指定 UID
#   -c "注释"     设置 GECOS 字段
#   -e 2025-12-31 设置账户失效日期
```

**`useradd` vs `adduser`**：在 Debian/Ubuntu 上，`adduser` 是一个更友好的 Perl 脚本封装，它会交互式地问密码、自动创建家目录、自动复制 `/etc/skel` 里的初始配置文件（`.bashrc` 等）：

```bash
sudo adduser alice                    # Debian/Ubuntu：交互式，推荐新手
```

在 RHEL 系上 `adduser` 只是 `useradd` 的符号链接，没有交互能力。**不要假设 `adduser` 在任何系统上行为一致。**

另外注意：`/etc/skel/` 的内容会被复制到新用户的家目录。如果希望新用户一登录就有某些配置（例如公司统一的 `.bashrc`、`.vimrc`），就放到 `/etc/skel/`。而**已经存在的用户不会自动获得这些文件**——这是"为什么我的配置和同事不一样"的常见原因。

### 3.2 修改用户

```bash
# 加入附加组 —— 必须用 -aG，这是最容易犯的错误之一
sudo usermod -aG docker alice

# ❌ 错误写法：-G 会【替换】整个附加组列表，而不是追加！
sudo usermod -G docker alice
# 后果：alice 原来的 sudo、devs 等附加组全部丢失 → 可能连 sudo 都不能用了
```

其他常用操作：

```bash
sudo usermod -s /bin/zsh alice        # 改登录 shell
sudo usermod -d /new/home -m alice    # 改家目录并移动内容
sudo usermod -L alice                 # 锁定账户（在 shadow 的哈希前加 !）
sudo usermod -U alice                 # 解锁
sudo usermod -l newname alice         # 改用户名（家目录不会自动跟着改）
sudo usermod -e 2025-12-31 alice      # 设置失效日期
```

**`-aG` 与 `-G` 的区别是本节最重要的一条**。判断是否加成功：

```bash
id alice            # 确认附加组列表里出现了目标组
```

改完组之后，**用户需要重新登录**（退出所有会话，或 `newgrp`）才能生效。因为进程的组列表是在登录时确定的，已运行的 shell 不会自动更新。这是"我明明加了组但还是 Permission denied"的经典原因。

### 3.3 删除用户与组

```bash
sudo userdel alice            # 删除用户，保留家目录和邮件
sudo userdel -r alice         # 同时删除家目录与邮件池（-r = remove）

sudo groupadd newgroup
sudo groupdel newgroup        # 如果还有用户以它为主组，会失败
sudo gpasswd -a alice devs    # 添加组成员（等价于 usermod -aG）
sudo gpasswd -d alice devs    # 移除组成员
sudo gpasswd -A alice devs    # 指定组管理员
```

**`userdel` 不删除该用户遗留的文件**。系统里会散落属主为"数字 UID"的文件。要找出它们：

```bash
sudo find / -uid 1000 -o -gid 1000 2>/dev/null | head -50
```

### 3.4 密码与过期策略

```bash
passwd                        # 改自己的密码
sudo passwd alice             # root 改别人的密码（不需要旧密码）
sudo passwd -l alice          # 锁定（lock）
sudo passwd -u alice          # 解锁（unlock）
sudo passwd -e alice          # 强制下次登录必须改密码（expire）
sudo passwd -S alice          # 查看密码状态（L=locked, P=set, NP=no password）

sudo chage -l alice           # 查看过期策略
sudo chage -M 90 alice        # 最长 90 天必须改
sudo chage -m 1 alice         # 最短 1 天（防止立刻又改回去）
sudo chage -W 7 alice         # 提前 7 天警告
sudo chage -E 2025-12-31 alice # 账户到期日
sudo chage -d 0 alice         # 强制下次登录改密码
```

`chage -l` 的输出对应 `/etc/shadow` 的第 3-8 字段，两者互相印证，是理解 shadow 字段含义最直观的方式。

## 4. `su` 与 `sudo`

### 4.1 核心区别

| 维度 | `su` (substitute user) | `sudo` (superuser do) |
| --- | --- | --- |
| 需要谁的口令 | **目标用户**的口令（切到 root 就要 root 密码） | **自己的**口令 |
| 是否共享密码 | 是的——要知道 root 密码，一旦泄露影响极大 | 不需要知道 root 密码，可随时撤销 |
| 授权粒度 | 全有或全无（要么能变成 root，要么不能） | **可细到单个命令与参数** |
| 审计 | 系统只知道"有人以 root 登录了" | 每次执行都记录**是谁、在哪个终端、执行了什么** |
| 默认环境 | 继承原用户的部分环境变量 | 有 `env_reset` 等策略控制 |
| 撤销方式 | 必须改 root 密码（影响所有人） | 从 sudoers 移除该用户即可 |
| 配置 | `/etc/pam.d/su`、`/etc/login.defs` | `/etc/sudoers` 与 `/etc/sudoers.d/` |

**结论**：多用户环境应该用 `sudo`，并 **`su` 到 root 通常被限制**（现代发行版的 root 账户默认往往没有可用口令，或 `su` 被 `pam_wheel` 限制为只有 `wheel`/`sudo` 组成员能用）。`su` 仍然有用——用来切换到**非 root** 的另一个用户来做测试：

```bash
su - alice                    # 切成 alice，并要求 alice 的口令
```

### 4.2 `su` 的三种写法与 `sudo` 的三种写法

```bash
su                            # 切成 root，但【保留】当前环境变量与当前目录
su -                          # 切成 root，并【模拟完整登录】：重置环境、切换到 root 家目录、执行 profile
su - alice                    # 切成 alice 并模拟完整登录

sudo cmd                      # 以 root 执行单条命令，环境大致保留（受 sudoers env_reset 影响）
sudo -i                       # 启动一个【登录 shell】(login shell)，加载 root 的 profile，环境更"干净"
sudo -s                       # 启动一个【非登录 shell】，保留当前环境与目录，但 euid 是 root
sudo su -                     # 先 sudo 再 su -（等同 sudo -i，但多了两层，没必要）
sudo -u alice cmd             # 以 alice 身份执行命令（不需要 root）
sudo -g devs cmd              # 以指定组身份执行
```

**`su` 与 `su -` 的区别非常重要**：不带 `-` 时，你的 `PATH`、`HOME`、当前目录都保持原样，可能导致奇怪的问题（例如 `HOME` 还是 `/home/alice`，于是某些程序去写 alice 的配置；`PATH` 里可能没有 `/sbin`，找不到 `ifconfig`）。**需要"完整的另一个用户环境"时永远加 `-`**。`sudo -i` 与 `sudo -s` 的区别同理。

### 4.3 `/etc/sudoers` 与 `visudo`

```bash
sudo visudo                   # 编辑 /etc/sudoers 的【唯一正确方式】
sudo visudo -f /etc/sudoers.d/myapp   # 编辑一个 drop-in 文件
```

**为什么必须用 `visudo`**：
1. 它在保存时**做语法检查**，语法错误会拒绝保存（手工编辑出错可能导致**任何人都无法使用 sudo**，甚至需要单用户模式救援）；
2. 它使用**文件锁**，避免两个管理员同时编辑造成互相覆盖；
3. 它是原子写入（写临时文件再重命名），避免编辑中途被中断导致文件损坏。

### 4.4 sudoers 的字段含义

一条规则的一般形式：

```text
谁  在哪些主机 = (可以变成哪些用户:哪些组) [是否需要密码:] 可以执行哪些命令
```

真实示例：

```text
# 1. 允许 sudo 组成员以任何用户、任何组身份执行任何命令（并需要自己的口令）
%sudo   ALL=(ALL:ALL) ALL

# 2. 同上，但组名以 % 开头表示"组"；RHEL 系上组名通常是 wheel
%wheel  ALL=(ALL)       ALL

# 3. 允许 alice 免密码执行特定命令（注意必须写绝对路径）
alice   ALL=(root) NOPASSWD: /usr/bin/systemctl restart nginx

# 4. 允许 ops 组免密码重启、查看状态，但不能做其他操作（命令用逗号分隔）
%ops    ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp, /usr/bin/systemctl status myapp

# 5. 允许执行某目录下的任何命令
%devs   ALL=(root) /usr/bin/docker

# 6. 默认设置（Defaults 行）
Defaults    env_reset
Defaults    timestamp_timeout=15        # 15 分钟后需要重新输密码
Defaults    passwd_tries=3
Defaults    logfile="/var/log/sudo.log"
```

**逐个字段拆解第 3 条**：

| 字段 | 值 | 含义 |
| --- | --- | --- |
| 用户规格 | `alice` | 谁（`%`前缀是组，`User_Alias` 可定义别名） |
| 主机规格 | `ALL` | 在哪些主机上有效（多机共享 sudoers 时有用） |
| 运行身份 | `(root)` | 可以 sudo 成哪个用户 |
| 标签 | `NOPASSWD:` | 是否需要输入口令 |
| 命令规格 | `/usr/bin/systemctl restart nginx` | 允许执行的命令，**必须用绝对路径** |

**关于命令规格的安全教训**：命令的**参数也参与匹配**。写 `/usr/bin/systemctl restart nginx` 只允许这精确的一条；写 `/usr/bin/systemctl` 则允许 `systemctl` 的**任何**子命令，而 `systemctl` 能做的事远超"重启一个服务"（例如 `systemctl edit` 可以改配置、`systemctl link` 可以加载任意 unit）——**这等于变相给了 root**。这是编写 sudoers 时最危险的错误。

同理，任何允许 `sudo` 执行编辑器的规则（`sudo vim`、`sudo less`、`sudo find -exec`）都可以从编辑器内部逃逸出一个 shell。**授权时一定要问"这条命令能否被用来执行任意代码"，如果答案是肯定的，那它等价于完全授权。**

### 4.5 推荐的实践：用 drop-in 文件

不要直接改 `/etc/sudoers` 本体（升级时容易冲突、难以追踪谁加过什么）。在 `/etc/sudoers.d/` 下按用途建文件：

```bash
sudo visudo -f /etc/sudoers.d/90-myapp
```

```text
# /etc/sudoers.d/90-myapp —— 允许 deploy 用户免密码重启本应用
deploy ALL=(root) NOPASSWD: /usr/bin/systemctl restart myapp
```

```bash
# 权限要求：sudo 会忽略权限过于开放的 drop-in 文件
sudo chmod 440 /etc/sudoers.d/90-myapp
sudo chown root:root /etc/sudoers.d/90-myapp
```

`/etc/sudoers` 本体里通常有一行 `#includedir /etc/sudoers.d`（注意 `#` 在这里**不是注释**，这是 sudoers 语法的一个历史怪癖），它会把该目录下的文件都包含进来。

> ⚠️ 文件名的约定：包含 `.` 或 `~` 的文件会被 sudo 跳过。所以不要用 `90-myapp.conf` 这种带点的名字。

### 4.6 常用 `sudo` 操作

```bash
sudo -l                       # 【最重要】列出当前用户被允许执行哪些命令
sudo -v                       # 刷新时间戳（延长免密时间）
sudo -k                       # 立即失效时间戳（下次必须重新输密码）
sudo -u postgres psql         # 以另一个用户执行
sudo -E cmd                   # 保留当前环境变量（默认会被 env_reset 清理）
sudo !!                       # 用 sudo 重跑上一条命令（经典救命命令）
sudo -n cmd                   # 非交互模式，需要密码就直接失败（脚本里用）
```

`sudo -l` 是做权限排查时的第一步：**不要猜自己有什么权限，直接问系统。**

## 5. `umask`

### 5.1 概念：默认权限是"减"出来的

新创建的文件和目录的权限不是凭空的，而是从一个**基线上把 umask 的位去掉**：

```text
文件：基线 666 (rw-rw-rw-)   因为新建文件默认不给执行权（安全的默认值）
目录：基线 777 (rwxrwxrwx)   因为目录需要 x 才能进入
最终权限 = 基线 & ~umask
```

**注意方向**：umask 是**掩码 (mask)**，表示"要**屏蔽**掉的位"，不是"要设置的位"。所以 **umask 越大，权限越少**。这与直觉相反，需要特别小心。

### 5.2 计算示例

**umask = 022**（Debian/Ubuntu 的常见默认值，root 也常用 022）：

```text
文件: 666 & ~022
  666 = rw- rw- rw-
~ 022 = rwx r-x r-x  → 取反后：111 101 101 → 要"减掉"的是 w 和 x 位
  结果 = rw- r-- r-- = 644

目录: 777 & ~022
  777 = rwx rwx rwx
  减掉 w（组和其他）→ rwx r-x r-x = 755

即：umask 022 → 新文件 644，新目录 755
```

**umask = 077**（更严格，适合处理敏感数据的账户）：

```text
文件: 666 & ~077 → 600  (rw-------)
目录: 777 & ~077 → 700  (rwx------)
即：只有属主能访问
```

**umask = 002**（适合**团队协作的共享目录**）：

```text
文件: 666 & ~002 → 664  (rw-rw-r--)
目录: 777 & ~002 → 775  (rwxrwxr-x)
即：同组的人也能读写
```

完整对照表：

| umask | 新文件权限 | 新目录权限 | 适用场景 |
| --- | --- | --- | --- |
| `022` | `644` `rw-r--r--` | `755` `rwxr-xr-x` | 常见默认，个人与服务器 |
| `002` | `664` `rw-rw-r--` | `775` `rwxrwxr-x` | 团队共享目录 |
| `027` | `640` `rw-r-----` | `750` `rwxr-x---` | 只有属组可读，其他完全看不到 |
| `077` | `600` `rw-------` | `700` `rwx------` | 私密数据、密钥 |
| `000` | `666` | `777` | ⚠️ 危险，不要用 |

**为什么文件基线是 666 而不是 777**：如果基线是 777，那么新建的文件默认就是可执行的，任何被下载的文本文件都可能被当成程序运行。**不自动给执行位**是一个刻意的安全默认。这也是为什么 `chmod +x` 总是要手工做——umask **不能**给文件加执行权，它只能减权限。

### 5.3 查看与设置

```bash
umask                  # 以八进制显示，例如 0022
umask -S               # 以符号显示，例如 u=rwx,g=rx,o=rx
umask 027              # 设置（当前 shell 内生效）
umask 0022             # 八进制写法（前导 0 是习惯写法）
```

**持久化（什么时候该放哪里）**：

| 位置 | 适用范围 | 说明 |
| --- | --- | --- |
| 当前 shell 里直接执行 `umask` | 该 shell 及其子进程 | 临时测试用 |
| `~/.bashrc`、`~/.profile` | 该用户的交互式会话 | 用户级设置 |
| `/etc/profile`、`/etc/bash.bashrc` | 登录 shell（所有用户） | 系统级但只覆盖 shell 启动的进程 |
| `/etc/login.defs` 的 `UMASK` | 登录时由 PAM 应用 | 影响面更广，包括非交互登录 |
| `pam_umask` 模块配置 | 通过 PAM 的各类会话 | 最"底层"的做法 |
| `systemd` unit 里的 `UMask=` | 该服务的进程 | 服务类进程推荐在这里设，而不是依赖全局 |

**关键提醒**：umask 是**进程属性**，会被子进程继承。因此：
- 在服务配置里写好 `UMask=` 比依赖系统的全局 umask 更可靠（否则服务产生的文件权限取决于它被谁、在什么环境下启动）；
- 一个用 `sudo` 启动的脚本可能继承 root 的 umask 而不是你的，这会让输出文件的权限与预期不同。

**`umask` 与已存在文件无关**：它只影响**创建时刻**。已经存在的文件不会因为改 umask 而改变权限。

### 5.4 一个真实的坑：`mkdir` 与 `-p`

```bash
$ umask 022
$ mkdir -p a/b/c
$ ls -ld a a/b a/b/c
drwxr-xr-x ... a
drwxr-xr-x ... a/b
drwxr-xr-x ... a/b/c
# 每一层都是按当前 umask 计算的，一致
```

但如果是**在共享目录里协作**，同事用 `umask 022` 创建了文件，即使目录设了 SGID（见下节），文件内的**属组会被继承，但权限位仍是 644**——组内其他人只能读不能写。这就是共享目录协作问题的完整图景：**SGID 解决"属组是谁"，umask 解决"权限位是多少"，两者必须一起配**。

## 6. 特殊权限位

除了 9 个普通权限位，还有 3 个特殊位，对应八进制数字的**第 4 位（最左边一位）**。

```text
    chmod 4755 file
          │└┬┘└┬┘└┬┘
          │ │  │  └── other: r-x (5)
          │ │  └───── group: r-x (5)
          │ └──────── user:  rwx (7)
          └────────── 特殊位：4 = SUID
```

| 位 | 八进制值 | 名称 | 作用于文件 | 作用于目录 |
| --- | --- | --- | --- | --- |
| SUID | `4` | Set User ID | 执行时进程的 **euid** 变成文件**属主**的 UID | 在 Linux 上**基本无意义**（历史上曾用于 BSD） |
| SGID | `2` | Set Group ID | 执行时进程的 **egid** 变成文件**属组**的 GID | **新建的文件/子目录继承该目录的属组**（非常有用） |
| sticky | `1` | Sticky bit | 在现代 Linux 上对文件无意义（历史上表示"常驻内存"） | **只有文件属主、目录属主或 root 能删除/重命名目录内的文件** |

### 6.1 `ls -l` 里怎么显示

关键规则：**小写字母表示"该位已设置且对应的执行位也设置了"；大写字母表示"该位已设置但对应的执行位没设置"**。

| 位置 | 普通 | 特殊位设置后 | 说明 |
| --- | --- | --- | --- |
| 属主的 `x` 位 | `-rwxr-xr-x` | `-rwsr-xr-x` | **小写 s** = SUID 且属主有 x |
| 属主的 `x` 位 | `-rw-r-xr-x` | `-rwSr-xr-x` | **大写 S** = SUID 但属主**没有** x |
| 属组的 `x` 位 | `-rwxr-xr-x` | `-rwxr-sr-x` | 小写 s = SGID 且属组有 x |
| 属组的 `x` 位 | `-rwxr--r-x` | `-rwxr-Sr-x` | 大写 S = SGID 但属组没有 x |
| 其他的 `x` 位 | `-rwxr-xr-x` | `-rwxr-xr-t` | 小写 t = sticky 且其他有 x |
| 其他的 `x` 位 | `-rwxr-xr--` | `-rwxr-xr-T` | 大写 T = sticky 但其他没有 x |

**大写 vs 小写的区分很有用**：一个 `-rwSr--r--` 告诉你 SUID 位设置了，但这个文件**本身不可执行**，所以 SUID 实际上不起作用。这是排查"这个 SUID 位到底有没有用"时的第一眼判断。

### 6.2 SUID（Set User ID）

**机制**：可执行文件被设置 SUID 后，任何用户执行它时，进程的**有效用户 ID (euid) 变成文件属主的 UID**，而**真实用户 ID (ruid) 仍然是执行者的**。因此程序可以"以文件属主的权限做事，但知道真正调用它的是谁"。

**经典真实例子：`passwd`**。

```bash
$ ls -l /usr/bin/passwd
-rwsr-xr-x 1 root root 68208 ... /usr/bin/passwd
```

普通用户改密码需要写 `/etc/shadow`，而这个文件是 `root:root` 且权限 `640`——普通用户没有写权限。那为什么能改？因为 `/usr/bin/passwd` 带着 SUID root 位，它运行时 euid 是 root，于是能写 shadow。同时它能通过 **ruid** 知道"真正调用我的是 alice"，因此只允许 alice 改自己的密码。**这个设计同时解决了"需要特权"和"不能滥用特权"两个问题**，是 SUID 最标准的使用场景。

其他典型的 SUID 程序：`/usr/bin/su`、`/usr/bin/mount`、`/usr/bin/umount`、`/bin/ping`（某些发行版上通过 capabilities 而非 SUID 实现，这是另一种更细粒度的机制）。

**SUID 在脚本上会被忽略**：

```bash
$ chmod 4755 myscript.sh
$ ls -l myscript.sh
-rwsr-xr-x 1 root root ... myscript.sh
$ ./myscript.sh
# 以你的身份运行，SUID 完全没有生效
```

**Linux 内核会忽略脚本文件上的 SUID 位**。原因是安全问题：shebang 里的解释器路径可以被替换，脚本内容也是可读可改的，如果 SUID 生效就会形成极其容易利用的提权路径。所以"设了 SUID 但没有提权"通常是这个原因。**要 SUID 生效，必须是编译出来的二进制。**

如果确实需要让脚本以特定身份运行，正确做法是：
- 用 `sudo` 显式授权执行该脚本（`/etc/sudoers.d/`）；
- 或者用 `systemd` 服务；
- 或者写一个极小的 C 包装程序（但仍然要小心设计）。

**SUID 的安全风险**：每一个 SUID root 二进制都是**攻击面的一部分**。它的任何内存漏洞（栈溢出、UAF、路径遍历、命令注入）都可能直接导致本地提权。因此：

```bash
# 审计系统上所有 SUID 文件 —— 这是标准的安全加固步骤
find / -perm -4000 -type f 2>/dev/null

# 同时也查一下 SGID 文件
find / -perm -2000 -type f 2>/dev/null

# 用 ls 看它们的属性，确认都是预期内的
find / -perm -4000 -type f -exec ls -l {} \; 2>/dev/null
```

自己打包或安装软件时，**不要随手动给文件加 SUID**。如果某个操作确实需要特权，优先考虑 capabilities（`setcap`）、systemd 的 `User=`/`AmbientCapabilities=`，或者把特权操作拆成一个极小的、接口严格的守护进程。

### 6.3 SGID（Set Group ID）

**对文件**：执行时进程的 egid 变成文件的属组。这与 SUID 对称，但用得少得多。

**对目录（这才是 SGID 最实用的场景）**：在该目录里**新建的文件和子目录，属组自动继承目录的属组**（而不是创建者的主组）。

**为什么这很重要**：设想一个共享目录 `/srv/project`，属组是 `devs`：

```bash
# 没有 SGID 时
$ mkdir /srv/project
$ chgrp devs /srv/project
$ chmod 775 /srv/project

$ cd /srv/project && touch alice.txt
$ ls -l alice.txt
-rw-rw-r-- 1 alice alice 0 ... alice.txt
#                    ^^^^^ 属组是 alice 的主组，不是 devs！
# 于是 bob（devs 成员）如果对文件没有 other 权限就访问不了
```

```bash
# 加上 SGID 后
$ chmod 2775 /srv/project        # 或 chmod g+s /srv/project
$ ls -ld /srv/project
drwxrwsr-x 2 root devs ... /srv/project

$ cd /srv/project && touch bob.txt
$ ls -l bob.txt
-rw-rw-r-- 1 bob devs 0 ... bob.txt
#                  ^^^^ 属组正确继承了 devs ✅
```

**完整的共享目录配方**（这是实践中最常用的一套）：

```bash
sudo groupadd devs
sudo usermod -aG devs alice
sudo usermod -aG devs bob

sudo mkdir -p /srv/project
sudo chgrp devs /srv/project
sudo chmod 2775 /srv/project      # 2 = SGID, 775 = 组内可读写执行

# 让目录里已有的内容也符合
sudo chgrp -R devs /srv/project
sudo chmod -R g+rwX /srv/project  # 大写 X：只给目录和已有可执行文件加 x，不给普通文件加

# 成员需要重新登录才能获得新组
```

**`chmod g+rwX` 里大写 `X` 的含义**（这是一个很实用的细节）：`X` 表示"仅当对象是目录，或已有某个执行位被设置时，才加执行位"。避免把一堆数据文件变成可执行文件。这是"递归设权限"时的正确写法。

**注意 umask 仍然生效**：如上所述，SGID 只解决属组，权限位还是由创建者的 umask 决定。如果团队成员的 umask 是 `022`，新文件会是 `644`，**组内其他人只能读不能改**。要真正协作，成员的 umask 需要设为 `002`（在各自 `~/.bashrc` 里），或者用 ACL（见下）。

### 6.4 Sticky bit

**对目录**：在该目录里，只有**文件的属主、目录的属主或 root** 可以删除或重命名文件。其他人的写权限只能用来**创建新文件**，不能用来**删别人的文件**。

**标准例子：`/tmp`**。在大多数 Linux 发行版上：

```bash
$ ls -ld /tmp
drwxrwxrwt 1 root root ... /tmp
#        ^ 最后的 t 表示 sticky bit 已设置
```

`/tmp` 是 `1777`：**所有用户都能在里面创建文件**（任何程序都需要一个临时目录），但**不能删除别人的文件**。如果没有 sticky bit，任何用户都能删掉别人的临时文件、甚至通过删除 + 重建同名文件来劫持其他程序的运行（这是经典的 `/tmp` 竞争攻击，也与操作系统板块里讲的 TOCTOU 相关）。

```bash
chmod 1777 /tmp           # 或用 chmod +t /tmp
chmod +t shared_upload/   # 上传目录的常见配置
```

**其他常见场景**：共享的上传目录、协作目录里"大家都能建自己的文件但不能乱删别人的"。

**对文件**：sticky bit 在现代 Linux 上对普通文件没有实际效果（它源于早期 Unix 的"把这个程序的正文段常驻内存"优化，已经在现代系统上废弃）。看到文件上的 `t` 位不必紧张，它不起作用。

### 6.5 特殊权限位的组合八进制

| 八进制 | 二进制（特殊位部分） | 含义 |
| --- | --- | --- |
| `4755` | `100` + `755` | SUID，其余 `rwxr-xr-x`（可执行程序） |
| `2755` | `010` + `755` | SGID 文件 |
| `1777` | `001` + `777` | sticky，其余全开（`/tmp`） |
| `2775` | `010` + `775` | SGID 目录，组内可读写（**共享目录**） |
| `2770` | `010` + `770` | SGID 目录，组外完全不可见 |
| `3775` | `011` + `775` | SGID + sticky（组内协作，但只能删自己的文件） |
| `6777` | `110` + `777` | SUID + SGID（几乎总是设计问题） |
| `7777` | `111` + `777` | 三个特殊位全开（极其可疑） |

用符号表示法设置这三个位：

```bash
chmod u+s file        # SUID
chmod g+s dir         # SGID
chmod +t dir          # sticky（+t 不指定 ugo 时通常作用在 other 位，也就是 1xxx 的效果）
chmod u-s file        # 去掉 SUID
chmod 0755 file       # 用完整 4 位数字【明确清掉】所有特殊位（推荐）
```

**安全提示**：当你不确定时，用完整的 4 位数字（如 `0755`、`0644`）来设置权限，可以**明确地把特殊位清零**。而用三位数字（`chmod 755`）在 GNU 的实现里也会清掉特殊位（因为指定了完整的权限模式），但写 4 位更明确、可读性更好。

## 7. 访问控制列表 (ACL) 简介

9 个权限位只能表达"一个属主、一个属组、其他所有人"三档。当需求变成"alice 可读写、bob 只读、carol 完全不能访问"时，三档就不够了。**ACL (Access Control List)** 扩展了这个模型。

```bash
# 查看 ACL（如果只有基本权限，会显示 + 号缺失）
getfacl /srv/project

# 给指定用户单独授权
sudo setfacl -m u:carol:rwx /srv/project
sudo setfacl -m u:bob:r-x /srv/project
sudo setfacl -m g:devs:rwx /srv/project

# 递归设置默认 ACL（新建的文件自动继承）
sudo setfacl -R -m u:carol:rwx /srv/project
sudo setfacl -d -m u:carol:rwx /srv/project    # -d 设置 default ACL

# 移除
sudo setfacl -x u:carol /srv/project
sudo setfacl -b /srv/project                   # 清空所有 ACL
```

`ls -l` 会在权限字符串后显示 `+` 表示该文件有 ACL：

```bash
$ ls -ld /srv/project
drwxrwsr-x+ 2 root devs ... /srv/project
#         ^ 有 ACL
```

**必须诚实说明的两点**：
1. **ACL 依赖文件系统支持**。ext4、XFS、Btrfs 都支持，但需要在挂载时启用 `acl` 选项。多数现代发行版的默认挂载选项已包含它，但这不是绝对保证——用 `mount | grep acl` 或 `tune2fs -l` 确认。
2. **`cp -a` / `rsync -a` 是否保留 ACL 取决于工具与选项**（`rsync` 需要 `-A`）。备份时如果 ACL 丢失，权限会静默降级。

## 8. 排查清单：Permission denied 但"我应该有权限"

按这个顺序排除，能覆盖绝大多数情况：

| # | 检查项 | 命令 | 说明 |
| --- | --- | --- | --- |
| 1 | 文件本身的权限 | `ls -l file` | 我是否在属主/属组/其他里的哪一档？ |
| 2 | 我是谁、在哪些组 | `id` | 属组对不对？**刚加的组需要重新登录** |
| 3 | 路径上每级目录的 `x` | `namei -l /full/path` | 任何一级缺 `x` 都无法穿越 |
| 4 | 写操作看父目录 | `ls -ld $(dirname file)` | 创建/删除需要**父目录**的 `w`+`x` |
| 5 | 挂载选项 | `mount \| grep ' /target '` | `ro`（只读）、`nosuid`、`noexec` 会否决配置 |
| 6 | 强制访问控制 | `getenforce`（SELinux）/ `aa-status`（AppArmor） | 应用层权限对，MAC 层可能仍拒绝 |
| 7 | 是否真的没有权限 | `sudo -l` | 也许我其实有 sudo 可以做 |
| 8 | 文件是否被锁或不可变 | `lsattr file` | `i` 属性（immutable）会拒绝写入，即使 root |
| 9 | 文件系统是否只读/满 | `df -h .`、`dmesg \| tail` | 磁盘满或文件系统错误会导致离奇的写失败 |
| 10 | ACL | `getfacl file` | 9 位权限之外可能还有 ACL 条目在起作用 |

第 5 和第 8 项特别值得强调：**`nosuid` 挂载会让 SUID 位失效**（安全加固的常用手段，但也会让依赖 SUID 的程序失效）；**`i`（immutable）属性**会让文件连 root 都无法修改，必须先用 `chattr -i` 解除——这是"权限看起来没问题却改不了"的一个隐蔽原因。

## 小结

- `/etc/passwd` 的 `x` 只是占位符，真正的口令哈希在只有 root 可读的 `/etc/shadow`；哈希前缀（`$6$`、`$y$`）表示算法，不能靠它判断强弱。
- **UID 0 才是超级用户**，用户名无关；系统账户 UID 范围是**发行版策略**而非内核规定。
- 主组只有一个且在 `/etc/passwd` 里，附加组在 `/etc/group` 的成员列表里；`/etc/group` **不列出**以该组为主组的用户，用 `id`/`getent` 才准确。
- `usermod -aG` 的 `-a` 不能省——`-G` 会**替换**整个附加组列表，是最常见的自锁操作。
- 改完组必须**重新登录**才生效，因为进程的组列表在登录时确定。
- `su` 要目标用户的口令、粒度是全有全无；`sudo` 要自己的口令、可以细到单条命令、有审计记录。多用户环境应当用 sudo。
- `su -` 的 `-` 非常关键：模拟完整登录（重置环境、切家目录）。不带 `-` 会继承当前环境，容易出怪问题。
- 编辑 sudoers **必须用 `visudo`**；优先用 `/etc/sudoers.d/` 下的 drop-in 文件（文件名**不能含点**，权限 `440`）。
- sudoers 里的命令规格要写**绝对路径并限定参数**；允许 `sudo vim` / `sudo systemctl` 这类命令等价于给了完全 root 权限。
- `umask` 是"屏蔽位"，**越大权限越少**；文件基线 `666`、目录基线 `777`。`022` → `644`/`755`，`002` → `664`/`775`（团队），`077` → `600`/`700`（私密）。
- **umask 不能给文件加执行权**，所以 `chmod +x` 总是要手工做；umask 只影响新建对象，不改已存在的。
- SUID 让进程的 **euid** 变成文件属主，是 `passwd` 能改 `/etc/shadow` 的原因；**脚本上的 SUID 被 Linux 忽略**，只有二进制有效。
- SGID 在**目录**上让新建对象继承目录的属组，是共享目录的标准做法（`chmod 2775`）；但它不改变权限位，成员 umask 仍需为 `002` 才能真正协作。
- sticky bit 在目录上让"只能删自己的文件"，`/tmp` 的 `1777` 就是它。
- `ls -l` 里小写 `s`/`t` 表示"特殊位与执行位都设置了"，大写 `S`/`T` 表示"特殊位设了但执行位没设"——大写意味着该位实际不生效。
- 每个 SUID root 二进制都是提权攻击面，`find / -perm -4000 -type f` 是标准审计动作。
- 排查 Permission denied 的顺序：文件权限 → 我的组 → 路径上每级目录的 x → 父目录写权限 → 挂载选项（`ro`/`nosuid`/`noexec`）→ SELinux/AppArmor → `lsattr` 的 immutable → 磁盘状态 → ACL。
