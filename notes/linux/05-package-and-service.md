# 软件包管理与 systemd 服务

本篇梳理 Debian 系的 apt 与 RHEL 系的 dnf/yum 的常用命令与差异、换源的原理与风险，以及 systemd 的 service 管理、unit 文件写法与 journalctl 日志查询。

## 1. 包管理的三个层次

理解这三个层次，就不会把 `apt` 和 `dpkg` 搞混：

| 层次 | 作用 | Debian 系 | RHEL 系 |
| --- | --- | --- | --- |
| 底层工具 | 直接安装/查询**单个包文件**，**不解决依赖** | `dpkg`（`.deb`） | `rpm`（`.rpm`） |
| 高层工具 | 从**仓库**下载、**自动解决依赖**、管理版本 | `apt` | `dnf`（新一代）/ `yum` |
| 仓库与元数据 | 包从哪里来 | `/etc/apt/sources.list`、`/etc/apt/sources.list.d/` | `/etc/yum.repos.d/*.repo` |

**为什么要有两层**：底层工具的职责单一（解包、放文件、跑脚本），高层工具负责"从网络获取元数据、算出依赖闭包、按正确顺序安装"。所以 `sudo dpkg -i ./some-package.deb` 经常因为缺依赖而失败或留下半配置状态，而 `sudo apt install ./some-package.deb` 不会——**日常应该用高层工具**。

### 1.1 各发行版的包管理器对照

| 发行版家族 | 包格式 | 高层工具 | 底层工具 |
| --- | --- | --- | --- |
| Debian / Ubuntu / Mint | `.deb` | `apt`（旧写法 `apt-get`/`apt-cache`） | `dpkg` |
| Fedora / RHEL 8+ / CentOS Stream / Rocky / Alma | `.rpm` | `dnf` | `rpm` |
| RHEL 7 / CentOS 7（旧） | `.rpm` | `yum` | `rpm` |
| openSUSE | `.rpm` | `zypper` | `rpm` |
| Arch / Manjaro | `.pkg.tar.zst` | `pacman` | — |
| Alpine | `.apk` | `apk` | — |

**关于 `yum` 的一个重要事实**：在 RHEL 8 及之后（包括 CentOS Stream、Rocky、Alma、Fedora），`yum` 实际上是**指向 `dnf` 的兼容符号链接/包装**。所以在新系统上敲 `yum install` 能用，但底层跑的是 dnf。学习时以 `dnf` 为准。

## 2. `apt` 常用命令

### 2.1 必须先区分 `update` 与 `upgrade` —— 最常见的混淆

| 命令 | 做什么 | 会不会装/改软件 | 是否要 sudo |
| --- | --- | --- | --- |
| `apt update` | **只刷新包的索引**（从仓库下载"有哪些包、什么版本"的元数据） | ❌ **完全不会** | 需要（索引文件在 `/var/lib/apt/lists/`） |
| `apt upgrade` | 把已安装的包升级到索引里的新版本 | ✅ | 需要 |
| `apt full-upgrade` | 升级，且**允许为了满足依赖而安装新包或移除已有包** | ✅ 可能删包 | 需要 |
| `apt dist-upgrade` | `full-upgrade` 的旧名字，行为相同 | ✅ | 需要 |
| `apt install pkg` | 安装 | ✅ | 需要 |
| `apt remove pkg` | 卸载，**保留配置文件** | ✅ | 需要 |

**为什么必须"先 update 再 upgrade"**：`upgrade` 依据的是本地缓存的索引。如果很久没 `update`，索引是旧的，`upgrade` 就什么也升不了（或者升到旧版本），并且会提示 "packages have been kept back"。这是一个非常常见的新手困惑："我执行了 `apt upgrade`，为什么没有更新？"

**`upgrade` 与 `full-upgrade` 的区别要理解清楚**：`upgrade` 有一条硬规则——**绝不安装新包、绝不删除已有包**。如果某个升级需要引入一个新的依赖包，`upgrade` 就会**跳过这个包**并提示 "kept back"。`full-upgrade` 则允许增删包来完成任务。因此：

- 日常小更新用 `apt upgrade`（更保守）；
- 大版本升级（例如 Ubuntu 22.04 → 24.04）才用 `apt full-upgrade`，**并且要意识到它可能移除包**。

### 2.2 安装、卸载、清理

```bash
sudo apt update                                   # 刷新索引（第一步总是它）
sudo apt install nginx                            # 安装
sudo apt install -y nginx                         # 自动回答 yes（脚本里用）
sudo apt install --dry-run nginx                  # 只显示会做什么，不真做（强烈建议）
sudo apt install nginx=1.24.0-1ubuntu1            # 安装指定版本（版本号要用 apt-cache policy 查）
sudo apt install ./local-package.deb              # 安装本地 deb，同时自动解决依赖

sudo apt remove nginx                             # 卸载，保留配置文件
sudo apt purge nginx                              # 连配置文件一起删
sudo apt autoremove                               # 删除"已不再被任何包依赖"的孤儿包
sudo apt autoremove --purge                       # 连它们的配置也删

sudo apt clean                                    # 清空 /var/cache/apt/archives 里的 .deb 缓存
sudo apt autoclean                                # 只删除已经过期的缓存包
```

