# VMware 虚拟机与 Linux 开发环境搭建

本篇记录用 VMware 装 Linux 虚拟机的完整流程、NAT/桥接/仅主机三种网络模式的取舍、快照的正确用法，以及换源、Python 环境、编译工具链等开发环境的配置。

## 1. 为什么要用虚拟机

| 好处 | 说明 |
| --- | --- |
| 隔离 | 实验不会影响宿主机；危险操作（改内核参数、玩权限、跑漏洞样例）有安全边界 |
| 快照回滚 | 折腾坏了 30 秒回到干净状态，这是虚拟机最大的价值 |
| 多发行版并存 | 同时跑 Ubuntu、CentOS、Alpine，互不干扰 |
| 可复制 | 配好的环境可以直接复制给别人，或者复制多份做集群实验 |
| 网络实验自由 | 可以随便改 IP、断网、模拟内网，不怕把自己网络搞坏 |

**同时也要诚实地说代价**：

| 代价 | 说明 |
| --- | --- |
| 性能开销 | CPU 有虚拟化加速后损耗较小，但磁盘 I/O 与图形性能明显不如物理机 |
| 磁盘占用 | 一个最小化的 Ubuntu 桌面就 10 GB 起，加上快照和缓存很快到 30-50 GB |
| 内存竞争 | 虚拟机的内存是从宿主机切的，8 GB 宿主机跑两个 2 GB 的虚拟机就很紧张 |
| 嵌套虚拟化麻烦 | 在虚拟机里再跑虚拟机（Docker Desktop、WSL2、KVM）需要额外配置且性能打折 |
| 硬件直通困难 | USB 设备、显卡、蓝牙需要额外设置，且不总是可靠 |

**我的建议**：如果只是想学 Linux 命令，WSL2 更轻量；如果要做系统级实验（内核、网络、权限、服务管理）、或者要模拟"一台真正的服务器"，虚拟机是更合适的选择，因为它是**完整独立的操作系统**，有真正的 init、真正的内核、真正的网络栈。

## 2. 前置准备

### 2.1 打开硬件虚拟化

必须在 BIOS/UEFI 里开启 **Intel VT-x** 或 **AMD-V**，否则虚拟机性能极差甚至无法启动 64 位系统。

**检查是否已开启**：

```powershell
# Windows PowerShell 中查看（结果非 0 说明已启用）
(Get-CimInstance Win32_Processor).VirtualizationFirmwareEnabled
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent
```

```bash
# Linux 宿主机上检查
grep -E -o 'vmx|svm' /proc/cpuinfo | sort -u
# vmx = Intel VT-x, svm = AMD-V
```

进入 BIOS 的按键因厂商而异（常见 `F2`、`F10`、`Del`、`Esc`），开机时通常有提示。找不到时按"Virtualization Technology"、"Intel VT-x"、"SVM Mode"这类关键词找。

### 2.2 Windows 上 Hyper-V/VBS 与 VMware 的兼容性

这是一个**真实存在、且很容易踩的坑**：Windows 上的 Hyper-V、WSL2、Windows Sandbox、以及基于虚拟化的安全 (VBS, Virtualization-Based Security) 都会占用 CPU 的虚拟化能力（让 Windows 本身变成一个 hypervisor）。历史上这会使 VMware Workstation 性能大幅下降甚至无法运行。

**处理思路**（不要凭空猜某个版本的具体行为，按下面的顺序实际验证）：

```powershell
# 1. 看 Windows 是否已经在跑 hypervisor
(Get-CimInstance Win32_ComputerSystem).HypervisorPresent
# True 表示有 hypervisor 在运行（Hyper-V/WSL2/VBS 等）

# 2. 查看相关可选功能状态
Get-WindowsOptionalFeature -Online | Where-Object FeatureName -match 'Hyper-V|VirtualMachinePlatform|WindowsHypervisorPlatform' |
    Select-Object FeatureName, State
```

**两种方向**：

- **方案 A（保留 Hyper-V/WSL2）**：装较新版本的 VMware Workstation，它可以使用 **Windows Hypervisor Platform (WHP)** API，与 Hyper-V 共存（可能有一定性能损失）。这是目前更推荐的路线，因为你不用放弃 WSL2。
- **方案 B（完全关闭）**：关闭 Hyper-V 与 VBS 相关功能，让 VMware 直接使用硬件虚拟化。命令类似 `bcdedit /set hypervisorlaunchtype off` 与关闭相关 Windows 功能，**都需要管理员权限且需要重启**。注意关闭 Hyper-V 会让 WSL2 与 Docker Desktop 的 Hyper-V 后端不可用。

> 因为我无法在别人的机器上确认具体版本的行为，这里只给出**判断方法**与**两个方向**，不给具体版本号。装之前先跑上面那两条 PowerShell 命令看实际情况。

**其他前置检查**：

- 磁盘空间：至少留 60 GB 空闲（虚拟机 + 快照增长）
- 宿主机内存：建议 16 GB 起，虚拟机分 4-8 GB
- 关闭宿主机上会干扰的软件：某些杀毒软件的实时监控会显著拖慢虚拟机磁盘 I/O

## 3. 安装流程

VMware Workstation 是真实产品名（VMware 现属 Broadcom）。个人使用有免费授权，具体下载页与授权方式以官网当前说明为准（**不要从我这里抄版本号或下载链接**）。

### 3.1 选发行版

| 发行版 | 适合谁 | 说明 |
| --- | --- | --- |
| **Ubuntu LTS** | 新手、想少折腾 | 生态最好，文档最多，遇到问题好搜。**LTS = Long Term Support，长期支持版，支持周期比普通版长得多**，适合装完不折腾 |
| Ubuntu 普通版 | 想用新特性 | 支持周期短，需要频繁升级 |
| Debian | 想要稳定、干净 | 比 Ubuntu 少一些预装与便利工具，更接近"纯"Linux |
| Fedora | 想学 RHEL 系 | 新技术引入快，dnf 系命令 |
| Rocky / Alma | 想学服务器运维 | RHEL 兼容，企业环境最常见 |
| Arch | 想彻底搞懂 Linux | 从零手工装，学习曲线陡，但收获最大 |

**我建议第一次装 Ubuntu LTS**：不是因为它是"最好的"，而是因为**出问题时最容易搜到答案**。等熟悉了再折腾其他发行版。

### 3.2 新建虚拟机

1. **新建虚拟机** → 选「典型 (Typical)」（自定义用于特殊硬件需求）；想完全控制参数就选「自定义 (Advanced)」
2. **安装来源** → 选「安装程序光盘映像文件 (ISO)」，指向下载好的 `.iso`
3. **客户机操作系统** → 选 Linux，版本按实际选（例如 Ubuntu 64-bit）。选对了 VMware 会自动配置合适的默认参数（磁盘控制器类型、网络适配器类型）
4. **虚拟机名称与位置** → 名字只是显示用；**位置要选空间充足且不是系统盘的路径**，因为虚拟机文件会持续增长
5. **磁盘大小** → 建议 **40 GB 起**（够装桌面 + 开发工具链）
6. **磁盘存储方式**：
   - 「将虚拟磁盘存储为单个文件」：性能略好，但单个大文件不便移动到 FAT32 设备
   - 「拆分成多个文件」：便于移动与备份，性能差异在日常使用中不明显
