# Linux 学习笔记

Linux 方向的自学笔记索引：从文件系统与进程，到文本处理、权限模型、包管理、Shell 脚本，最后落到虚拟机与开发环境的实际搭建。

## 这个板块在学什么

我学 Linux 的目标很具体：**能独立把一台机器配好、把问题查出来**。所以这个板块不是命令罗列，而是按"你会遇到的任务"组织的：

1. **文件在哪里、怎么动它** —— 目录结构、路径、增删改查、链接
2. **程序在干什么、怎么控制它** —— 进程状态、信号、作业控制、后台运行
3. **怎么从一堆文本里捞出答案** —— grep / sed / awk 与管道
4. **谁能碰什么** —— 用户、组、权限位、sudo、特殊权限
5. **软件从哪来、服务怎么跑** —— apt/dnf 与 systemd
6. **怎么把重复劳动自动化** —— Shell 脚本
7. **怎么从零搭出一个能用的环境** —— VMware、网络模式、换源、工具链

前六个是"会用"，第七个是"能搭"。缺了第七个，前六个总归是在别人的机器上练手。

## 文件说明

| 文件 | 内容 | 难度 |
| --- | --- | --- |
| [01-file-system.md](01-file-system.md) | FHS 目录结构、路径概念、ls/cd/cp/mv/rm/find/ln、权限 rwx 与 chmod/chown 数字表示法 | 入门 |
| [02-process-and-jobs.md](02-process-and-jobs.md) | ps/top/kill/jobs/fg/bg/nohup、进程状态、信号机制 | 入门-中 |
| [03-text-processing.md](03-text-processing.md) | grep/sed/awk/sort/uniq/wc/cut 与管道组合实战 | 中 |
| [04-permissions-and-users.md](04-permissions-and-users.md) | /etc/passwd 与 shadow、用户组管理、sudo 与 sudoers、umask、SUID/SGID/sticky | 中 |
| [05-package-and-service.md](05-package-and-service.md) | apt/dnf/rpm 常用命令、换源原理、systemctl 与 unit 文件、journalctl | 中 |
| [06-shell-scripting.md](06-shell-scripting.md) | 变量、条件、循环、函数、数组、四个完整实用脚本、调试与 shellcheck | 中 |
| [07-vm-and-env-setup.md](07-vm-and-env-setup.md) | VMware 装 Ubuntu、NAT/桥接/仅主机网络模式、快照、换源、Python venv、编译工具链 | 中 |

## 学习路线

下面是我实际走过的顺序，每一步都配一个"过关标准"——达不到就先不要往下走，否则后面全是夹生饭。

```text
阶段一：能在命令行里活着（约 1 周）
  01-file-system.md
    - FHS 目录结构、绝对/相对路径、. 与 ..
    - ls -l 输出逐列读懂、cp/mv/rm/mkdir/find/ln
    - chmod 数字表示法 755 / 644 / 600 能心算
  过关标准：不看笔记能说出 -rwxr-xr-- 对应多少、属于谁能做什么

阶段二：能控制程序（约 1 周）
  02-process-and-jobs.md
    - ps aux 各列含义、进程状态 R/S/D/T/Z
    - 信号：SIGTERM 与 SIGKILL 的区别
    - Ctrl+C / Ctrl+Z / Ctrl+D 三者各是什么
    - jobs / fg / bg / nohup
  过关标准：能解释"为什么 kill -9 杀不掉一个 zombie"和"为什么 nohup 能让进程活过后台关终端"

阶段三：能从文本里捞东西（约 1-2 周，最值得投入）
  03-text-processing.md
    - 重定向与管道的顺序语义（> file 2>&1 vs 2>&1 > file）
    - grep 常用选项、sed 替换与寻址、awk 字段模型
    - sort | uniq -c | sort -rn | head 这条"top N"流水线
  过关标准：给一份 access log，能一条管道写出"访问量前 10 的 IP"

阶段四：能管权限和用户（约 1 周）
  04-permissions-and-users.md
    - /etc/passwd 七个字段、shadow 存在的理由
    - useradd / usermod -aG / sudo / visudo
    - umask 与"默认 666/777 减掉掩码"的计算
    - SUID / SGID / sticky 分别在什么场景下用
  过关标准：能解释 /tmp 为什么是 1777，以及为什么 SUID 脚本是无效的

阶段五：能装东西、能管服务（约 1 周）
  05-package-and-service.md
    - apt update 与 apt upgrade 的区别
    - dpkg -S 找出文件属于哪个包
    - 写一个 systemd unit 并 enable --now
    - journalctl -u 看服务日志
  过关标准：能把自己的一个脚本做成开机自启的服务并查出它为什么起不来

阶段六：能自动化（约 1-2 周）
  06-shell-scripting.md
    - set -euo pipefail 的作用与坑
    - "$VAR" 与 $VAR 的区别（词分隔）、"$@" 的正确用法
    - while read -r line < file 而不是 cat file | while read
    - 四个完整脚本：日志清理、健康巡检、批量备份、服务自愈
  过关标准：能写出带 trap 清理、遇错即退、有 dry-run 的脚本

阶段七：能自己搭环境（约 1 周）
  07-vm-and-env-setup.md
    - VMware 三种网络模式分别谁能访问谁
    - 快照不是备份
    - 换源必须匹配发行版代号
    - pip 不要 sudo、要用 venv
  过关标准：从空白虚拟机到能编译运行一个 C 程序并 ssh 进去，全程不看教程
```

**我自己的时间分配建议**：阶段三（文本处理）值得给最多时间，因为它是唯一一个"学一次、之后每天都在用"的技能。阶段六的脚本能力是阶段三的自然延伸——awk 写得好的人，脚本通常不会太差。

## 配套资源

- 经典书目与公开课程见 [../../resources/reading-list.md](../../resources/reading-list.md)
- 高频命令速查见 [../../resources/command-cheatsheet.md](../../resources/command-cheatsheet.md)
- 与本仓库其他板块的联系：`02-process-and-jobs.md` 的信号与竞态，与 [../operating-systems/02-memory-vulnerabilities.md](../operating-systems/02-memory-vulnerabilities.md) 第 6 节呼应；`07-vm-and-env-setup.md` 里装的 `valgrind` / `-fsanitize=address` 与 [../operating-systems/03-defense-strategies.md](../operating-systems/03-defense-strategies.md) 第 8 节呼应。

## 说明

- 笔记主要以 **Debian/Ubuntu 系**为主线（这也是 07 篇里装的那个发行版），涉及 RHEL/CentOS/Fedora 系的地方会明确标注。两系的命令差异（`apt` vs `dnf`、`ssh` vs `sshd` 的服务名）在文中都会指出。
- 文中所有命令都是可直接执行的真实命令，没有伪造的输出示例。示例输出若为示意会明确写成"（示意）"。
- 本板块是本科生的自学记录，不是官方教材。