**`remove` 与 `purge` 的区别很实际**：`remove` 之后重新 `install`，之前的配置（`/etc/nginx/` 里的东西）还在，可能造成"重装了还是老配置"的困惑。想彻底重来必须 `purge`。

```bash
# 一个完整清理示例
sudo apt autoremove --purge
sudo apt clean
```

### 2.3 查询：找到你想要的信息

```bash
apt search keyword                    # 在包名与描述里搜索
apt show nginx                        # 显示包的详细信息（版本、依赖、大小、描述）
apt list --installed                  # 列出所有已安装的包
apt list --installed | grep nginx     # 按名字过滤
apt list --upgradable                 # 哪些包可以升级
apt-cache policy nginx                # 查看可用版本与当前安装版本（非常有用）
apt-cache depends nginx               # 查看依赖
apt-cache rdepends nginx              # 反向依赖：谁依赖它（删包前必查）
```

`apt-cache policy` 的输出是排查"为什么升级不了"的关键：

```text
nginx:
  Installed: 1.24.0-1ubuntu1
  Candidate: 1.24.0-1ubuntu1.1
  Version table:
     1.24.0-1ubuntu1.1 500
        500 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 Packages
 *** 1.24.0-1ubuntu1 100
        100 /var/lib/dpkg/status
```

`Installed` 是当前版本，`Candidate` 是"下一次 install/upgrade 会装的版本"。如果 Candidate 等于 Installed，说明没有可升级的版本。**如果 Candidate 显示 `(none)`，通常说明当前安装的版本比仓库里的新，或者仓库里已经不再提供这个包。**

### 2.4 `dpkg` —— 底层查询（这些命令很实用）

```bash
dpkg -l                                # 列出所有已知包（ii 表示已正确安装）
dpkg -l | grep nginx                   # 过滤
dpkg -L nginx                          # 【列出该包安装了哪些文件】
dpkg -S /usr/sbin/nginx                # 【查询某个文件属于哪个包】—— 排障利器
dpkg -s nginx                          # 显示包状态信息
dpkg -i package.deb                    # 安装本地 deb（不解决依赖！）
dpkg -r package                        # 卸载
dpkg -P package                        # 彻底卸载（含配置）
dpkg --get-selections > pkgs.txt       # 导出已安装包清单（迁移用）
```

**`dpkg -S` 是我最常用的排障命令**：当你在系统里发现一个来路不明的文件，想知道它是哪个包装的，或者想知道某个命令的包名是什么：

```bash
$ dpkg -S $(which python3)
python3-minimal: /usr/bin/python3
```

### 2.5 修复损坏的包管理状态

```bash
sudo apt --fix-broken install          # 修复依赖被破坏的状态（最常用）
sudo dpkg --configure -a               # 完成所有被中断的配置
sudo apt install -f                    # 等价于 --fix-broken
```

**典型的锁冲突问题**：

```text
E: Could not get lock /var/lib/dpkg/lock-frontend.
   It is held by process 1234 (apt)
```

**正确处理方法**：先看是谁在持有锁，**等它结束**。

```bash
# 1. 找出是谁在跑 apt
ps aux | grep -E '[a]pt|[d]pkg|[u]nattended'
# 或者
sudo lsof /var/lib/dpkg/lock-frontend

# 2. 如果确实是别的 apt 在跑，等它完成（可能是 unattended-upgrades）

# 3. 如果确认是一个卡死的进程（且你确定没有 apt 在正常工作），才终止它
sudo kill <PID>
# 如果它不响应，再考虑
sudo kill -9 <PID>

# 4. 然后必须修复状态
sudo dpkg --configure -a
sudo apt --fix-broken install
```

> ⚠️ **绝对不要**用 `rm /var/lib/dpkg/lock*` 来"解决"锁问题。删掉锁文件并不能保证另一个 apt 已经结束，两个 apt 同时改包数据库可能造成包数据库损坏，那要修起来就麻烦得多。这是网上流传最广的错误建议之一。
>
> 另外，`unattended-upgrades` 会在后台自动运行 apt（Debian/Ubuntu 上常见），所以你看到锁被占用往往是因为它在工作。正确做法是等，或者临时停掉它：`sudo systemctl stop unattended-upgrades`。

## 3. `dnf` / `yum` 常用命令

```bash
sudo dnf check-update                  # 检查可更新（相当于 apt update + list --upgradable）
sudo dnf upgrade                       # 升级所有包（dnf 的 upgrade 允许增删包，与 apt upgrade 不同！）
sudo dnf install nginx                 # 安装
sudo dnf remove nginx                  # 卸载
sudo dnf search keyword                # 搜索
sudo dnf info nginx                    # 详情
sudo dnf list installed                # 已安装列表
sudo dnf list available                # 可安装列表
sudo dnf provides */nginx.conf         # 【反查：哪个包提供这个文件】（相当于 dpkg -S）
sudo dnf repoquery -l nginx            # 【列出包里有哪些文件】（相当于 dpkg -L）
sudo dnf clean all                     # 清理缓存
sudo dnf makecache                     # 重建缓存（相当于 apt update）
sudo dnf history                       # 查看操作历史（每次操作有 ID）
sudo dnf history info 42               # 看某次操作做了什么
sudo dnf history undo 42               # 【回滚某次操作】—— dnf 相比 apt 的一个明显优势
sudo dnf groupinstall "Development Tools"   # 安装包组
```