7. **自定义硬件** → 调内存、CPU、网络适配器（见下一节）、是否加载打印机等

### 3.3 资源分配建议

| 资源 | 建议值 | 说明 |
| --- | --- | --- |
| CPU 核心 | 宿主机物理核心数的 1/4 ~ 1/2 | **不要超过物理核心数**（不是逻辑线程数）；分太多会与宿主机抢 |
| 内存 | 4 GB（纯命令行）～ 8 GB（桌面 + IDE） | 不要超过宿主机内存的 50%，否则宿主机自己会开始 swap，整体变慢 |
| 磁盘 | 40 GB 起，选"动态扩展" | 虚拟磁盘是稀疏文件，实际占用随使用增长 |
| 显存 | 默认即可，除非要跑图形程序 | 桌面环境需要至少 128 MB |
| 网络 | NAT（新手推荐，见下一节） | |

**一个实用的分配原则**：让宿主机始终保持"不 swap、不卡顿"的状态。虚拟机的性能上限不是它自己有多少资源，而是宿主机还剩多少余量。

### 3.4 安装 Ubuntu 过程中的关键选择

| 步骤 | 建议 | 说明 |
| --- | --- | --- |
| 语言 / 键盘布局 | 按需要 | 键盘布局选错会导致符号打不出来，要选 `English (US)` 而不是 `English (UK)`，除非你的键盘真的是 UK 布局 |
| 网络 | 保持连接 | 安装时下载更新会慢很多；可以选"最小安装"跳过 |
| 安装类型 | 默认 | 见下面的分区说明 |
| 磁盘分区 | **新手用"擦除磁盘并安装"** | 因为整块"磁盘"就是虚拟机的一个文件，不怕 |
| 用户名 / 计算机名 | 字母数字，不要含空格 | 用户名会成为家目录名与主机名的一部分 |
| 登录方式 | 设密码即可 | 不需要自动登录（除非你确定这台机器没有敏感数据） |

**关于分区**：安装器默认会创建 EFI 分区（或 BIOS boot 分区）+ 根分区 `/`。新手不用手动分区。想学分区的话，手动方案的常见结构是：

```text
/boot/efi    512 MB   (UEFI 引导，FAT32)
/boot        1 GB     (内核与 initramfs；也可以并入 /)
/            20-40 GB (根，ext4)
swap         2-4 GB   (交换；内存大的机器可以少给或不用)
/home        剩余     (用户数据独立，重装系统不影响)
```

**LVM (Logical Volume Manager)** 的价值是可以在线扩缩容、做快照。在虚拟机里学 LVM 是好主意，但日常桌面不用也没问题。

### 3.5 安装 VMware Tools / open-vm-tools

装完系统后需要安装"虚拟化增强工具"，它提供：

- **分辨率自适应**（拖动窗口时客户机分辨率跟着变）
- **宿主机与客户机之间的复制粘贴**
- **共享文件夹**
- **拖拽文件**
- **时间同步**
- 更优雅的关机

**两条路线**：

| 方案 | 说明 |
| --- | --- |
| **open-vm-tools**（推荐） | 由发行版仓库维护的开源实现，`apt install` 就能装，随系统更新。**绝大多数情况下应该用这个** |
| VMware 官方的 VMware Tools | 从 VMware 的菜单「虚拟机 → 安装 VMware Tools」挂载一个 ISO，手工编译安装。在新内核上容易编译失败 |

```bash
# Ubuntu / Debian 上的安装（推荐方案）
sudo apt update
sudo apt install -y open-vm-tools open-vm-tools-desktop
#                                    ^^^^^^^^^^^^^^^^^^^^ 桌面版才有剪贴板共享与分辨率自适应
# 如果不需要图形界面（纯服务器），只装 open-vm-tools

sudo reboot        # 或者至少重启相关服务
# systemctl status vmtoolsd    # 检查服务
```

**open-vm-tools 装好后仍不生效的常见原因**：
- 装的是 `open-vm-tools` 而不是 `open-vm-tools-desktop`（后者才管剪贴板与分辨率）；
- 没有重启（或者 `vmtoolsd` 服务没起来）；
- 虚拟机设置里禁用了相关功能。

## 4. 网络模式 —— 本篇重点

虚拟机要联网，必须理解 VMware 提供的三种模式。**选错模式是"虚拟机没网"的第一大原因。**

### 4.1 三种模式对比

| 维度 | 桥接 (Bridged) | NAT | 仅主机 (Host-only) |
| --- | --- | --- | --- |
| 虚拟机在哪个网络 | **与宿主机同一个物理网络** | VMware 创建的虚拟网络 (VMnet8) | VMware 创建的内部网络 (VMnet1) |
| 虚拟机 IP 来源 | 物理网络的 DHCP（或手工静态） | VMware 的 DHCP 服务 | VMware 的 DHCP 服务 |
| 能否访问外网 | ✅ 可以 | ✅ 可以（经宿主机做地址转换） | ❌ **不能** |
| 外网/局域网能否主动访问虚拟机 | ✅ 可以（虚拟机有同网段 IP） | ❌ 默认不行（需配端口转发） | ❌ 不行 |
| 宿主机能否访问虚拟机 | ✅ | ✅ | ✅ |
| 是否占用物理网络的一个 IP | ✅ 占用 | ❌ 不占 | ❌ 不占 |
| 依赖外部网络策略 | ✅ 高（企业/校园网可能限制 MAC 数量或要求认证） | ❌ 低 | ❌ 低 |
| 换到别的网络后是否受影响 | ✅ 会（IP 段变了） | ❌ 基本不受影响 | ❌ 不受影响 |
| **典型用途** | 需要被局域网其他机器访问（做本地服务器、被同学访问）；模拟真实主机 | **日常学习（最省事）** | 宿主机-虚拟机隔离实验（网络靶场、协议测试） |

### 4.2 图解