**`apt upgrade` 与 `dnf upgrade` 的语义差异**（容易踩坑）：Debian 系的 `apt upgrade` **不会**移除已有包，而 `dnf upgrade` **会**为满足依赖而移除包。所以 RHEL 系上"升级"的风险默认更高，生产环境应当先 `dnf check-update` 并用 `--assumeno` 预演。

**`dnf history undo` 是 dnf 的一个实用优势**：能撤销之前的某次事务。apt 没有等价的内建能力（只保留了 `/var/log/apt/history.log` 供人工参考）。

### 3.1 `rpm` 查询

```bash
rpm -qa                                # 列出所有已安装的包
rpm -qa | grep nginx
rpm -qf /usr/sbin/nginx                # 查文件属于哪个包
rpm -ql nginx                          # 列出包里的文件
rpm -qi nginx                          # 包信息
rpm -q --changelog nginx | head -30    # 查看更新日志
rpm -V nginx                           # 校验包内文件是否被修改（安全审计用）
rpm -ivh package.rpm                   # 安装本地 rpm（不解决依赖）
rpm -e nginx                           # 卸载
rpm -qp --scripts package.rpm          # 查看包里带的安装脚本（审计第三方包时重要）
```

**`rpm -V` 值得单独一提**：它校验包内每个文件的大小、权限、MD5 等是否与数据库记录一致。输出里每个位置一个字符表示一项检查通过（`.`）或失败（如 `S`=size、`M`=mode、`5`=MD5、`U`=user、`G`=group、`T`=mtime）。**这是发现"系统二进制被替换"这类入侵痕迹的标准手段**。

## 4. 换源 (switching to a faster mirror)

### 4.1 为什么要换

默认仓库地址在境外时，下载速度可能很慢，导致 `apt update` 和安装过程漫长。换到地理位置更近的镜像站可以显著提速。国内常用的公共镜像站包括**清华大学 TUNA 镜像站、中国科学技术大学 USTC 镜像站、阿里云镜像站、网易镜像站**等（这里只列站名，具体 URL 请到各站官网确认，避免复制过期的地址）。

### 4.2 Debian / Ubuntu 的源配置

**位置**：

| 文件 | 说明 |
| --- | --- |
| `/etc/apt/sources.list` | 传统的主配置文件 |
| `/etc/apt/sources.list.d/*.list` | 附加源，一个文件一个源 |
| `/etc/apt/sources.list.d/*.sources` | **较新版 Ubuntu 的 deb822 格式**（键值对风格，比单行格式更可读） |

**传统 `sources.list` 格式的字段含义**：

```text
deb  http://archive.ubuntu.com/ubuntu  noble  main restricted universe multiverse
 │   └──────────── URI ─────────────┘   │     └──────── component ────────┘
 │                                       └─ suite（发行版代号）
 └─ 类型：deb = 二进制包，deb-src = 源码包
```

| 字段 | 含义 |
| --- | --- |
| `deb` | 二进制包的仓库；`deb-src` 是源码包仓库 |
| URI | 仓库根地址 |
| suite | 发行版**代号**（Ubuntu 用 `noble`、`jammy` 这种代号；Debian 用 `bookworm`）或 `noble-updates`、`noble-security` 这类子仓库 |
| component | 软件分类。Ubuntu: `main`（官方支持的自由软件）、`restricted`（官方支持的专有驱动）、`universe`（社区维护的自由软件）、`multiverse`（受版权/法律限制的软件）。Debian: `main`、`contrib`、`non-free` |

**deb822 格式**（较新 Ubuntu 的风格）：

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

功能等价，只是写得更明确。

**正确的换源步骤**（必须按顺序）：

```bash
# 1. 确认自己的发行版代号（绝对不能弄错！）
lsb_release -cs              # 例如输出 noble
# 或者
. /etc/os-release && echo "$VERSION_CODENAME"

# 2. 备份原始配置
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
# 如果用的是 deb822 格式
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak

# 3. 编辑，把 URI 换成镜像站地址（保持 suite 与 component 不变！）
sudo nano /etc/apt/sources.list

# 4. 刷新索引验证
sudo apt update
# 如果报 "404 Not Found" 或 "Release file ... is not valid yet"，
# 说明源地址或代号写错了，立刻恢复备份
```

> ⚠️ **换源最常见的灾难：把发行版代号写错**。例如在 Ubuntu 24.04 (`noble`) 上写了 `jammy`（22.04 的代号），那么 `apt update` 会成功（因为那个仓库确实存在），但随后安装软件时会拉到**为另一个版本编译的包**，造成依赖地狱、库版本冲突、甚至系统无法启动。
>
> **另一条纪律**：不要在 Debian 上使用 Ubuntu 的源，反之亦然。也不要混用多个发行版的源。

### 4.3 RHEL 系的源配置

```bash
ls /etc/yum.repos.d/           # 每个 .repo 文件是一个仓库
```

```ini
# /etc/yum.repos.d/example.repo
[example]
name=Example Repository
baseurl=http://mirror.example.org/centos/$releasever/os/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-example
```

| 字段 | 含义 |
| --- | --- |
| `[example]` | 仓库 ID，必须唯一 |
| `name` | 人类可读的名称 |
| `baseurl` | 仓库地址（也可用 `mirrorlist=` 指向一个镜像列表） |
| `enabled` | 1 启用，0 禁用 |
| `gpgcheck` | 是否校验包签名 |
| `gpgkey` | 公钥位置 |

`$releasever` 和 `$basearch` 是 dnf 的变量，会在运行时替换成实际值。

**不要关闭 `gpgcheck`**：它验证包的签名，是防止从被劫持的镜像安装恶意包的基本保障。如果签名校验失败，正确做法是**导入正确的公钥**，而不是关掉检查：

```bash
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-<name>
```

```bash
sudo dnf config-manager --add-repo=http://...   # 添加仓库（旧写法，部分版本已改为 --add-repo）
sudo dnf config-manager --set-enabled repo-id  # 启用
sudo dnf config-manager --set-disabled repo-id # 禁用
```

（`dnf config-manager` 的可用子命令在不同版本间有过变化，使用前用 `dnf config-manager --help` 确认。）

### 4.4 换源之外的选择

如果只是偶尔需要加速，不一定非要改全局源：

```bash
# apt：为单次下载指定代理
sudo apt -o Acquire::http::Proxy="http://proxy:3128" update

# pip：使用镜像索引（只影响这一次）
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple requests

# dnf：临时禁用所有仓库，只用一个
sudo dnf --disablerepo='*' --enablerepo=myrepo install pkg
```

## 5. 包管理的实践建议

| 建议 | 原因 |
| --- | --- |
| 优先用仓库安装，而不是手工下载 `.deb`/`.rpm` | 高层工具会解决依赖、记录到数据库、可被 `apt remove` 干净卸载 |
| 不要混用不同发行版/不同版本的包 | ABI 与依赖不匹配，可能破坏系统 |
| 升级前先 `--dry-run` / `--assumeno` | 看清会安装、升级、删除哪些包 |
| 生产环境避免盲目 `full-upgrade` | 它可能移除包；应当分批、有计划地进行 |
| 锁定不要升级的包 | `sudo apt-mark hold nginx` / `sudo dnf versionlock add nginx`，用 `unhold` / `versionlock delete` 解除 |
| 升级前备份配置 | 有些包的升级会带新的默认配置，旧配置可能被覆盖或需要人工合并 |
| 记录操作 | `/var/log/apt/history.log`、`/var/log/dnf.log` 是事后追溯的依据 |
| 关注安全更新 | Debian/Ubuntu 的 `-security` 仓库、RHEL 的 `dnf updateinfo list security` |

```bash
# 查看 apt 的操作历史
less /var/log/apt/history.log

# 只安装安全更新（RHEL 系）
sudo dnf updateinfo list security
sudo dnf upgrade --security

# 只安装安全更新（Ubuntu，需要 unattended-upgrades 或指定源）
sudo unattended-upgrade --dry-run -d
```

## 6. systemd 基础

**systemd 是现代 Linux 的 init 系统与系统管理器**，PID 1 就是它。它替代了旧的 SysV init（`/etc/init.d/` 脚本 + `service`/`chkconfig`）和 Upstart。

### 6.1 `systemctl` 命令表

**服务生命周期**：

| 命令 | 作用 |
| --- | --- |
| `systemctl start nginx` | 立即启动 |
| `systemctl stop nginx` | 立即停止 |
| `systemctl restart nginx` | 重启（先停再启，**会断开现有连接**） |
| `systemctl reload nginx` | **重新加载配置**而不中断服务（服务需要支持；本质上是发 `SIGHUP` 或执行 unit 里的 `ExecReload`） |
| `systemctl reload-or-restart nginx` | 支持 reload 就 reload，否则 restart（**最实用的一个**） |
| `systemctl status nginx` | 查看状态（含最近日志、PID、CGroup） |
| `systemctl is-active nginx` | 只输出 `active`/`inactive`（脚本里用） |
| `systemctl is-failed nginx` | 是否失败 |

**开机自启（与 start/stop 完全独立！）**：

| 命令 | 作用 |
| --- | --- |
| `systemctl enable nginx` | 创建开机自启的符号链接。**不启动服务** |
| `systemctl disable nginx` | 移除自启链接。**不停止正在运行的服务** |
| `systemctl enable --now nginx` | 启用**并**立即启动（等价于 enable + start） |
| `systemctl disable --now nginx` | 禁用**并**立即停止 |
| `systemctl is-enabled nginx` | 是否自启（`enabled`/`disabled`/`static`/`masked`） |
| `systemctl mask nginx` | **屏蔽**：链接到 `/dev/null`，连作为依赖被启动都不行 |
| `systemctl unmask nginx` | 解除屏蔽 |

> **`enable` 与 `start` 是两件独立的事，这是新手最常见的困惑来源。** "我 enable 了，为什么现在没跑？"（因为没 start）和"我 start 了，为什么重启后就没了？"（因为没 enable）是两个方向的问题。
>
> **`mask` 与 `disable` 的区别**：`disable` 只是不自动启动，但**其他服务依赖它时它仍会被拉起来**。`mask` 是彻底堵死——把它链接到 `/dev/null`，任何启动尝试都直接失败。要"确保这个服务绝对不跑"（例如公司策略禁止某个服务），用 `mask`。