```text
【桥接 Bridged】
   物理网络 (例如校园网 10.x.x.x / 家用 192.168.1.x)
   ┌────────────────────────────────────────────────────┐
   │  路由器 / 网关                                       │
   │     │                                                │
   │     ├── 宿主机  192.168.1.10  ← 直接在同一网段        │
   │     │                                                │
   │     └── 虚拟机  192.168.1.11  ← 也直接在同一网段      │
   │            （有独立的 MAC，对外部网络可见）            │
   └────────────────────────────────────────────────────┘
   外部机器可以 ping 通 192.168.1.11


【NAT】
   物理网络
   ┌────────────────────────────────────────────────────┐
   │  宿主机  192.168.1.10                                │
   │    │                                                 │
   │    └── VMware 虚拟网络 VMnet8  192.168.x.0/24         │
   │          ├── 虚拟网关  192.168.x.2                    │
   │          └── 虚拟机  192.168.x.128                    │
   │               （对外表现为宿主机的 IP，做地址转换）    │
   └────────────────────────────────────────────────────┘
   虚拟机 → 外网  ✅（经宿主机 NAT）
   外网 → 虚拟机  ❌（除非在 VMware 里配端口转发：宿主机:8080 → 虚拟机:80）


【仅主机 Host-only】
   ┌────────────────────────────────────────────────────┐
   │  宿主机  192.168.56.1  (虚拟网卡 VMnet1)             │
   │    │                                                 │
   │    └── 虚拟机  192.168.56.128                        │
   │          （只有这两台能互相看见，完全与外界隔离）      │
   └────────────────────────────────────────────────────┘
   虚拟机 → 外网  ❌
   宿主机 ↔ 虚拟机  ✅
```

### 4.3 关于 VMnet 编号

VMware 会在宿主机上创建几块虚拟网卡，命名规律**通常是**：

| 虚拟网络 | 常见用途 | 典型网段 |
| --- | --- | --- |
| `VMnet0` | 桥接（桥接到宿主机的物理网卡） | 与物理网络相同 |
| `VMnet1` | 仅主机 (Host-only) | 常见 `192.168.56.0/24` |
| `VMnet8` | NAT | 常见 `192.168.x.0/24`（如 `192.168.75.0/24`） |

**必须诚实说明**：这些编号与网段是**默认值**，可以在 VMware 的「编辑 → 虚拟网络编辑器」里改。所以不要在脚本里硬编码 `192.168.56.x`——**永远从虚拟机里 `ip a` 查出实际 IP**。

### 4.4 我的选择建议

| 场景 | 选择 |
| --- | --- |
| 日常学习、装完就想用 | **NAT**（最省事，换个网络环境也不受影响） |
| 想让同学/手机访问我虚拟机上的服务 | 桥接（或 NAT + 端口转发） |
| 只想宿主机和虚拟机互相通信 | 仅主机 |
| 做网络靶场（模拟内网渗透实验） | 仅主机（+ 第二块网卡用 NAT 让靶机能更新） |
| 需要同时具备"能上网"和"被访问" | 加**两块网卡**：一块 NAT 出网，一块桥接对外 |

**多网卡是一个很实用的技巧**：在 VMware 虚拟机设置里「添加 → 网络适配器」，然后分别设置不同的模式。虚拟机里会有 `ens33`、`ens37` 两块网卡，可以分别配 IP。这样"出网走 NAT、对外服务走桥接"就同时满足了。

### 4.5 常见网络问题排查

**问题 1：虚拟机完全没网**

按这个顺序排查：

```bash
# 1. 网卡是否被识别、是否有 IP
ip a                             # 注意看有没有 "inet " 行，以及网卡状态 UP
ip route                         # 是否有 default via ... 的默认路由

# 2. 网卡是否被禁用（NetworkManager）
nmcli device status              # 看 state 是不是 disconnected
nmcli connection show
sudo nmcli connection up "Wired connection 1"

# 3. 能否 ping 通网关（先用 ip route 查网关地址）
ping -c 3 192.168.75.2

# 4. 能否 ping 通外网 IP（绕过 DNS 问题）
ping -c 3 1.1.1.1

# 5. 如果 IP 通但域名不通 → DNS 问题
cat /etc/resolv.conf
resolvectl status                # systemd-resolved 的系统上
nslookup example.com
```

**分层判断的逻辑**（这个逻辑比记命令更重要）：

```text
有 IP 吗？
├─ 没有 → 网卡没启用 / DHCP 拿不到地址 / 虚拟网络配置被删
│          → 检查 nmcli、VMware 虚拟网络编辑器、虚拟网卡在宿主机是否被禁用
└─ 有 IP
   ├─ 能 ping 通网关吗？
   │  ├─ 不能 → 虚拟网络的网段配置与虚拟机的 IP 不匹配
   │  └─ 能
   │     ├─ 能 ping 通 1.1.1.1（IP）吗？
   │     │  ├─ 不能 → 路由或 NAT 问题（NAT 服务没起来、防火墙）
   │     │  └─ 能
   │     │     └─ 能 ping 通域名吗？
   │     │        ├─ 不能 → DNS 配置问题（改 /etc/resolv.conf 或 NM 配置）
   │     │        └─ 能 → 网络正常，问题在应用层（防火墙、监听地址、端口）
```

**问题 2：宿主机上虚拟网卡被禁用**

宿主机的「网络连接」里，如果 `VMware Network Adapter VMnet1` 或 `VMnet8` 被禁用了，仅主机和 NAT 都会失效。在 Windows 上：`控制面板 → 网络和共享中心 → 更改适配器设置`，右键启用。

**问题 3：NAT 模式下宿主机访问不了虚拟机上跑的服务**

NAT 模式下宿主机**可以**访问虚拟机的 IP（因为宿主机持有 VMnet8 的地址）。步骤：

```bash
# 在虚拟机里查 IP
ip a | grep -A2 'ens33\|ens160'
# 假设是 192.168.75.128

# 在本机（宿主机）上直接访问
curl http://192.168.75.128:8080/
```

**如果访问不了**，检查这三项：

```bash
# a) 服务是否监听在 0.0.0.0 而不是 127.0.0.1
ss -lntp | grep 8080
# 127.0.0.1:8080  → 【只监听本地】，宿主机访问不到！
# 0.0.0.0:8080    → 监听所有网卡，可以被访问
# :::8080         → IPv6 所有网卡

# b) 虚拟机里有没有防火墙
sudo ufw status
sudo ufw allow 8080/tcp          # 按需放行

# c) 宿主机上是否有防火墙/杀毒软件拦截
```

**第 a 项是最常见的原因**：开发服务器（Flask 默认、`python -m http.server --bind 127.0.0.1`）常常只绑定本地回环，从外部根本连不上。要让宿主机访问，必须绑定 `0.0.0.0`：

```bash
python3 -m http.server 8000 --bind 0.0.0.0
# Flask: app.run(host='0.0.0.0', port=8000)
```

**问题 4：需要外网主动访问 NAT 后的虚拟机 → 端口转发**

在 VMware 的「编辑 → 虚拟网络编辑器」→ 选中 VMnet8 → 「NAT 设置」→「添加」，配置：

```text
主机端口: 2222        →  虚拟机 IP: 192.168.75.128, 虚拟机端口: 22
```

之后从宿主机（甚至同一物理网络的其他机器，如果宿主机防火墙允许）就能：

```bash
ssh -p 2222 user@<宿主机IP>
```

**静态 IP 与 DHCP 的配合**（一个容易出错的点）：在 NAT 模式下改静态 IP，必须满足两个条件：

1. IP **落在 VMnet8 的网段内**（在虚拟网络编辑器里能看到）；
2. IP **避开 DHCP 地址池**（也在虚拟网络编辑器里能看到范围）。