**列表与诊断**：

```bash
systemctl list-units --type=service              # 当前加载的服务
systemctl list-units --type=service --all        # 含未运行的
systemctl list-unit-files --type=service         # 所有已安装的 unit 文件及其自启状态
systemctl --failed                               # 【只看失败的服务】—— 排障第一步
systemctl list-units --failed
systemctl --user status myapp                    # 用户级服务（--user）
```

**这是排查时最该先敲的一条**：`systemctl --failed` 一眼就能看到系统上所有处于失败状态的服务。

**最重要的一条：改完 unit 文件必须 `daemon-reload`**：

```bash
sudo systemctl daemon-reload
```

原因：systemd 不会每次都去读磁盘上的 unit 文件，它把 unit 定义缓存在内存里。**编辑了 unit 文件但不 `daemon-reload`，你运行的就是旧定义**——表现为"我改了 `ExecStart` 但报错信息还是老的"，非常容易让人困惑。

**查看与覆盖 unit**：

```bash
systemctl cat nginx                       # 显示 unit 文件的完整内容（含所有覆盖）
systemctl show nginx                      # 显示所有属性（合并默认值后的最终值）
systemctl show nginx -p ExecStart          # 只看某个属性
systemctl edit nginx                      # 创建 drop-in 覆盖文件（推荐方式）
systemctl edit --full nginx               # 编辑完整的 unit 副本
systemctl revert nginx                    # 撤销所有本地修改
```

`systemctl show` 与 `systemctl cat` 的区别值得记住：`cat` 显示**磁盘上的文件**，`show` 显示**systemd 实际生效的值**（已合并默认值与 drop-in）。排查"配置没生效"时两者对比往往能立刻找到答案。

### 6.2 Unit 文件结构

一个 service unit 由三个段组成。

**`[Unit]` —— 元数据与依赖**：

| 指令 | 含义 |
| --- | --- |
| `Description=` | 人类可读的描述，出现在 `status` 输出里 |
| `Documentation=` | 文档链接（`man:`、`file:`、`https:`） |
| `After=` | **顺序依赖**：本服务在列出的 unit **之后**启动。**不表示依赖关系**——如果对方没启动，本服务照样会启动 |
| `Before=` | 在列出的 unit 之前启动 |
| `Requires=` | **强依赖**：对方启动失败则本服务也失败；停止本服务会**同时停止**对方 |
| `Wants=` | **弱依赖**：尽力启动对方，但对方失败**不影响**本服务（**最常用**） |
| `Conflicts=` | 互斥：启动本服务会停止对方 |

**`After=`/`Before=` 与 `Requires=`/`Wants=` 是两个正交的维度**（顺序 vs 依赖），这是 systemd 最需要理解清楚的一点：

```text
After=network.target    →  "等网络就绪后再启动我"（顺序）
Wants=network.target    →  "我希望网络也启动"（依赖，但不强制成功）
Requires=postgresql.service → "必须有数据库，否则我也不该跑"（强依赖）
```

**常见错误**：只写 `Requires=postgresql.service` 而不写 `After=postgresql.service`。这会导致依赖关系存在但**启动顺序不确定**，服务可能在数据库还没就绪时就开始连接而失败。**正确做法是两个都写**（很多情况下用 `Wants=` + `After=` 即可）。

**`[Service]` —— 进程如何运行**：

| 指令 | 含义 |
| --- | --- |
| `Type=simple` | 默认。`ExecStart` 启动的就是主进程，systemd 认为启动即成功 |
| `Type=forking` | 主进程会 fork 到后台，需要配合 `PIDFile=` 才能追踪（传统守护进程） |
| `Type=oneshot` | 只执行一次就结束（适合初始化任务），常搭配 `RemainAfterExit=yes` |
| `Type=notify` | 服务通过 `sd_notify()` 主动通知就绪（最精确） |
| `Type=dbus` | 通过 D-Bus 声明就绪 |
| `ExecStart=` | **启动命令，必须用绝对路径**。`Type=oneshot` 时可以有多条 |
| `ExecStop=` | 停止命令（一般不需要，systemd 默认发 `SIGTERM`） |
| `ExecReload=` | `systemctl reload` 执行的命令 |
| `Restart=` | `no`（默认）/ `on-success` / `on-failure` / `on-abnormal` / `always` |
| `RestartSec=` | 重启前等待多久（**防止疯狂重启打爆日志**） |
| `User=` / `Group=` | 以哪个用户/组运行（**权限最小化的关键**） |
| `WorkingDirectory=` | 工作目录 |
| `Environment=` | 直接设置环境变量 |
| `EnvironmentFile=` | 从文件读环境变量（**适合放密钥，注意文件权限**） |
| `StandardOutput=` / `StandardError=` | 输出去向，默认 `journal`（`journalctl` 能看），可设为 `file:`、`null` |
| `LimitNOFILE=` | 文件描述符上限（高并发服务必须调） |
| `UMask=` | 该服务创建文件的默认权限掩码（比依赖全局 umask 可靠） |
| `KillSignal=` / `TimeoutStopSec=` | 停止时的信号与超时 |
| `ProtectSystem=` / `ProtectHome=` / `PrivateTmp=` | **沙箱加固选项**，限制服务能看到的文件系统 |

**`[Install]` —— 如何被 enable**：

| 指令 | 含义 |
| --- | --- |
| `WantedBy=multi-user.target` | `enable` 时创建到 `multi-user.target.wants/` 的链接。**这是绝大多数服务的标准写法** |
| `RequiredBy=` | 与 `WantedBy` 类似但更强 |
| `Alias=` | 为 unit 建立别名 |

**注意**：`[Install]` 段决定 `enable` 的行为。**如果一个 unit 没有 `[Install]` 段，`enable` 会失败**（报 "has no installation config"）——这是"为什么我的服务 enable 不了"的答案。这类 unit 通常是 `static` 的（只能被别的 unit 依赖拉起）。

### 6.3 一个完整的、可用的 unit 示例

假设我们有一个用 Python 写的 Web 服务 `/opt/myapp/app.py`，想让它作为系统服务运行。

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Python Web Application
Documentation=https://example.com/myapp/docs
# 顺序：等网络和文件系统挂载完成后再启动
After=network-online.target
Wants=network-online.target

[Service]
# Type=simple：ExecStart 启动的进程就是主进程（Python 直接前台运行，属于这种）
Type=simple

# 以非 root 用户运行 —— 权限最小化，这是最重要的一行
User=myapp
Group=myapp
WorkingDirectory=/opt/myapp

# 环境变量：非敏感的直接写
Environment=APP_ENV=production
Environment=PORT=8080
# 敏感的（数据库密码、API key）放在单独文件里，权限设为 600
EnvironmentFile=-/etc/myapp/myapp.env
# 注意路径前的 "-"：表示该文件不存在时不报错

# 启动命令必须用【绝对路径】
ExecStart=/opt/myapp/.venv/bin/python3 /opt/myapp/app.py

# reload 时重新加载配置（如果应用支持 SIGHUP）
ExecReload=/bin/kill -HUP $MAINPID

# 异常退出就重启，等 5 秒（避免疯狂重启刷爆日志）
Restart=on-failure
RestartSec=5s

# 日志交给 journald（默认行为，写出来是为了明确）
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# 加固：限制服务能看到什么
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/var/lib/myapp

# 提高文件描述符上限（高并发服务常需要）
LimitNOFILE=65536

[Install]
# enable 时创建到 multi-user.target.wants/ 的符号链接
WantedBy=multi-user.target
```

**安装与启用**：

```bash
# 1. 确保运行身份存在
sudo useradd --system --no-create-home --shell /usr/sbin/nologin myapp

# 2. 准备环境变量文件并收紧权限
sudo install -d -m 0755 /etc/myapp
sudo install -m 0600 -o root -g root /dev/null /etc/myapp/myapp.env
sudo tee /etc/myapp/myapp.env >/dev/null <<'EOF'
DATABASE_URL=postgresql://myapp:secret@localhost/myapp
SECRET_KEY=replace-with-a-real-secret
EOF
# 注意 <<'EOF' 用【单引号】可以阻止 shell 展开 $ 变量
# 若用 <<EOF，上面的内容会被 shell 尝试展开

# 3. 准备目录并交给服务用户
sudo install -d -o myapp -g myapp -m 0755 /var/lib/myapp

# 4. 注册 unit
sudo cp myapp.service /etc/systemd/system/myapp.service
sudo systemctl daemon-reload                  # 【必须！】

# 5. 启用并启动
sudo systemctl enable --now myapp