否则会和别的虚拟机冲突，或者改了之后网络时通时不通。**最省事的做法是继续用 DHCP，然后在 VMware 的 DHCP 设置里按 MAC 地址保留固定 IP**（"静态 DHCP"）。如果一定要手工设静态 IP，Ubuntu 上用 netplan：

```yaml
# /etc/netplan/01-netcfg.yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 192.168.75.50/24
      routes:
        - to: default
          via: 192.168.75.2          # VMware 的虚拟网关
      nameservers:
        addresses: [192.168.75.2, 1.1.1.1]
```

```bash
sudo netplan try            # 【推荐】应用并等待确认，超时自动回滚
# 或
sudo netplan apply
```

**`netplan try` 的价值**：如果你把网络配错了，它会在一段时间后（默认 120 秒）**自动回滚**，避免你把自己锁在虚拟机外面。改网络配置时应该养成先 `try` 的习惯。

## 5. 快照 (snapshot)

### 5.1 快照保存什么

快照保存**某一时刻的虚拟机状态**，包括：

- **磁盘状态**（写时复制：快照之后的新写入存到 delta 文件中，原磁盘保持不变）
- **内存状态**（可选；勾选后恢复快照会回到"当时正在运行的程序和内存内容"）
- 虚拟机的配置

**内存快照 vs 磁盘快照**：

| 类型 | 恢复后 | 说明 |
| --- | --- | --- |
| 只含磁盘 | 相当于"当时关机后的磁盘"，需要重新开机 | 占用小，一致性好 |
| 含内存 | 程序还在运行状态，像"暂停后继续" | 恢复快，但占用大；跨 VMware 版本兼容性差 |

### 5.2 操作路径

在 VMware Workstation 中：

```text
菜单：虚拟机 (VM) → 快照 (Snapshot) → 拍摄快照 (Take Snapshot)
                                     → 快照管理器 (Snapshot Manager)  ← 查看/恢复/删除/克隆
```

在快照管理器里可以：回到某个快照、克隆某个快照为新虚拟机、删除快照（**删除会合并 delta 文件，可能需要几分钟**）。

### 5.3 何时拍快照

| 时机 | 理由 |
| --- | --- |
| **刚装完系统 + 装好 open-vm-tools + 配好网络** | 这是"干净基线"，叫 `00-clean-install`。**最重要的一张** |
| 装好一整套开发工具链（build-essential、Python、git 等）之后 | 叫 `01-dev-ready`，之后做实验坏了可以回到这里 |
| **做危险操作之前** | 改分区、升级内核、改网络配置、玩权限实验、跑漏洞样例 |
| 教学/演示前 | 保证演示过程可重放 |

### 5.4 快照不是什么 —— 必须理解这一点

**快照不是备份 (backup)**。原因：

| 风险 | 说明 |
| --- | --- |
| 依赖父磁盘 | 快照只记录相对于父磁盘的差异。父磁盘损坏，所有快照一起失效 |
| 链式依赖 | 多个快照形成**链**。链越长，每次读写要经过越多 delta 文件，**性能明显下降** |
| 磁盘持续增长 | 每个 delta 文件都会增长，快照留着不用也会一直占空间 |
| 同一物理磁盘 | 快照与虚拟机在同一个磁盘上。这块盘坏了，快照和原盘一起没了 |
| 无版本历史保护 | 误删快照无法恢复 |

**正确的备份**是**完整副本**：用 VMware 的「克隆 (Clone)」功能，或者直接在宿主机上复制整个虚拟机目录（必须先关机），或者用 `tar` 打包虚拟机目录复制到别处。

```bash
# 关机后在宿主机上打包虚拟机（Windows 上可以用 PowerShell 的 Compress-Archive
# 或 7-Zip；Linux 宿主机上直接 tar）
tar -czf ubuntu-vm-backup-$(date +%Y%m%d).tar.gz /path/to/vm-directory/
```

**实践建议**：
- 快照保留 2-3 个，**用完就删**（删除会合并，恢复磁盘性能）；
- 长期保存的状态用克隆，不用快照；
- 定期检查虚拟机目录大小，发现异常增长就去看是不是有陈旧的快照。

```bash
# 虚拟机目录里这些是快照相关文件（.vmdk 后缀带 -000001 之类的是 delta）
ls -lh /path/to/vm/
# ubuntu.vmdk            ← 基础磁盘
# ubuntu-000001.vmdk     ← 快照 delta
# ubuntu-Snapshot1.vmsn  ← 含内存的快照状态
```

### 5.5 克隆 (Clone) 与快照的区别

| | 快照 (Snapshot) | 克隆 (Clone) |
| --- | --- | --- |
| 本质 | 同一虚拟机的历史状态点 | **一台新的独立虚拟机** |
| 独立可用 | ❌ 必须依附原虚拟机 | ✅ 完全独立 |
| 磁盘占用 | 小（只存差异） | 大（完整副本，除非链接克隆） |
| 适合 | 临时回滚 | 做基线镜像、分发给别人、建集群 |

**完整克隆 vs 链接克隆**：

| | 完整克隆 (Full Clone) | 链接克隆 (Linked Clone) |
| --- | --- | --- |
| 独立性 | 完全独立 | 依赖父磁盘（父盘损坏则失效） |
| 占用 | 大（一份完整副本） | 小（只存差异） |
| 性能 | 好 | 略差（读时要查父盘） |
| 适合 | 长期使用、分发 | 临时做多个实验环境 |

## 6. 开发环境配置（Ubuntu/Debian 主线）

### 6.1 换源

**为什么要换**：默认仓库可能在地理上很远，`apt update` 和安装大包会很慢。

**配置文件位置**：

| 文件 | 说明 |
| --- | --- |
| `/etc/apt/sources.list` | 传统主配置 |
| `/etc/apt/sources.list.d/*.list` | 附加源 |
| `/etc/apt/sources.list.d/*.sources` | 较新版 Ubuntu 的 **deb822** 格式 |

**sources.list 一行的字段含义**：

```text
deb  http://archive.ubuntu.com/ubuntu  noble  main restricted universe multiverse
 │   └──────────── URI ─────────────┘   │     └──────── component ────────┘
 │                                       └─ suite = 发行版【代号】
 └─ deb = 二进制包仓库（deb-src 是源码包）
```

**正确的操作顺序**（不能跳步）：

```bash
# 第 1 步：确认自己的发行版代号 —— 这一步错了后面全错
lsb_release -cs
# 或者
. /etc/os-release && echo "$VERSION_CODENAME"
cat /etc/os-release | grep -E '^(VERSION_ID|VERSION_CODENAME)'
```

**必须知道几个真实的代号对应关系**（这样看到代号能反应过来）：

| 发行版 | 代号 |
| --- | --- |
| Ubuntu 20.04 LTS | `focal` |
| Ubuntu 22.04 LTS | `jammy` |
| Ubuntu 24.04 LTS | `noble` |
| Debian 11 | `bullseye` |
| Debian 12 | `bookworm` |