# 6. 验证
systemctl status myapp
journalctl -u myapp -n 50 --no-pager
```

**这个示例里几个值得注意的设计**：
- `User=myapp` + `NoNewPrivileges=true` + `ProtectSystem=strict`：三层权限收紧，即使服务被攻破，能造成的破坏也有限（与本仓库操作系统板块的"纵深防御"思路一致）；
- `EnvironmentFile=-`：路径前的 `-` 让文件缺失不致命，便于开发环境；
- `Restart=on-failure` + `RestartSec=5s`：不是 `always`（正常退出不该重启），且有延迟防止重启风暴；
- `WantedBy=multi-user.target`：标准做法，保证在多用户模式下被拉起。

### 6.4 `journalctl` —— 查日志

systemd 的日志由 **journald** 收集，查询工具是 `journalctl`。

```bash
journalctl -u myapp                    # 某个 unit 的日志
journalctl -u myapp -f                 # 持续跟踪（相当于 tail -f）
journalctl -u myapp -n 100             # 最近 100 行
journalctl -u myapp --since "1 hour ago"
journalctl -u myapp --since "2025-03-12 10:00" --until "2025-03-12 11:00"
journalctl -u myapp -p err             # 只看 err 及以上级别
journalctl -u myapp -p warning -b      # 本次启动以来的 warning 及以上
journalctl -b                          # 本次启动的所有日志
journalctl -b -1                       # 上一次启动（-2 是上上次）—— 排障利器
journalctl -k                          # 只看内核日志（相当于 dmesg）
journalctl --disk-usage                # 日志占了多少磁盘
journalctl --list-boots                # 列出所有启动记录
journalctl -o json-pretty -n 1         # 以 JSON 查看（字段最全）
journalctl _PID=1234                   # 按 PID 过滤
journalctl -u myapp -o short-precise    # 精确到微秒的时间戳
```

**日志级别**（`-p` 参数，从高到低）：

| 级别 | 名称 | 含义 |
| --- | --- | --- |
| 0 | emerg | 系统不可用 |
| 1 | alert | 必须立即处理 |
| 2 | crit | 严重 |
| 3 | err | 错误 |
| 4 | warning | 警告 |
| 5 | notice | 正常但重要 |
| 6 | info | 信息 |
| 7 | debug | 调试 |

`-p 3` 表示"级别 3 及更严重"（因为数字越小越严重）。这是容易搞反的地方。

**日志持久化 —— 一个非常重要的事实**：

默认情况下，很多发行版**不保存历史日志**：日志写在 `/run/log/journal/`，而 `/run` 是 **tmpfs（内存）**，**重启即丢失**。所以你会发现"`journalctl -b -1` 说没有上次启动的记录"。

要让日志跨重启保留，需要创建 `/var/log/journal/`：

```bash
sudo mkdir -p /var/log/journal
sudo systemd-tmpfiles --create --prefix /var/log/journal
sudo systemctl restart systemd-journald
# 验证
journalctl --disk-usage
```

或者改 `/etc/systemd/journald.conf` 里的 `Storage=persistent`（并去掉 `#` 注释）。

**日志占满磁盘的常见问题**：即使持久化，journald 也有体积上限（`SystemMaxUse=`，默认约为文件系统大小的 10%）。可以限制与清理：

```bash
journalctl --disk-usage
sudo journalctl --vacuum-size=500M      # 清理到总共 500M
sudo journalctl --vacuum-time=30d       # 只保留 30 天
```

在 `/etc/systemd/journald.conf` 里设置：

```ini
[Journal]
Storage=persistent
SystemMaxUse=1G
SystemMaxFileSize=100M
MaxRetentionSec=1month
```

改完要 `sudo systemctl restart systemd-journald`。

### 6.5 targets

**target 是 systemd 对"一组 unit"的分组**，用来表达系统状态，替代了旧的 runlevel。

| target | 对应旧 runlevel | 含义 |
| --- | --- | --- |
| `poweroff.target` | 0 | 关机 |
| `rescue.target` | 1 (单用户) | 单用户救援模式 |
| `multi-user.target` | 3 | 多用户、无图形界面（**服务器默认**） |
| `graphical.target` | 5 | 多用户 + 图形界面（**桌面默认**） |
| `reboot.target` | 6 | 重启 |
| `emergency.target` | — | 更极端的救援（只有 root shell，几乎不挂载任何东西） |

```bash
systemctl get-default                     # 当前默认 target
sudo systemctl set-default multi-user.target   # 设为无图形界面启动
sudo systemctl isolate multi-user.target  # 【立即切换】到该 target（不用重启）
# 旧的对应写法：
#   sudo systemctl isolate rescue.target   ≈ 旧 telinit 1
```

常用的非 target 单元：

| 单元 | 含义 |
| --- | --- |
| `network.target` | 网络配置已完成（**不代表网络已连通**） |
| `network-online.target` | 网络真正可用（等待时间更长，需要它的服务才写） |
| `local-fs.target` | 本地文件系统已挂载 |
| `basic.target` | 基础系统就绪（局部文件系统挂载、socket 就绪） |
| `sockets.target` | socket 激活的 units 就绪 |
| `sysinit.target` | 系统初始化完成 |

### 6.6 与旧机制 (SysV init) 的对应

| 旧命令 | systemd 等价 |
| --- | --- |
| `service nginx start` | `systemctl start nginx` |
| `service nginx status` | `systemctl status nginx` |
| `chkconfig nginx on` | `systemctl enable nginx` |
| `chkconfig --list` | `systemctl list-unit-files` |
| `chkconfig --list nginx` | `systemctl is-enabled nginx` |
| `/etc/init.d/nginx restart` | `systemctl restart nginx` |
| `telinit 3` | `systemctl isolate multi-user.target` |
| 查看 `/var/log/messages` | `journalctl` |

在 systemd 系统上，`service` 和 `chkconfig` 命令仍然存在（作为兼容封装），它们**内部会转发给 systemctl**。所以现在敲 `service nginx start` 是能用的，但新写文档与脚本应当直接用 `systemctl`。

## 7. 排查实践：服务起不来怎么办

按这个顺序，几乎能定位所有"服务启动失败"：

```bash
# 1. 看状态 —— 第一条命令
systemctl status myapp
# 关注：Active 一行（active/failed）、退出码、以及最后几行日志

# 2. 看完整日志
journalctl -u myapp -n 100 --no-pager
journalctl -u myapp -p err -b            # 本次启动的错误

# 3. 看 systemd 的通用诊断（它会给出具体建议！）
systemd-analyze verify /etc/systemd/system/myapp.service
# 例如：提示 "ExecStart path is not absolute"

# 4. 【最关键的一步】手动以相同身份、相同目录执行 ExecStart
sudo -u myapp /opt/myapp/.venv/bin/python3 /opt/myapp/app.py
# 这一步能直接把应用自己的错误信息暴露出来，
# 而这些信息经常因为"日志还没初始化好"而没有进 journal

# 5. 逐个排查常见原因
```

**常见原因清单**：

| 症状 | 原因 | 排查命令 |
| --- | --- | --- |
| 改了 unit 但行为没变 | **忘了 `daemon-reload`** | `systemctl show myapp -p ExecStart` 与实际文件对比 |
| `enable` 报 "has no installation config" | unit 缺 `[Install]` 段 | `systemctl cat myapp` |
| `ExecStart path is not absolute` | 用了相对路径或依赖 PATH | 改成绝对路径 |
| 启动几秒后失败，日志里有 "address already in use" | 端口被占用 | `ss -lntp \| grep 8080` |
| 权限不足 | `User=` 指定的用户不能读文件/写目录 | `sudo -u myapp test -r <file>`、`ls -l` |
| 一直重启，日志刷屏 | `Restart=always` 但没有 `RestartSec` | 加 `RestartSec=5s`，并先查根因 |
| 状态是 `active (exited)` 但没在跑 | `Type=oneshot` 且执行完就退了 | 改成 `Type=simple` 或检查 `RemainAfterExit` |
| 状态是 `activating` 一直不变 | `Type=notify` 但程序没有 `sd_notify()` | 改成 `Type=simple` |
| 手动跑能行，服务跑不行 | 环境变量/工作目录/用户不同 | 对比 `systemctl show myapp` 与手动环境 |
| SELinux 拒绝 | 安全上下文不允许 | `sudo ausearch -m avc -ts recent`、`ls -Z` |
| 找不到命令 | unit 里 PATH 很有限 | 全部写绝对路径 |
| `EnvironmentFile` 没生效 | 路径写错，或文件权限不允许该用户读 | `systemctl show myapp -p Environment` |

**最后一条通用技巧**：`systemctl show <unit>` 显示的是 systemd **实际生效**的完整配置（含默认值）。把它导出保存下来，与你的 unit 文件对比，几乎所有"我以为的设置没生效"问题都能立刻现形：

```bash
systemctl show myapp > /tmp/effective.txt
less /tmp/effective.txt
```

这个思路与 [../operating-systems/04-config-vs-runtime-gap.md](../operating-systems/04-config-vs-runtime-gap.md) 里讲的"不要读配置文件来确认行为，要问系统它实际认为的配置是什么"完全一致。

## 小结

- 包管理分三层：底层工具（`dpkg`/`rpm`）不解依赖，高层工具（`apt`/`dnf`）解依赖，仓库提供元数据。**日常用高层工具**。
- 在 RHEL 8+ 上 `yum` 就是 `dnf` 的兼容封装。
- `apt update` **只刷新索引，不装任何东西**；`apt upgrade` 依索引升级且**绝不增删包**；`apt full-upgrade` 允许增删包。**"先 update 再 upgrade"是必须的顺序**。
- `apt remove` 保留配置，`purge` 才彻底删；`dpkg -S <文件>` 是"这个文件属于哪个包"的排障利器，`dpkg -L <包>` 反向列出包内文件。
- **不要用删锁文件的方式解决 apt 锁冲突**——先确认持有者并等待，可能是 `unattended-upgrades` 在正常工作。
- `dnf upgrade` 与 `apt upgrade` 语义不同：dnf 会为满足依赖而移除包。`dnf history undo` 能回滚事务，是 apt 没有的能力。
- `rpm -V` 校验包内文件是否被篡改（`S`/`M`/`5`/`U`/`G`/`T` 标记），是入侵排查的标准手段。
- 换源的核心风险是**发行版代号写错**，会导致拉到为其他版本编译的包。必须先 `lsb_release -cs` 确认代号，先备份，换完立刻 `apt update` 验证并保留回滚能力。
- 不要关闭 `gpgcheck`；签名验证失败应导入正确公钥，而不是跳过校验。
- `systemctl enable`（开机自启）与 `start`（立即启动）**完全独立**；`mask` 比 `disable` 更强（连作为依赖都拉不起来）。
- **改完 unit 文件必须 `systemctl daemon-reload`**，因为 systemd 缓存 unit 定义。
- `After=` 是**顺序**，`Requires=`/`Wants=` 是**依赖**，两者正交，需要时应当同时写。用 `Wants=` 而不是 `Requires=` 通常更健壮。
- unit 没有 `[Install]` 段时 `enable` 会失败；`ExecStart` 必须用绝对路径。
- `Type=notify` 需要程序主动调用 `sd_notify()`，否则会一直卡在 activating；`Type=oneshot` 适合一次性任务。
- `journalctl` 的 `-p` 数字**越小越严重**；`-b -1` 看上次启动的日志。
- **journald 默认可能不持久化**（日志在 `/run`，重启即丢）。要保留历史日志需创建 `/var/log/journal/` 或设 `Storage=persistent`。
- 服务起不来的排查顺序：`status` → `journalctl -u` → **手动以相同身份执行 ExecStart** → `systemd-analyze verify` → 端口/权限/环境/`systemctl show` 对比。