```bash
# 第 2 步：备份（一定要做，这是你的退路）
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
# deb822 格式的
sudo cp /etc/apt/sources.list.d/ubuntu.sources /etc/apt/sources.list.d/ubuntu.sources.bak

# 第 3 步：编辑，只改 URI（把域名换成镜像站），保持 suite 与 component 完全不变
sudo nano /etc/apt/sources.list

# 第 4 步：验证
sudo apt update

# 如果出错，立刻回滚
sudo cp /etc/apt/sources.list.bak /etc/apt/sources.list && sudo apt update
```

**国内常用的公共镜像站**（只列名称，具体地址请到各站官网确认，避免使用过期 URL）：清华大学 TUNA 镜像站、中国科学技术大学 USTC 镜像站、阿里云镜像站、网易 163 镜像站、华为云镜像站。选择依据主要是你的网络环境——用 `curl -o /dev/null -s -w '%{time_total}\n' <地址>` 比较一下响应时间最直观。

> ⚠️ **换源最容易造成的灾难是代号写错**。在 Ubuntu 24.04 (`noble`) 上写了 `jammy`：
> - `apt update` 会**成功**（因为那个仓库确实存在）；
> - 但安装软件时会拉到**为 22.04 编译的包**；
> - 造成库版本冲突、依赖地狱，严重时系统起不来。
>
> 所以：**先 `lsb_release -cs` 确认，再备份，再改，改完立刻 `apt update` 验证**。
>
> **第二条纪律**：不要在 Debian 上用 Ubuntu 的源，反之亦然；也不要混用不同发行版的源。

**deb822 格式示例**（较新 Ubuntu 的默认风格，功能等价于上面的单行格式）：

```text
Types: deb
URIs: http://archive.ubuntu.com/ubuntu
Suites: noble noble-updates noble-backports
Components: main restricted universe multiverse
Signed-By: /usr/share/keyrings/ubuntu-archive-keyring.gpg
```

**换源的其他选择**：

```bash
# 为单次操作指定代理，不改全局配置
sudo apt -o Acquire::http::Proxy="http://proxy:3128" update

# 检查哪个镜像快（比较响应时间）
for u in http://mirror-a/ubuntu/ http://mirror-b/ubuntu/; do
    printf '%s  ' "$u"
    curl -o /dev/null -s -w '%{time_total}s\n' "$u"
done
```

### 6.2 基础工具

```bash
sudo apt update
sudo apt install -y \
    build-essential \
    git curl wget \
    vim tmux htop tree unzip \
    man-db less \
    net-tools iputils-ping dnsutils \
    ca-certificates gnupg
```

**`build-essential` 到底包含什么**（这是很多人不清楚的）：它是一个"元包"，依赖一组编译 C/C++ 程序所需的基本工具：

| 组件 | 作用 |
| --- | --- |
| `gcc` | C 编译器 |
| `g++` | C++ 编译器 |
| `make` | 构建工具 |
| `libc6-dev` | C 标准库的头文件与开发文件 |
| `dpkg-dev` | Debian 打包工具 |
| （以及它们的依赖，如 `binutils`） | 链接器 `ld`、汇编器 `as` 等 |

**注意它不含什么**：不含 `cmake`、不含 `gdb`、不含 `clang`、不含 `pkg-config` 的所有库的开发包。这些要单独装。**"为什么我 `./configure` 报错找不到某个库"——因为库本身装了但它的 `-dev` 包没装**（头文件在 `-dev` 包里）。这是 Debian 系的一个基本规律。

**其他常用工具的作用**：

| 包 | 作用 |
| --- | --- |
| `git` | 版本控制 |
| `curl` / `wget` | 下载文件、测试 HTTP 接口 |
| `vim` | 编辑器（服务器上几乎一定会有 vi） |
| `tmux` | 终端复用（见下面第 6.6 节，非常值得学） |
| `htop` | 交互式进程查看（比 top 好用） |
| `tree` | 以树形显示目录 |
| `unzip` | 解压 zip |
| `net-tools` | `ifconfig`、`netstat`（已被 `ip`、`ss` 取代，但很多教程还在用） |
| `dnsutils` | `dig`、`nslookup` |
| `ca-certificates` | 根证书，`curl https://` 需要 |
| `gnupg` | 验证签名、加密 |

### 6.3 Python 环境

**最重要的一条纪律：不要 `sudo pip install`。**

原因有三：

1. **污染系统 Python**：pip 装的包会混进发行版管理的目录，与 `apt` 管理的包冲突。`apt upgrade` 时可能覆盖、破坏它们；
2. **可能与系统工具冲突**：很多系统工具（如 Ubuntu 的 `apt` 本身、`cloud-init`）依赖特定版本的 Python 库，装错版本可能让系统工具挂掉；
3. **现代系统会直接拒绝**：较新的 Debian/Ubuntu 实现了 **PEP 668**（`externally-managed-environment`），`pip install` 到系统环境会报错并建议你用 venv 或 `--break-system-packages`。**看到这个错误不该去加 `--break-system-packages`**，而应该用 venv。

**正确做法：虚拟环境 (venv)**

```bash
# 1. 确认 Python 3 与 venv 模块
python3 --version
sudo apt install -y python3 python3-pip python3-venv python3-dev

# 2. 为项目创建虚拟环境
cd ~/projects/myproject
python3 -m venv .venv

# 3. 激活
source .venv/bin/activate
# 提示符会变成 (.venv) user@host:~/projects/myproject$

# 4. 在虚拟环境里工作
pip install --upgrade pip
pip install requests pandas
pip freeze > requirements.txt

# 5. 退出
deactivate
```

**venv 的原理**：它创建一个目录（`.venv/`），里面有独立的 `python`、`pip` 可执行文件（实际上是指向系统 Python 的脚本或软链接）和一个独立的 `site-packages` 目录。激活时把 `.venv/bin` 放到 `PATH` 最前面，于是 `python` 和 `pip` 都指向虚拟环境的版本。**系统 Python 完全不受影响。**

**配置 pip 镜像**（在国内网络环境下可能明显更快）：

```bash
# 全局配置（对当前用户）
mkdir -p ~/.config/pip
cat > ~/.config/pip/pip.conf <<'EOF'
[global]
index-url = https://pypi.tuna.tsinghua.edu.cn/simple
timeout = 60
EOF

# 或者每次显式指定
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple requests
```

`<<'EOF'` 用单引号定界符，避免 shell 展开内容里的特殊字符（这里没有，但是个好习惯）。

**`.gitignore` 里一定要加 `.venv/`**——虚拟环境目录体积大且包含绝对路径，绝对不应提交到 git。

**关于 conda / miniconda**：

| | venv + pip | conda |
| --- | --- | --- |
| 体积 | 极小（几十 MB） | 大（几个 GB） |
| 管什么 | Python 包 | Python 包**和**非 Python 的二进制依赖（MKL、CUDA 库、编译器） |
| 适合 | Web 开发、一般脚本 | 数据科学、需要特定 BLAS/CUDA 版本、需要严格复现科研环境 |

如果你的项目涉及科学计算、GPU，conda 的"连二进制依赖一起管"是 venv 做不到的，那时用 conda 是合理的。否则 venv 足够且更轻。**不要混用**：在 conda 环境里又用 pip 装同一个包容易造成环境不一致。

### 6.4 编译器与工具链

```bash
# 验证基础工具链
gcc --version
g++ --version
make --version
ld --version
```

**安装调试与内存排查工具**（与仓库的操作系统板块直接呼应）：

```bash
sudo apt install -y gdb valgrind clang lldb strace ltrace binutils

# 可选：静态分析工具
sudo apt install -y cppcheck
```

| 工具 | 用途 |
| --- | --- |
| `gdb` | 调试器：断点、单步、栈回溯 |
| `valgrind` | 内存错误检测（未初始化读、越界、UAF、double free、泄漏），**不需要重新编译**，但很慢 |
| `clang` / `lldb` | LLVM 工具链（ASan/MSan 在 clang 上支持更全） |
| `strace` | 跟踪系统调用，排查"为什么打不开文件/连不上端口"的利器 |
| `ltrace` | 跟踪库函数调用 |
| `binutils` | `objdump`、`readelf`、`nm`、`strings`（查看二进制结构） |

**推荐的编译选项组合**：

```bash
# 开发调试：优化关掉，调试信息全开，警告全开，sanitizer 打开
gcc -g -O0 -Wall -Wextra -Wpedantic -std=c17 \
    -fsanitize=address,undefined \
    -o app_dbg app.c

# 运行
./app_dbg
# ASan 会在出错时打印详细报告：出错类型、地址、分配与释放的调用栈

# 发布：优化打开，加固选项全开（详见 OS 板块 03-defense-strategies.md）
gcc -O2 -D_FORTIFY_SOURCE=3 -fstack-protector-strong -fPIE -pie \
    -Wl,-z,relro,-z,now -Wall -Wextra -o app app.c

# 验证加固是否真的生效（不要假定）
readelf -l ./app | grep -i relro
readelf -l ./app | grep -A1 GNU_STACK
nm ./app | grep -i stack_chk
```

**`-fsanitize=address` 需要什么**：GCC 或 Clang 的较新版本（`libasan` 会随编译器一起装）。如果编译时报 "unrecognized command-line option"，说明编译器版本太旧：

```bash
# 检查是否支持
gcc -fsanitize=address -x c -o /dev/null - <<< 'int main(){return 0;}'
```

**关于 `-g -O0` 与 `-O2` 的区别在调试时的体现**：`-O0` 下变量与源码一一对应，单步调试符合直觉；`-O2` 下编译器会重排、内联、消除变量，`gdb` 里可能出现"变量被优化掉"（`<optimized out>`）。所以**调试用 `-O0 -g`，性能测试用 `-O2` 且不加 sanitizer**（sanitizer 会明显改变时序）。

### 6.5 SSH 与远程开发

**为什么要从宿主机 SSH 进虚拟机**：可以不用 VMware 的窗口，用宿主机上顺手的终端（Windows Terminal、iTerm、VS Code）操作虚拟机；复制粘贴、字体、快捷键都舒服得多。

**服务端（虚拟机里）**：

```bash
# 1. 安装 SSH 服务端
sudo apt install -y openssh-server

# 2. 启动并设为开机自启
sudo systemctl enable --now ssh
systemctl status ssh

# 3. 查 IP
ip a | grep 'inet '
hostname -I

# 4. 防火墙放行（如果启用了 ufw）
sudo ufw allow ssh        # 或者 sudo ufw allow 22/tcp
sudo ufw status
```

> ⚠️ **Debian/Ubuntu 上的服务名是 `ssh`，不是 `sshd`**。这是从 RHEL 系转过来时非常容易踩的坑：
> ```bash
> sudo systemctl restart ssh      # ✅ Debian/Ubuntu
> sudo systemctl restart sshd     # ❌ 会报找不到单元
> ```
> 而 RHEL/CentOS/Fedora 上恰恰相反，服务名是 `sshd`。配置文件在两者上都是 `/etc/ssh/sshd_config`（服务名与配置文件名不一致，这确实容易混淆）。

**客户端（宿主机）**：

```bash
# 用密码登录
ssh user@192.168.75.128

# 指定端口（NAT 端口转发到 2222 时）
ssh -p 2222 user@127.0.0.1

# 常用选项
ssh -v user@host              # 详细输出，排查连接失败（-vvv 更详细）
ssh -o ConnectTimeout=5 user@host
```

**配置密钥登录（更安全也更方便）**：

```bash
# 1. 在【宿主机】上生成密钥对（ed25519 是现代推荐算法）
ssh-keygen -t ed25519 -C "my-vm-key"
# 一路回车（密码短语可以留空以便免密，或设一个更安全）
# 生成 ~/.ssh/id_ed25519（私钥）和 ~/.ssh/id_ed25519.pub（公钥）

# 2. 把公钥复制到虚拟机
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@192.168.75.128
# 如果 ssh-copy-id 不可用（Windows 上可能没有），手工做：
cat ~/.ssh/id_ed25519.pub | ssh user@192.168.75.128 \
    "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"

# 3. 配置 ~/.ssh/config 简化登录（宿主机上）
```

```text
# ~/.ssh/config
Host vm
    HostName 192.168.75.128
    User alice
    IdentityFile ~/.ssh/id_ed25519
    ServerAliveInterval 60
```

之后只需 `ssh vm`。

> ⚠️ **关于 SSH 私钥权限的一个重要事实**：`ssh` 会检查私钥文件的权限，**如果其他用户可读它会拒绝使用并报错**（"Permissions 0644 for 'id_ed25519' are too open"）。正确权限：
> ```bash
> chmod 700 ~/.ssh
> chmod 600 ~/.ssh/id_ed25519        # 私钥 600
> chmod 644 ~/.ssh/id_ed25519.pub    # 公钥 644
> chmod 600 ~/.ssh/authorized_keys   # 服务端也是 600
> ```
> 这是"权限太松反而不能用"的真实例子（详见 [04-permissions-and-users.md](04-permissions-and-users.md)）。

**加固服务端配置**（`/etc/ssh/sshd_config`）：

```text
PermitRootLogin no                    # 禁止 root 直接登录
PasswordAuthentication no             # 只允许密钥登录（确认密钥能用了再改！）
PubkeyAuthentication yes
Port 2222                             # 改默认端口（减少扫描噪音，但不是安全措施）
AllowUsers alice                      # 白名单
X11Forwarding no
MaxAuthTries 3
```

**改完必须验证再断开**：

```bash
sudo sshd -t                      # 【关键】语法检查，不通过就别重启
sudo systemctl restart ssh
# 然后【另开一个窗口】测试新连接是否成功，确认无误后再关掉当前连接
```

> ⚠️ **`PasswordAuthentication no` 是一个高危改动**：如果密钥还没配好就改了这个，你会**把自己锁在虚拟机外面**（只能通过 VMware 的控制台进去修）。**永远先确认密钥登录成功，再关闭密码登录**。`sudo sshd -t` 做语法检查也是同样的道理——配置语法错误会导致 sshd 起不来。

**VS Code Remote-SSH** 是常见的开发工作流：在宿主机上装 VS Code + Remote-SSH 扩展，直接连到虚拟机的 IP，编辑与调试都在虚拟机里进行，界面在宿主机上。它需要一个可用的 SSH 连接（上面的配置就是前提）。

### 6.6 tmux 为什么值得学

在 ssh 会话里跑长任务时，一旦网络抖动或你关掉终端，进程就会收到 `SIGHUP` 而死（详见 [02-process-and-jobs.md](02-process-and-jobs.md)）。`tmux` 解决这个问题，而且能重新接管会话。

```bash
sudo apt install -y tmux

tmux new -s work              # 新建名为 work 的会话
# 在会话里跑你的长任务

# 断开（detach）：按 Ctrl+B 然后按 D
# 会话继续在后台运行

tmux ls                       # 列出所有会话
tmux attach -t work           # 重新接管
tmux attach                   # 只有一个会话时可以省略名字

# 常用快捷键（前缀键默认是 Ctrl+B）
# Ctrl+B  D      断开
# Ctrl+B  C      新建窗口
# Ctrl+B  N/P    下一个/上一个窗口
# Ctrl+B  %      垂直分割窗格
# Ctrl+B  "      水平分割窗格
# Ctrl+B  方向键  在窗格间切换
```

**为什么它比 `nohup` 好**：`nohup` 只能让进程活着，你**看不到它的实时输出、也不能在它跑完后回到那个 shell 环境**。`tmux` 让会话完整保留（包括整个 shell 状态、历史、运行中的前台程序），随时可以接管。**这是远程工作最值得先学的一个工具。**

### 6.7 共享文件夹与文件传输

**路线一：open-vm-tools 的共享文件夹**

在 VMware 里「虚拟机设置 → 选项 → 共享文件夹」启用并添加目录，然后在客户机里：

```bash
# 检查是否已经自动挂载
ls /mnt/hgfs/

# 如果没有，手工挂载
sudo mkdir -p /mnt/hgfs
sudo vmhgfs-fuse .host:/ /mnt/hgfs -o allow_other -o uid=$(id -u) -o gid=$(id -g)

# 开机自动挂载：写入 /etc/fstab
# .host:/  /mnt/hgfs  fuse.vmhgfs-fuse  allow_other,defaults  0 0
```

**共享文件夹的常见坑**：

| 坑 | 说明 |
| --- | --- |
| 权限问题 | 挂载后属主可能是 root 或映射错误，需要用 `uid=`/`gid=` 选项 |
| 性能差 | 通过 VMware 的共享机制访问文件比本地慢不少，**不适合当构建目录**（`npm install`、编译大型项目会非常慢） |
| 符号链接 | 共享文件夹对符号链接支持有限，很多工具（npm、pip、git）会用到 |
| `vmhgfs-fuse` 偶发断连 | 需要重新挂载 |

**我的建议**：共享文件夹适合"临时丢个文件进去"，**不适合当开发目录**。开发应该在虚拟机内部的本地目录里做，用下面两种方式同步代码。

**路线二：SCP / rsync（推荐）**

```bash
# 从宿主机拷贝到虚拟机
scp file.txt user@192.168.75.128:/home/user/

# 递归拷贝目录
scp -r ./project user@192.168.75.128:/home/user/

# 从虚拟机拷到宿主机
scp user@192.168.75.128:/var/log/app.log ./

# rsync：增量同步，只传变化的文件（更适合反复同步）
rsync -avz --progress ./project/ user@192.168.75.128:/home/user/project/
rsync -avz --delete ./project/ user@192.168.75.128:/home/user/project/   # 删除目标端多余文件（小心！）
```

**`rsync` 的关键细节**：源路径末尾的 `/` 有实际意义。

```bash
rsync -av src/  dst/     # 把 src 的【内容】同步到 dst  （结果是 dst/file.txt）
rsync -av src   dst/     # 把 src 这个【目录】同步到 dst （结果是 dst/src/file.txt）
```

这个差异极其容易搞错，动手前先 `--dry-run`（`rsync -avn`）看一遍。

**路线三：Git**

如果代码本来就在 Git 仓库里，那就直接 `git clone` / `git push` / `git pull`，不需要文件传输工具。**这也是最推荐的路线**——它顺便解决了版本管理。

### 6.8 常见坑清单

| 问题 | 原因与解决 |
| --- | --- |
| **中文输入法打不出字** | Ubuntu 桌面需要在「设置 → 区域与语言 → 输入源」添加中文（如"智能拼音"）；可能需要装 `ibus-libpinyin` 或 `fcitx5` 并重启会话 |
| **时区不对** | `timedatectl set-timezone Asia/Shanghai`；检查 `timedatectl` 输出的 NTP 同步状态 |
| **分辨率不能自适应** | 装 `open-vm-tools-desktop`（不是 `open-vm-tools`）并重启；或手动 `xrandr` |
| **复制粘贴失效** | 同上，需要 `open-vm-tools-desktop`；也可以在 VMware 设置里检查「隔离」选项是否勾选 |
| **时间不同步（越来越快/慢）** | 装 open-vm-tools 后由它同步；也可 `sudo systemctl enable --now systemd-timesyncd` |
| **磁盘越用越大，删了文件也不变小** | 虚拟磁盘是"只增不减"的。需要在客户机里清理后，用 VMware 的「虚拟机 → 设置 → 硬盘 → 压缩 (Compact)」（需先清理空闲空间：新建一个填满全零的大文件再删掉，即"零填充"，然后关机压缩） |
| **宿主机休眠后虚拟机网络断掉** | 虚拟网卡在宿主机恢复后可能需要重连；重启 VMware 的 NAT/DHCP 服务，或在虚拟机里 `sudo nmcli connection up <名字>` |
| **虚拟机里时间跳变导致构建工具报错** | 快照恢复（尤其含内存快照）后系统时间可能与实际不同步，等 NTP 校准或手动 `sudo systemctl restart systemd-timesyncd` |
| **开机进入 emergency mode** | 常见于 `/etc/fstab` 写错了挂载项（例如共享文件夹那行）导致挂载失败。在 emergency shell 里注释掉出错的行再重启 |
| **`apt` 提示 `Could not get lock`** | 见 [05-package-and-service.md](05-package-and-service.md) 第 2.5 节——**等**，不要删锁文件 |
| **鼠标被"吸"在虚拟机里出不来** | 按 `Ctrl+Alt` 释放（VMware 的默认热键） |
| **虚拟机窗口里的系统特别卡** | 检查是否给足了内存、是否开了 3D 加速、宿主机是否在跑 Hyper-V/WSL2 争抢资源；关掉桌面特效（用 XFCE 或直接跑无桌面服务器版） |

**关于"磁盘压缩"补充一句**：虚拟磁盘的增长机制是"分配过的块不会因为文件删除而归还"。所以一个用满过 35 GB 的 40 GB 磁盘，即使现在只用 5 GB，文件仍然是将近 40 GB。要缩小：在客户机里零填充空闲空间（`dd if=/dev/zero of=zero.fill bs=1M || true; rm zero.fill`，或者用 `zerofree` 工具），关机后在 VMware 里执行压缩/清理。**这也提醒我们：选磁盘大小时要给够余量，但不要无脑给 200 GB 然后指望它能缩回来。**

## 7. 一个完整的搭建顺序（可直接照做）

```text
1. 前置检查
   - BIOS 里开 VT-x / AMD-V
   - Windows 上确认 Hyper-V/WSL2/VBS 状态，决定方案 A 还是 B
   - 宿主机留出 60 GB 空间与足够内存

2. 新建虚拟机
   - 下载 Ubuntu LTS ISO（到官网下载，验证校验和）
   - VMware「新建虚拟机」，选典型，指定 ISO
   - 客户机类型 Linux / Ubuntu 64-bit
   - 磁盘 40 GB，拆分多文件
   - 内存 4-8 GB，CPU 2-4 核，网络选 NAT

3. 安装系统
   - 语言/键盘（注意选 English (US)）
   - 磁盘用默认（擦除磁盘并安装）
   - 设置用户名（纯字母小写）、强密码、主机名
   - 等待安装完成并重启

4. 装增强工具
   sudo apt update && sudo apt install -y open-vm-tools open-vm-tools-desktop
   sudo reboot
   → 验证分辨率自适应与复制粘贴

5. 【拍第一张快照】名字：00-clean-install

6. 基础配置
   sudo timedatectl set-timezone Asia/Shanghai
   → 换源（先 lsb_release -cs 确认代号，先备份，改完 apt update 验证）
   → 【拍快照】名字：01-apt-configured

7. 装工具链
   sudo apt install -y build-essential git curl wget vim tmux htop tree \
                       python3 python3-pip python3-venv python3-dev \
                       gdb valgrind clang strace man-db
   → 验证：gcc --version / python3 --version / git --version
   → 【拍快照】名字：02-dev-ready  ← 这是做实验前的标准回滚点

8. 配 SSH（方便以后用终端操作）
   sudo apt install -y openssh-server
   sudo systemctl enable --now ssh
   → 在宿主机上生成密钥、ssh-copy-id
   → 验证密钥登录成功后，再考虑关闭密码登录

9. 验证开发流程
   - 写个 hello.c，用 gcc -Wall -Wextra 编译运行
   - 写个有内存错误的程序，用 -fsanitize=address 看报告
   - 建个 Python venv，装个包，确认系统 Python 没被污染
   - 从宿主机 ssh 进来，跑 tmux，detach 再 attach

10. 拍一张最终快照：03-workflow-verified
    并且【导出一份完整备份】（克隆或复制目录），因为快照不是备份
```

## 小结

- 虚拟机最大的价值是**快照回滚**与**隔离**；代价是性能、磁盘占用与内存竞争。系统级实验用虚拟机，只学命令用 WSL2 更轻。
- 必须在 BIOS 开 **VT-x / AMD-V**；Windows 上 **Hyper-V/WSL2/VBS 与 VMware 存在虚拟化资源争抢**，先查 `HypervisorPresent`，再决定"用 WHP 共存"还是"关掉 Hyper-V"。
- 资源分配原则：CPU 不超过**物理**核心数，内存不超过宿主机的一半，让宿主机始终不 swap。
- **VMware 官方的 VMware Tools 优先用发行版仓库里的 `open-vm-tools`**；要剪贴板共享与分辨率自适应必须装 `open-vm-tools-desktop`。
- 三种网络模式：**桥接**（虚拟机进入物理网络，可被外部访问，但受外部网络策略限制）、**NAT**（经宿主机上网，最省事，适合日常）、**仅主机**（完全隔离，适合纯内网实验）。需要"能上网 + 能被访问"就加两块网卡。
- VMnet1 通常是 host-only、VMnet8 通常是 NAT，**但这些都是默认值可以改——永远用 `ip a` 查实际 IP，不要在脚本里硬编码**。
- 网络排查按分层走：有没有 IP → 能不能 ping 通网关 → 能不能 ping 通外网 IP → 能不能解析域名。**IP 通但域名不通就是 DNS 问题。**
- NAT 下宿主机访问虚拟机服务失败，最常见的原因是**服务只监听 `127.0.0.1` 而不是 `0.0.0.0`**（用 `ss -lntp` 确认），其次是防火墙。
- 改 netplan 网络配置用 **`sudo netplan try`**，配错会自动回滚，避免把自己锁在外面。
- **快照不是备份**：它依赖父磁盘、链越长性能越差、留久了也会持续占空间。长期保存状态要用**克隆**或完整复制目录。
- 快照的最佳用法：**装完系统拍一张干净基线**，做危险操作前拍一张，用完删掉。
- 换源的核心风险是**代号写错**（在 noble 上写 jammy 会拉到为 22.04 编译的包）。流程必须是：`lsb_release -cs` 确认 → 备份 → 只改 URI → `apt update` 验证 → 有错立刻回滚。**不要跨发行版混用源。**
- `build-essential` 包含 gcc/g++/make/libc6-dev 等，但**不含 cmake、gdb、clang**；"找不到某个库"通常是缺它的 `-dev` 包。
- **绝不要 `sudo pip install`**：会污染系统 Python 并与 apt 冲突，现代系统还会用 PEP 668 直接拒绝。正确做法是 `python3 -m venv .venv` 并激活。看到 `externally-managed-environment` 错误**不要加 `--break-system-packages`**。
- 调试编译用 `-g -O0 -Wall -Wextra -fsanitize=address,undefined`；发布编译用 `-O2 -D_FORTIFY_SOURCE=3 -fstack-protector-strong -fPIE -pie -Wl,-z,relro,-z,now`。两者用途不同，不要混。
- **Debian/Ubuntu 上 SSH 服务名是 `ssh`，RHEL 系是 `sshd`**；配置文件在两者上都是 `/etc/ssh/sshd_config`。改配置前先 `sudo sshd -t` 做语法检查，改完后**另开窗口验证**再断开。
- SSH **私钥权限必须是 `600`**，太开放 `ssh` 会拒绝使用；关闭 `PasswordAuthentication` 前必须确认密钥登录已成功，否则会把自己锁在外面。
- **`tmux` 比 `nohup` 更适合远程长任务**：会话可以随时重新接管，这是远程工作最值得先学的工具。
- 共享文件夹适合临时传文件，**性能差且有符号链接问题，不要当构建目录或开发目录**。用 scp/rsync/git 更可靠；`rsync` 源路径末尾的 `/` 会改变语义，动手前先 `-n` 演练。
- 虚拟机磁盘"只增不减"，需要零填充 + 关机压缩才能缩小；一开始就别无脑给 200 GB。
