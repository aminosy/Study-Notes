# Shell 脚本基础：变量、条件、循环与函数

本篇梳理 Bash 脚本的核心语法：变量与引号、条件判断、循环、函数、数组、重定向技巧，并给出四个完整可用的实用脚本与调试方法。

## 1. 脚本骨架

### 1.1 shebang 与执行方式

```bash
#!/bin/bash
# 这是注释

echo "hello"
```

**第一行的 `#!/bin/bash` 叫 shebang**（`#!` + 解释器路径）。内核在执行一个"带执行位的文本文件"时，读取前两个字节，若是 `#!`，就用后面指定的解释器来运行它。

**可移植的写法**：

```bash
#!/usr/bin/env bash
```

`env` 会在 `PATH` 里查找 `bash`，因此在 bash 装在 `/usr/local/bin/bash`（例如 macOS 上用 Homebrew 装的）的系统上也能工作。**写要分享给他人的脚本时推荐这种形式。**

**两种运行方式的区别**（这个区别很关键）：

```bash
./script.sh          # 需要：【可执行位】+【正确的 shebang】
                     # 内核读取 shebang，用它指定的解释器运行

bash script.sh       # 不需要可执行位，【忽略 shebang】
                     # 明确用 bash 运行；即使文件里写 #!/bin/sh，也是 bash 在跑

sh script.sh         # 用 sh 运行（可能是 dash 等更严格的 POSIX shell）
source script.sh     # 或 . script.sh —— 在【当前 shell】里执行，会污染当前环境
```

第 4 种（`source`）在配置文件里是必要的（`~/.bashrc`），但**用它跑普通脚本要小心**：脚本里的 `cd`、变量赋值、`set -e` 都会影响你当前的 shell，脚本出错可能把你的终端搞坏。

### 1.2 退出码与 `exit`

```bash
echo "done"
exit 0        # 显式返回成功
exit 1        # 返回失败（一般约定 0 = 成功，非 0 = 失败）
```

- 脚本的退出码是**最后一条命令的退出码**（如果没有显式 `exit`）；
- `$?` 保存上一条命令的退出码，**必须紧接着使用**（任何命令都会覆盖它）：

```bash
grep -q "FAIL" report.txt
status=$?                     # 立刻保存！
echo "grep returned $status"
```

### 1.3 `set -euo pipefail` —— 推荐的开头

```bash
#!/usr/bin/env bash
set -euo pipefail
```

| 选项 | 名称 | 作用 |
| --- | --- | --- |
| `-e` | errexit | 命令返回非 0 时**立即退出**脚本 |
| `-u` | nounset | 引用**未定义变量**时报错退出（否则展开成空字符串，静默出错） |
| `-o pipefail` | pipefail | 管道中**任何一个**命令失败，整个管道的退出码就是非 0（默认只看最后一个命令） |
| `-x` | xtrace | 打印每条执行的命令（调试用，见第 8 节） |

**为什么 `pipefail` 重要**：默认情况下 `a | b` 的退出码只取 `b` 的。所以：

```bash
# 没有 pipefail 时的经典陷阱
set -e
false | true          # 管道退出码是 true 的 0
echo "还是会执行到这里"  # 前面的 false 被静默忽略了
```

**`set -e` 的坑（必须知道，否则会被它坑）**：

```bash
# 坑 1：命令在 if 条件里时不触发 -e（这是设计好的，通常是你想要的）
set -e
if grep -q pattern file; then echo "found"; fi   # grep 返回 1 不会退出

# 坑 2：命令在 && / || 左侧时，-e 不生效
false && echo "no"        # 不退出

# 坑 3：命令被 ! 取反时不生效
! false                   # 不退出

# 坑 4：函数内部的失败，如果函数在条件里调用，整条链都不触发
set -e
check() { false; echo "still here"; }
check || true             # 因为被 || 保护，check 内部的 false 不会退出脚本！

# 坑 5：命令替换里的失败不一定能传播
set -e
x=$(false)                # 在 bash 里这会触发 -e（赋值语句的退出码是命令替换的退出码）
x=$(false; echo "")       # 这样就不会了，因为最后一条 echo 成功了
```

**实践建议**：`set -euo pipefail` 是好的默认值，但**不要以为它保证了一切**。关键步骤要显式检查：

```bash
if ! cp important.txt /backup/; then
    echo "备份失败" >&2
    exit 1
fi
```

需要临时关掉 `-e` 时：

```bash
set +e
risky_command
rc=$?
set -e
if [ "$rc" -ne 0 ]; then
    echo "risky_command 失败: $rc"
fi
```

## 2. 变量

### 2.1 赋值与引用

```bash
name="alice"            # 赋值：等号两边【不能有空格】！
name = "alice"          # ❌ 错误：这会被解析成"执行名为 name 的命令，参数是 = 和 alice"

echo "$name"            # 引用：推荐加双引号
echo "${name}"          # 花括号形式，用于与后续字符区分
echo "${name}_backup"   # 不加花括号会变成 $name_backup（不存在的变量）
echo "$name_backup"     # ❌ 展开的是 name_backup 这个变量（通常为空）
```

**为什么赋值不能有空格**：shell 的命令行语法是"命令 + 参数"，空格是分隔符。`name = "alice"` 会被解析为"执行 `name` 这个命令，第一个参数是 `=`，第二个是 `alice`"。变量赋值是 shell 语法的特殊情况，必须写成 `name=value` 且中间无空格。

**为什么引用时要加引号**（这是 shell 脚本最重要的一个习惯）：

```bash
file="my document.txt"

echo $file              # 不加引号 → 词分隔 → 输出: my document.txt（看起来没事）
rm $file                # ❌ 实际的参数是 "my" 和 "document.txt" 两个！灾难
rm "$file"              # ✅ 一个参数

files="a.txt b.txt"
for f in $files; do ... ; done    # 不加引号 → 会正确拆成两个单词（这可能是想要的）
for f in "$files"; do ... ; done  # 加引号 → 只有一个元素，通常是错的
```

**引号的三种形式**（必须区分清楚）：

| 形式 | 变量展开 | 命令替换 | 通配符展开 | 反斜杠转义 |
| --- | --- | --- | --- | --- |
| `'单引号'` | ❌ 不展开 | ❌ | ❌ 不展开 | ❌ 全部字面量 |
| `"双引号"` | ✅ 展开 | ✅ | ❌ 不展开 | ✅ 部分（`\$` `\"` `\\`） |
| 不加引号 | ✅ 展开 | ✅ | ✅ **会展开** | ✅ 部分 |

```bash
echo '$HOME'      # 输出字面量: $HOME
echo "$HOME"      # 输出: /home/alice
echo $HOME        # 同上，但没有通配符保护
echo "*.txt"      # 输出: *.txt（字面量）
echo *.txt        # 输出: a.txt b.txt（被 shell 展开成匹配的文件名）
```

**核心规则**：**不确定时就加双引号**。这一条能避免绝大部分脚本 bug。

### 2.2 默认值、长度、字符串操作

```bash
# 默认值
echo "${VAR:-default}"      # VAR 未设置或为空 → 用 default（不修改 VAR）
echo "${VAR:=default}"      # 同上，但【同时赋值给 VAR】
echo "${VAR:?必须设置 VAR}"  # VAR 未设置 → 报错并退出（配合 set -u 很有用）
echo "${VAR:+设置了}"        # VAR 已设置则输出"设置了"，否则输出空

# 长度
echo "${#VAR}"              # 字符串长度（字符数）
arr=(a b c)
echo "${#arr[@]}"           # 数组元素个数

# 截取
echo "${VAR:0:3}"           # 从位置 0 取 3 个字符
echo "${VAR:3}"             # 从位置 3 到末尾
echo "${VAR: -3}"           # 最后 3 个字符（注意空格，否则 : - 会被误解析）

# 删除前后缀（非常常用）
path="/var/log/nginx/access.log"
echo "${path##*/}"          # 从前面删掉最长的 */ → access.log   （取文件名）
echo "${path#*/}"           # 从前面删掉最短的 */ → var/log/nginx/access.log
echo "${path%/*}"           # 从后面删掉最短的 /* → /var/log/nginx   （取目录）
file="archive.tar.gz"
echo "${file%.*}"           # 删掉最后一个 .* → archive.tar
echo "${file%%.*}"          # 删掉最长的 .* → archive
echo "${file##*.}"          # 取扩展名 → gz

# 替换
echo "${VAR/old/new}"       # 替换第一处
echo "${VAR//old/new}"      # 替换所有
echo "${VAR/#prefix/new}"   # 只替换开头的 prefix
echo "${VAR/%suffix/new}"   # 只替换结尾的 suffix

# 大小写（bash 4+）
echo "${VAR,,}"             # 全小写
echo "${VAR^^}"             # 全大写
```

**`##` 与 `#`、`%%` 与 `%` 的记忆方法**：符号的长度表示删除的长度——`##`/`%%` 删**长**（最长匹配），`#`/`%` 删**短**（最短匹配）；`#` 从**左**（前缀）删，`%` 从**右**（后缀）删（`%` 的形状像是行尾）。

**记忆口诀**：`#` 在键盘左边 → 处理左/前缀；`%` 在键盘右边 → 处理右/后缀。

### 2.3 命令替换与算术

```bash
# 命令替换：用 $() ，不要用反引号
today=$(date +%F)               # ✅ 推荐
today=`date +%F`                # ⚠️ 能用，但难以嵌套、反斜杠处理不同

# 嵌套时 $() 的优势很明显
count=$(grep -c "^" "$(find /tmp -name '*.log' | head -1)")

# 算术
n=5
echo $((n + 3))                 # 8
echo $((n * 2))                 # 10
echo $((n / 2))                 # 2（整数除法！）
((n++))                         # n 变成 6
((n += 10))                     # n 变成 16
echo $(( 10 % 3 ))              # 1

# 浮点：bash 不支持，必须用 bc 或 awk
echo "scale=2; 10/3" | bc       # 3.33
awk 'BEGIN{printf "%.2f\n", 10/3}'   # 3.33
```

> ⚠️ **bash 的算术只支持整数**。`echo $((10/3))` 得到 `2`。需要小数必须用 `bc` 或 `awk`。这是刚写脚本时最容易踩的坑之一。

### 2.4 特殊变量

| 变量 | 含义 |
| --- | --- |
| `$0` | 脚本自身的名字（或 shell 名） |
| `$1` `$2` ... `${10}` | 第 1、2... 个位置参数（**第 10 个起必须用花括号**） |
| `$#` | 位置参数的**个数** |
| `$@` | 所有位置参数 |
| `$*` | 所有位置参数（作为**一个**字符串） |
| `$?` | 上一条命令的退出码 |
| `$$` | 当前 shell 的 PID |
| `$!` | 最近一个后台进程的 PID |
| `$_` | 上一条命令的最后一个参数 |
| `$LINENO` | 当前行号（调试用） |
| `$FUNCNAME` | 当前函数名数组 |

**`"$@"` 与 `"$*"` 的区别极其重要**：

```bash
# 假设脚本被这样调用: ./s.sh "a b" c
echo "$#"
# 2    （两个参数）

# "$@" 展开成两个独立的参数（正确转发）
for arg in "$@"; do echo "[$arg]"; done
# [a b]
# [c]

# "$*" 展开成一个字符串，用 IFS 的第一个字符连接
for arg in "$*"; do echo "[$arg]"; done
# [a b c]

# 不加引号的 $@ 或 $* 都会做词分隔
for arg in $@; do echo "[$arg]"; done
# [a]
# [b]
# [c]     ← 错误！"a b" 被拆开了
```

**结论：转发参数给另一个命令时，永远用 `"$@"`。**

```bash
#!/usr/bin/env bash
# 正确的包装脚本
exec mycommand --verbose "$@"
```

`exec` 用新进程替换当前 shell，不产生额外的进程层级，信号也能正确传递。

## 3. 条件判断

### 3.1 `[ ]` 与 `[[ ]]`

| | `[ ]`（test 命令） | `[[ ]]`（bash 关键字） |
| --- | --- | --- |
| 是什么 | 一个**命令**（`/usr/bin/[` 或 shell 内建） | bash/ksh/zsh 的**语法关键字** |
| POSIX 兼容 | ✅ 是 | ❌ 不是（`sh`/`dash` 不支持） |
| 词分隔与通配符 | **会**发生（所以变量必须加引号） | **不发生**（可以不加引号） |
| `&&` / `\|\|` | 不能在内部用，要用 `-a` / `-o`（已废弃） | ✅ 可以直接用 |
| 正则匹配 | 不支持 | ✅ `=~` |
| 模式匹配 | 不支持 | ✅ `==` 右侧可用通配符 |

```bash
# [ ] 中变量必须加引号，否则空变量会导致语法错误
[ $x = "y" ]        # x 为空时变成 [ = "y" ] → 报错 "unary operator expected"
[ "$x" = "y" ]      # ✅ 安全

# [[ ]] 中不加引号也安全（但加引号仍是好习惯）
[[ $x = "y" ]]      # ✅
[[ $x == y* ]]      # ✅ 模式匹配（右侧不加引号）
[[ $x =~ ^[0-9]+$ ]]  # ✅ 正则匹配
```

**推荐：bash 脚本里用 `[[ ]]`**（更安全、功能更多）。只有需要 POSIX 兼容（例如目标是 `/bin/sh` 的脚本）时才用 `[ ]`，并且那时**必须给变量加引号**。

### 3.2 常用测试运算符

**字符串**：

| 运算符 | 含义 |
| --- | --- |
| `-z "$s"` | 长度为**零**（zero） |
| `-n "$s"` | 长度**非零**（non-zero） |
| `"$a" = "$b"` | 相等（`==` 在 `[[ ]]` 里也可用，`=` 更可移植） |
| `"$a" != "$b"` | 不等 |
| `"$a" < "$b"` | 字典序小于（在 `[ ]` 里必须转义为 `\<`） |

```bash
if [ -z "$name" ]; then
    echo "name 为空"
fi

if [[ -n "$name" && "$name" != "root" ]]; then
    echo "name 有效且不是 root"
fi
```

**整数**（注意：整数比较**必须用这些**，不能用 `<` `>`）：

| 运算符 | 含义 |
| --- | --- |
| `-eq` | 等于 (equal) |
| `-ne` | 不等于 (not equal) |
| `-lt` | 小于 (less than) |
| `-le` | 小于等于 |
| `-gt` | 大于 (greater than) |
| `-ge` | 大于等于 |

```bash
if [ "$count" -gt 100 ]; then
    echo "超过阈值"
fi
```

> ⚠️ **`[ "$a" > "$b" ]` 是错的**：`>` 在 `[ ]` 里会被 shell 解释成**重定向**，结果是把文件 `$b` 创建出来，而且判断结果永远为真。这是最经典的 shell 陷阱之一。数字比较用 `-gt`，字符串比较在 `[[ ]]` 里用 `>`（不需要转义）。

**文件**：

| 运算符 | 含义 |
| --- | --- |
| `-e file` | 存在 (exists) |
| `-f file` | 存在且是普通文件 |
| `-d file` | 存在且是目录 |
| `-L file` / `-h file` | 是符号链接 |
| `-r` / `-w` / `-x` | 当前用户可读 / 可写 / 可执行 |
| `-s file` | 存在且**大小大于 0**（常用于检查文件非空） |
| `-nt` / `-ot` | newer than / older than（按 mtime 比较） |
| `file1 -ef file2` | 是同一个 inode（同一文件） |

```bash
if [ -f /etc/nginx/nginx.conf ]; then
    echo "配置文件存在"
fi

if [ ! -d /var/log/myapp ]; then
    mkdir -p /var/log/myapp
fi

# 检查命令是否存在（脚本里的标准做法）
if ! command -v jq >/dev/null 2>&1; then
    echo "需要安装 jq" >&2
    exit 1
fi
```

`command -v` 比 `which` 更可移植（`which` 在有些系统上不是内建命令，行为不一致），是检查依赖的推荐方式。

### 3.3 `if` / `elif` / `else`

```bash
#!/usr/bin/env bash
set -euo pipefail

usage() {
    echo "用法: $0 <环境> [--dry-run]"
    echo "  环境: dev | staging | prod"
    exit 1
}

env_name="${1:-}"

if [[ -z "$env_name" ]]; then
    usage
elif [[ "$env_name" == "prod" ]]; then
    echo "⚠️  生产环境，需要二次确认"
    read -r -p "确认部署到生产? [y/N] " answer
    [[ "$answer" =~ ^[Yy]$ ]] || { echo "已取消"; exit 1; }
elif [[ "$env_name" =~ ^(dev|staging)$ ]]; then
    echo "部署到 $env_name"
else
    echo "未知环境: $env_name" >&2
    usage
fi
```

注意 `usage()` 用 `exit 1` 而不是 `return`——因为它是在脚本顶层被调用的，`exit` 才能终止脚本。如果它被别处调用后还想继续，就应该用 `return 1` 或直接 `echo >&2` 由调用者决定。

### 3.4 `case`

`case` 适合"一个变量对多个固定模式"的分支，比一长串 `elif` 清晰得多。

```bash
#!/usr/bin/env bash
set -euo pipefail

case "${1:-}" in
    start)
        echo "启动服务..."
        systemctl start myapp
        ;;
    stop)
        echo "停止服务..."
        systemctl stop myapp
        ;;
    restart|reload)                    # 用 | 表示"或"
        echo "重载服务..."
        systemctl reload-or-restart myapp
        ;;
    status)
        systemctl status myapp --no-pager
        ;;
    *.log)                             # 支持通配符模式
        echo "这是一个日志文件名: $1"
        ;;
    "")                                # 空参数
        echo "错误: 缺少参数" >&2
        exit 1
        ;;
    *)                                 # 兜底（相当于 default）
        echo "未知命令: $1" >&2
        echo "可用命令: start | stop | restart | status" >&2
        exit 1
        ;;
esac
```

**`case` 的几个语法要点**：每个分支以 `;;` 结束；`*)` 是兜底分支（应当总有一个）；模式里可以用 `*` `?` `[...]`；`esac` 是 `case` 反过来写。

## 4. 循环

### 4.1 `for`

```bash
# 列表遍历
for fruit in apple banana cherry; do
    echo "$fruit"
done

# C 风格（bash 特性）
for ((i = 1; i <= 5; i++)); do
    echo "第 $i 次"
done

# 范围（bash 特性）
for i in {1..10}; do echo "$i"; done
for i in {1..10..2}; do echo "$i"; done      # 步长 2
for i in $(seq 1 10); do echo "$i"; done     # seq（可移植）

# 通配符（注意：没有匹配时默认会展开成字面量）
for f in *.log; do
    echo "处理 $f"
done
```

**通配符不匹配的陷阱**：

```bash
# 目录里没有 .log 文件时
for f in *.log; do
    echo "$f"        # 输出: *.log  ← 字面量，不是文件名！
done
```

三种处理方式：

```bash
# 方式 1：设 nullglob，不匹配时展开成空（推荐）
shopt -s nullglob
for f in *.log; do echo "$f"; done

# 方式 2：先判断
for f in *.log; do
    [[ -e "$f" ]] || continue
    echo "$f"
done

# 方式 3：用 find（更可控，且能递归）
while IFS= read -r -d '' f; do
    echo "$f"
done < <(find . -name '*.log' -print0)
```

**`for f in $(ls)` 为什么是错的**：`$(ls)` 会被词分隔和通配符展开，含空格的文件名会被拆开。直接用 `for f in *` 或 `find` 更安全。

### 4.2 `while`

```bash
# 计数器
i=0
while [ "$i" -lt 5 ]; do
    echo "$i"
    ((i++))
done

# 【读取文件的标准写法】—— 注意重定向的位置
while IFS= read -r line; do
    echo "处理: $line"
done < input.txt
```

**`IFS= read -r line` 的每个部分都有用**：

| 部分 | 作用 |
| --- | --- |
| `IFS=` | 清空字段分隔符，**保留行首行尾的空白**（否则会被裁掉） |
| `read -r` | 禁止反斜杠转义（否则 `\t` 会被当转义序列吃掉） |
| `line` | 变量名 |
| `done < input.txt` | 把文件接到 **while 循环整体** 的 stdin |

**为什么不能写 `cat input.txt | while read -r line`**：

```bash
# ❌ 错误写法：变量会丢失
count=0
cat input.txt | while read -r line; do
    count=$((count + 1))
done
echo "行数: $count"        # 输出 0 ！

# 原因：管道会让 while 在【子 shell】里运行，
# 子 shell 里对 count 的修改对父 shell 不可见。

# ✅ 正确写法：用重定向
count=0
while read -r line; do
    count=$((count + 1))
done < input.txt
echo "行数: $count"        # 正确
```

如果确实需要把命令的输出接进来（而不是文件），用**进程替换 (process substitution)**：

```bash
count=0
while read -r line; do
    count=$((count + 1))
done < <(some_command)
echo "$count"              # ✅ 正确，因为不是管道
```

`< <(...)` 是 bash 的进程替换：`<(cmd)` 被替换成一个类似 `/dev/fd/63` 的路径，因此用重定向而不是管道，while 不在子 shell 里。

### 4.3 `until`、`break`、`continue`

```bash
# until：条件为【假】时循环（与 while 相反）
n=1
until [ "$n" -gt 3 ]; do
    echo "$n"
    ((n++))
done

# break / continue
for i in {1..10}; do
    if [ "$i" -eq 3 ]; then
        continue          # 跳过本次迭代
    fi
    if [ "$i" -eq 7 ]; then
        break             # 退出整个循环
    fi
    echo "$i"
done

# 带层级的 break（跳出多层循环）
for i in {1..3}; do
    for j in {1..3}; do
        if [ "$j" -eq 2 ]; then
            break 2       # 跳出两层
        fi
    done
done

# 无限循环 + 退出条件
while true; do
    if some_check; then
        break
    fi
    sleep 5
done
```

### 4.4 一个小的选项解析器

`while` + `case` + `shift` 组合可以解析命令行选项：

```bash
#!/usr/bin/env bash
set -euo pipefail

verbose=false
output=""
inputs=()

while [ $# -gt 0 ]; do
    case "$1" in
        -v|--verbose)
            verbose=true
            shift
            ;;
        -o|--output)
            if [ $# -lt 2 ]; then
                echo "错误: $1 需要一个参数" >&2
                exit 1
            fi
            output="$2"
            shift 2                 # 连同参数一起移掉
            ;;
        -h|--help)
            echo "用法: $0 [-v] [-o FILE] [INPUT...]"
            exit 0
            ;;
        --)
            shift                   # -- 之后全是位置参数
            inputs+=("$@")
            break
            ;;
        -*)
            echo "未知选项: $1" >&2
            exit 1
            ;;
        *)
            inputs+=("$1")          # 非选项参数
            shift
            ;;
    esac
done

$verbose && echo "verbose 模式"
echo "输出到: ${output:-<stdout>}"
echo "输入文件数: ${#inputs[@]}"
[ ${#inputs[@]} -gt 0 ] && printf '  - %s\n' "${inputs[@]}"
```

这个模式简单、无需依赖，是小型脚本的最佳选择。**复杂的选项解析（短选项组合 `-abc`、长选项缩写）应当用 `getopts` 或直接换成 Python。** `getopts` 是 shell 内建，支持 POSIX 风格的选项解析，但不支持长选项：

```bash
while getopts ":vo:h" opt; do
    case "$opt" in
        v) verbose=true ;;
        o) output="$OPTARG" ;;
        h) echo "用法: $0 [-v] [-o FILE]"; exit 0 ;;
        :) echo "错误: -$OPTARG 需要参数" >&2; exit 1 ;;
        \?) echo "错误: 未知选项 -$OPTARG" >&2; exit 1 ;;
    esac
done
shift $((OPTIND - 1))     # 移掉已解析的选项，剩下的在 "$@" 里
```

开头 `getopts` 的冒号（`":vo:h"`）会让 `getopts` 进入静默模式，让你自己处理错误（`:` 分支处理缺参数，`\?` 处理未知选项）。

## 5. 函数

### 5.1 定义与调用

```bash
# 两种等价的定义方式
myfunc() {
    echo "hello"
}
function myfunc {          # 另一种写法（bash 支持，但 POSIX 不支持）
    echo "hello"
}

myfunc                     # 调用（不加括号！）
```

### 5.2 参数与返回值

**函数没有具名参数**，位置参数 `$1` `$2` 在函数内部指的是**函数的参数**（相当于函数有自己的参数作用域，但注意 `$0` 仍是脚本名）。

**`return` 返回的是整数状态码（0-255），不是一个值**。这是与大多数语言根本不同的地方。

```bash
# 通过退出码返回"是否成功"
is_even() {
    if (( $1 % 2 == 0 )); then
        return 0        # 成功/真
    else
        return 1        # 失败/假
    fi
}

if is_even 4; then
    echo "4 是偶数"
fi

# 通过 echo 返回"一个值"，调用方用命令替换捕获
get_timestamp() {
    date +%Y%m%d-%H%M%S
}
ts=$(get_timestamp)
echo "时间戳: $ts"

# 返回多个值：用 printf 输出多行，调用方用 readarray 接收
get_disk_info() {
    df -h / | awk 'NR==2 {print $2, $3, $5}'
}
read -r total used pct <<< "$(get_disk_info)"
echo "总量=$total 已用=$used 使用率=$pct"
```

> ⚠️ **返回值必须是 0-255 的整数**。`return 300` 会变成 `300 % 256 = 44`。要返回大数字或字符串，必须用 `echo` + 命令替换。

### 5.3 `local` 与作用域

```bash
var="global"

myfunc() {
    local var="local"        # 局部变量，不影响外部
    local other="临时"       # 函数结束时自动消失
    echo "$var"              # local
}

myfunc
echo "$var"                  # global
```

**不加 `local` 的变量是全局的**——函数里一不小心就会覆盖外部的同名变量，这是 shell 脚本里非常隐蔽的 bug 来源。**函数内除有意共享的变量外，一律加 `local`。**

### 5.4 常用工具函数

```bash
#!/usr/bin/env bash
set -euo pipefail

# 日志函数：带时间戳，输出到 stderr（不污染 stdout 的数据流）
log() {
    printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*" >&2
}

# 错误并退出
die() {
    printf '[ERROR] %s\n' "$*" >&2
    exit 1
}

# 检查命令是否存在
require_cmd() {
    command -v "$1" >/dev/null 2>&1 || die "缺少必需的命令: $1"
}

# 检查变量是否已设置且非空
require_var() {
    local name="$1"
    [ -n "${!name:-}" ] || die "环境变量 $name 未设置"
}

# 重试封装
retry() {
    local max="$1"; shift
    local delay="$2"; shift
    local n=0
    until "$@"; do
        n=$((n + 1))
        if [ "$n" -ge "$max" ]; then
            die "重试 ${max} 次后仍失败: $*"
        fi
        log "第 $n 次失败，${delay}s 后重试: $*"
        sleep "$delay"
    done
}

# 使用示例
require_cmd curl
retry 3 2 curl -fsS http://localhost:8080/health
log "全部完成"
```

注意 `require_var` 里的 `${!name:-}` 是**间接引用 (indirect expansion)**：`!name` 表示"取名为 `$name` 的变量的值"。这是检查动态变量名的标准技巧。

另外 `log` 把输出送到 **stderr**（`>&2`）是一个有意的设计：这样脚本的 stdout 可以干净地只输出"数据"，便于被其他命令消费，而诊断信息走 stderr。

## 6. 输入输出与重定向

### 6.1 `read`

```bash
read -r -p "请输入名字: " name              # -p 提示符，-r 不解释反斜杠
read -r -s -p "密码: " password             # -s 静默（不回显）
echo                                        # read -s 后需要手动换行
read -r -t 10 -p "10 秒内回答: " answer     # -t 超时（秒），超时返回非 0
read -r -n 1 -p "按任意键继续..."           # -n 1 只读一个字符
read -r -a arr -p "输入多个值: "            # -a 读入数组（按空白拆分）
read -r -d '' content < file                # -d '' 用 NUL 作分隔符，读整个文件
```

```bash
# 带超时和默认值的交互
if read -r -t 5 -p "继续? [Y/n] " ans; then
    :
else
    echo                      # 超时了，补一个换行
    ans="Y"                   # 默认继续
fi
case "${ans:-Y}" in
    [Yy]*) echo "继续" ;;
    *)     echo "取消"; exit 0 ;;
esac
```

### 6.2 here-document 与 here-string

```bash
# here-document：变量【会】被展开
cat <<EOF
当前用户: $USER
当前目录: $(pwd)
EOF

# 加了引号的定界符：变量【不会】被展开（字面量）
cat <<'EOF'
这段里的 $USER 和 $(pwd) 都是字面量，不会被替换
EOF

# 缩进友好的写法：<<- 会去掉行首的【制表符】（不是空格！）
cat <<-EOF
	缩进的文本
	会被去掉前面的 tab
	EOF

# 写入文件
cat > /tmp/config.yml <<'YAML'
server:
  port: 8080
YAML

# 追加
tee -a /tmp/app.log >/dev/null <<EOF
[$(date)] 任务完成
EOF

# here-string：把一个字符串作为 stdin
grep "pattern" <<< "$content"
read -r a b c <<< "1 2 3"
awk '{print $2}' <<< "hello world"
```

> ⚠️ **`<<'EOF'` 与 `<<EOF` 的区别非常实际**：如果内容里有 `$`（例如密码、模板变量、SQL 里的 `$1`），**必须用引号形式**，否则 shell 会尝试展开它，可能把变量替换成空字符串，导致配置静默错误。
>
> 另外 `<<-` 只去 **Tab**，不去空格。所以用 `<<-` 时缩进必须用 Tab，这在"编辑器自动把 Tab 转成空格"的环境里会失效。

### 6.3 `printf` 与 `echo`

```bash
echo "hello"                # 简单输出，自动加换行
echo -n "no newline"        # -n 不加换行（bash 内建支持）
echo -e "a\tb"              # -e 解释转义序列（bash 内建支持）

printf '%s\n' "hello"                    # 可移植、可控
printf '%-10s|%5d|%8.2f\n' "name" 42 3.14159
printf '%s\n' "$@"                       # 一行一个参数（比 echo 安全）
printf '%q\n' "$dangerous"               # 以可安全复制回 shell 的形式转义输出
```

**为什么脚本里应该优先用 `printf`**：
- `echo -e` 和 `echo -n` **不是 POSIX 标准**，不同 shell（dash、zsh、busybox）行为不同。有些 shell 的 `echo` 默认就解释转义，于是 `echo "C:\new"` 会输出一个换行；
- 如果第一个参数以 `-` 开头（例如 `echo "$line"` 而 `$line` 是 `-n`），`echo` 可能把它当成选项；
- `printf '%s\n' "$var"` 完全没有这些问题，且格式控制能力强得多。

**`printf '%q'` 是一个不常被提到但很有用的功能**：它把字符串转义成"可以直接粘贴回 shell 执行"的形式，用于生成脚本或在调试时打印参数：

```bash
$ printf '%q\n' "it's a \"test\""
it\'s\ a\ \"test\"
```

### 6.4 日志输出到文件与屏幕

```bash
# 全部输出到文件（含 stderr，注意顺序）
{
    echo "=== 开始 $(date) ==="
    some_command
    echo "=== 结束 ==="
} >> /var/log/myapp/run.log 2>&1

# 同时到屏幕和文件
some_command 2>&1 | tee -a /var/log/myapp/run.log

# 整个脚本的所有输出都同时写日志（放在脚本开头）
exec > >(tee -a /var/log/myapp/script.log) 2>&1
```

最后那条用了 `exec` 配合进程替换来**重定向整个脚本**的 stdout 与 stderr。它是 bash 特性（依赖进程替换），在 `sh` 下不可用。注意它会让脚本的输出变成异步的，脚本退出时可能有缓冲区未刷新——需要时加一个短暂的 `sleep 1` 或显式 `wait`。

## 7. 数组与字符串处理（bash 特性）

```bash
# 索引数组
arr=("one" "two three" "four")
arr+=( "five" )                     # 追加
echo "${arr[0]}"                    # one
echo "${arr[2]}"                    # four
echo "${#arr[@]}"                   # 元素个数：4
echo "${arr[@]}"                    # 所有元素（各自独立）
echo "${!arr[@]}"                   # 所有下标
unset 'arr[1]'                      # 删除某个元素（注意下标有引号）

# 遍历（必须加引号，否则含空格的元素会被拆开）
for item in "${arr[@]}"; do
    echo "[$item]"
done

# 用 mapfile / readarray 读文件到数组
mapfile -t lines < file.txt         # -t 去掉每行末尾的换行
echo "共 ${#lines[@]} 行"
echo "第一行: ${lines[0]}"

# 关联数组（哈希表，bash 4+）
declare -A counts
counts[apple]=3
counts[banana]=5
for key in "${!counts[@]}"; do
    echo "$key -> ${counts[$key]}"
done

# 关联数组的实用模式：统计
declare -A status_count
while read -r code; do
    (( status_count[$code]++ )) || true   # 注意 ((...)) 在结果为 0 时返回非 0，配合 set -e 会退出
done < codes.txt
for code in "${!status_count[@]}"; do
    printf '%s: %d\n' "$code" "${status_count[$code]}"
done
```

> ⚠️ **`((var++))` 配合 `set -e` 的坑**：当 `var` 从 0 变 1 时，`((var++))` 的**返回值是基于旧值的**（0），在算术求值语义里 0 表示"假"，于是返回退出码 1，触发 `set -e` 使脚本退出。规避方式：用 `((var++)) || true`，或写成 `var=$((var + 1))`，或 `((++var))`（前置自增，返回值是新值 1）。
>
> **关联数组（`declare -A`）需要 bash 4.0+**。macOS 自带的 bash 是 3.2，不支持关联数组——这是跨平台脚本的一个真实限制，需要注意。

## 8. 调试

### 8.1 打印执行的命令

```bash
bash -x script.sh                # 从头就打印每条命令
bash -v script.sh                # 打印读到的每一行（含注释与未执行的）
```

脚本内部精确控制：

```bash
set -x            # 从这里开始打印
risky_command
set +x            # 停止打印
```

`set -x` 的输出以 `+` 前缀表示嵌套层级（`+` 是顶层，`++` 是命令替换或子 shell 里）。

**让 `-x` 的输出更好读**：设置 `PS4`（trace 提示符）：

```bash
export PS4='+ ${BASH_SOURCE##*/}:${LINENO}:${FUNCNAME[0]:-main}: '
set -x
```

这样每条 trace 都带**文件名、行号、函数名**，定位问题快得多。

### 8.2 `trap` —— 清理与信号处理

```bash
#!/usr/bin/env bash
set -euo pipefail

tmpdir=$(mktemp -d)
cleanup() {
    local rc=$?
    log "清理临时目录: $tmpdir"
    rm -rf "$tmpdir"
    exit "$rc"                 # 保留原始退出码
}
trap cleanup EXIT              # 无论正常结束还是出错退出都执行

trap 'echo "收到 SIGINT，正在退出..." >&2' INT TERM

# 现在即使中途失败，临时目录也会被清理
cp data.csv "$tmpdir/"
process "$tmpdir/data.csv"
```

**`trap ... EXIT` 是脚本质量的重要标志**：它保证临时文件、锁文件、临时挂载点一定被清理，不会因为中途出错而留下垃圾。

其他常用的 trap 目标：

| 目标 | 触发时机 |
| --- | --- |
| `EXIT` | 脚本退出时（**任何原因**，包括 `set -e` 触发的退出） |
| `ERR` | 命令失败时（与 `set -e` 配合） |
| `INT` | 收到 `SIGINT`（`Ctrl+C`） |
| `TERM` | 收到 `SIGTERM`（`kill`） |
| `DEBUG` | 每条命令执行前（可做逐行跟踪） |

**关于 `ERR` 的一个要点**：`trap ... ERR` 在命令失败时触发，但**它的行为受 `set -e` 的同一套例外规则影响**（在 `if` 条件、`&&`/`||` 左侧、`!` 取反时不触发）。所以它不是一个万能的错误处理器。

```bash
# 出错时打印出错的行号
trap 'echo "错误发生在第 $LINENO 行" >&2' ERR
```

### 8.3 `shellcheck` —— 必备的静态检查工具

**`shellcheck` 是一个真实存在且被广泛使用的 shell 脚本静态分析工具**，能发现大量"能运行但会出问题"的写法，例如未加引号的变量、`[ ]` 里用 `>` 做数值比较、`for f in $(ls)`、不可达代码、拼错的变量名等。

```bash
sudo apt install shellcheck      # Debian/Ubuntu
sudo dnf install ShellCheck      # RHEL 系
```

```bash
shellcheck script.sh                 # 检查
shellcheck -S warning script.sh      # 只报告 warning 及以上
shellcheck -e SC2086 script.sh       # 忽略某个检查项
```

**建议把 shellcheck 加进编辑器**（VS Code 的 ShellCheck 扩展、vim 的 ALE 等），这样写的时候就有反馈。它给出的每条警告都带一个 `SCxxxx` 编号和官方说明链接，是学习 shell 细节的好材料。

**几个 shellcheck 最常报、也最值得记住的问题**：

| 编号 | 问题 | 正确写法 |
| --- | --- | --- |
| SC2086 | 变量未加引号，会词分隔 | `"$var"` |
| SC2046 | 命令替换未加引号 | `"$(cmd)"` |
| SC2006 | 用了旧式反引号 | 改成 `$(...)` |
| SC2181 | `if [ $? -eq 0 ]`，应先直接判断命令 | `if cmd; then` |
| SC2015 | `A && B \|\| C` 的语义容易被误读 | 用显式的 if/else |
| SC2164 | `cd` 未检查失败 | `cd dir \|\| exit 1` |
| SC1090 | 无法跟随的 `source` 路径 | 明确路径或用 `# shellcheck source=` |
| SC2155 | `local x=$(cmd)` 会吞掉退出码 | 分两行写 |

SC2155 值得单独说明：

```bash
# ❌ 出错时 $? 是 local 的返回值（0），command 的失败被吞掉
local result=$(risky_command)

# ✅ 分两步
local result
result=$(risky_command)
# 现在 $? 反映 risky_command 的真实结果
```

## 9. 四个完整的实用脚本

### 9.1 日志清理（带 dry-run）

```bash
#!/usr/bin/env bash
# cleanup-logs.sh —— 删除指定目录下超过 N 天的日志文件
set -euo pipefail

LOG_DIR="${1:-/var/log/myapp}"
DAYS="${2:-30}"
DRY_RUN="${DRY_RUN:-true}"        # 默认演练，必须显式关掉才会真删

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }

# --- 参数校验（先验证，再操作）---
[[ -d "$LOG_DIR" ]] || { echo "错误: $LOG_DIR 不是目录" >&2; exit 1; }
[[ "$DAYS" =~ ^[0-9]+$ ]] || { echo "错误: DAYS 必须是整数" >&2; exit 1; }
[[ "$DAYS" -ge 1 ]] || { echo "错误: DAYS 至少为 1" >&2; exit 1; }

# 防止把根目录或家目录当目标
case "$(readlink -f "$LOG_DIR")" in
    /|/home|/root|/etc|/usr|/var)
        echo "错误: 拒绝在 $LOG_DIR 上操作" >&2
        exit 1
        ;;
esac

log "目标目录: $LOG_DIR"
log "保留天数: $DAYS"
log "演练模式: $DRY_RUN"

# --- 找出候选文件 ---
# -print0 + read -d '' 保证含空格/换行的文件名也安全
mapfile -d '' targets < <(find "$LOG_DIR" -type f -name '*.log' -mtime "+$DAYS" -print0)

if [ "${#targets[@]}" -eq 0 ]; then
    log "没有符合条件的文件"
    exit 0
fi

total_bytes=0
for f in "${targets[@]}"; do
    size=$(stat -c '%s' "$f" 2>/dev/null || echo 0)
    total_bytes=$((total_bytes + size))
done

log "找到 ${#targets[@]} 个文件，合计 $((total_bytes / 1024 / 1024)) MB"

if [[ "$DRY_RUN" == "true" ]]; then
    printf '%s\n' "${targets[@]}" | head -20
    [ "${#targets[@]}" -gt 20 ] && log "...（还有 $(( ${#targets[@]} - 20 )) 个）"
    log "演练结束。确认无误后加 DRY_RUN=false 重新运行。"
    exit 0
fi

# --- 真正删除 ---
# 先压缩再删，留一份归档（可选，按需删除本段）
ARCHIVE="/var/backups/logs-$(date +%Y%m%d-%H%M%S).tar.gz"
mkdir -p "$(dirname "$ARCHIVE")"
tar -czf "$ARCHIVE" --files-from <(printf '%s\n' "${targets[@]}")
log "已归档到 $ARCHIVE"

printf '%s\0' "${targets[@]}" | xargs -0 rm -v
log "完成"
```

**这个脚本体现的几条纪律**：
- **默认 dry-run**（`DRY_RUN=true`），必须显式关闭才会真删；
- **参数校验在操作之前**，且校验顺序是"类型 → 范围"；
- **拒绝在系统关键目录上操作**（用 `readlink -f` 先解析成绝对路径，避免符号链接绕过）；
- **用 `-print0` + `mapfile -d ''`** 处理任意文件名；
- **先归档再删除**，给后悔留余地。

### 9.2 系统健康巡检

```bash
#!/usr/bin/env bash
# health-check.sh —— 打印系统关键指标，超阈值时给出告警
set -uo pipefail                  # 故意不加 -e：某一项检查失败不应中断整份报告

CPU_WARN="${CPU_WARN:-80}"        # CPU 使用率告警阈值（%）
MEM_WARN="${MEM_WARN:-85}"        # 内存使用率告警阈值（%）
DISK_WARN="${DISK_WARN:-90}"      # 磁盘使用率告警阈值（%）
LOAD_WARN_RATIO="${LOAD_WARN_RATIO:-1.0}"   # load / 核心数 的告警比例

warnings=0
warn()  { printf '  ⚠️  %s\n' "$*"; warnings=$((warnings + 1)); }
ok()    { printf '  ✅  %s\n' "$*"; }
title() { printf '\n=== %s ===\n' "$*"; }

title "基本信息"
printf '  主机: %s\n' "$(hostname)"
printf '  时间: %s\n' "$(date '+%F %T %Z')"
printf '  内核: %s\n' "$(uname -r)"
printf '  运行: %s\n' "$(uptime -p 2>/dev/null || uptime)"
CORES=$(nproc)
printf '  CPU 核心数: %s\n' "$CORES"

title "负载 (load average)"
read -r l1 l5 l15 _ < /proc/loadavg
printf '  1/5/15 分钟: %s %s %s\n' "$l1" "$l5" "$l15"
# 用 awk 做浮点比较（bash 不支持浮点）
over=$(awk -v l="$l1" -v c="$CORES" -v r="$LOAD_WARN_RATIO" \
    'BEGIN { print (l > c * r) ? 1 : 0 }')
if [ "$over" -eq 1 ]; then
    warn "1 分钟负载 $l1 超过 ${LOAD_WARN_RATIO}x 核心数 ($CORES)"
else
    ok "负载正常"
fi

title "CPU 使用率"
# 通过两次读取 /proc/stat 算差值，得到真实瞬时使用率
read_cpu() { awk '/^cpu / {print $2+$3+$4, $5}' /proc/stat; }
read -r t1 i1 < <(read_cpu)
sleep 1
read -r t2 i2 < <(read_cpu)
cpu_pct=$(awk -v t1="$t1" -v i1="$i1" -v t2="$t2" -v i2="$i2" \
    'BEGIN { dt = t2 - t1; di = i2 - i1; if (dt > 0) printf "%.1f", (dt - di) * 100 / dt; else print "0" }')
printf '  使用率: %s%%\n' "$cpu_pct"
awk -v p="$cpu_pct" -v w="$CPU_WARN" 'BEGIN { exit !(p > w) }' \
    && warn "CPU 使用率 ${cpu_pct}% 超过阈值 ${CPU_WARN}%" \
    || ok "CPU 使用率正常"

title "内存"
read -r mem_total mem_used mem_free _ < <(free -m | awk '/^Mem:/ {print $2, $3, $4}')
mem_pct=$(awk -v u="$mem_used" -v t="$mem_total" 'BEGIN { printf "%.1f", u * 100 / t }')
printf '  总计: %s MB  已用: %s MB  空闲: %s MB  使用率: %s%%\n' \
    "$mem_total" "$mem_used" "$mem_free" "$mem_pct"
awk -v p="$mem_pct" -v w="$MEM_WARN" 'BEGIN { exit !(p > w) }' \
    && warn "内存使用率 ${mem_pct}% 超过阈值 ${MEM_WARN}%" \
    || ok "内存使用率正常"

# swap 使用情况（有 swap 时才报告）
swap_total=$(free -m | awk '/^Swap:/ {print $2}')
if [ "${swap_total:-0}" -gt 0 ]; then
    swap_used=$(free -m | awk '/^Swap:/ {print $3}')
    printf '  Swap: %s / %s MB\n' "$swap_used" "$swap_total"
fi

title "磁盘"
while read -r fs size used avail pct mount; do
    # 跳过非物理文件系统
    case "$fs" in
        tmpfs|devtmpfs|overlay|squashfs|none) continue ;;
    esac
    pct_num="${pct%\%}"
    printf '  %-24s %6s 已用 %6s / %6s (%s) 挂载于 %s\n' \
        "$fs" "$size" "$used" "$avail" "$pct" "$mount"
    if [ "${pct_num:-0}" -ge "$DISK_WARN" ]; then
        warn "磁盘 $mount 使用率 $pct 达到阈值 ${DISK_WARN}%"
    fi
done < <(df -hP | tail -n +2)

title "Top 5 CPU 进程"
ps -eo pcpu,pmem,pid,user,comm --sort=-pcpu | head -6

title "Top 5 内存进程"
ps -eo pmem,pcpu,pid,user,comm --sort=-pmem | head -6

title "失败的服务"
if command -v systemctl >/dev/null 2>&1; then
    failed=$(systemctl --failed --no-legend --plain 2>/dev/null | wc -l)
    if [ "$failed" -gt 0 ]; then
        systemctl --failed --no-pager
        warn "$failed 个服务处于失败状态"
    else
        ok "所有服务正常"
    fi
fi

title "汇总"
if [ "$warnings" -eq 0 ]; then
    echo "  全部检查通过 ✅"
    exit 0
else
    echo "  共 $warnings 项告警 ⚠️"
    exit 1                 # 非 0 退出码便于被监控系统捕获
fi
```

**这个脚本的几个技术点**：
- **故意不加 `set -e`**：巡检报告应该跑完所有项目，而不是第一个失败就中断；
- **`read -r l1 l5 l15 _ < /proc/loadavg`**：直接读内核暴露的数据源，不解析 `uptime` 的输出文本（更稳）；
- **两次读 `/proc/stat` 算 CPU 使用率**：`top`/`ps` 的 `%CPU` 是累计值，要瞬时值必须算差值；
- **用 `awk` 做浮点比较**：bash 的 `(( ))` 和 `[ ]` 只支持整数，`0.85 > 0.8` 这种比较必须交给 `awk`；
- **退出码反映是否有告警**，便于被 cron/监控直接使用。

### 9.3 批量备份

```bash
#!/usr/bin/env bash
# backup.sh —— 打包指定目录，保留最近 N 份，并校验归档完整性
set -euo pipefail

SRC="${1:?用法: $0 <源目录> [备份目录] [保留份数]}"
BACKUP_DIR="${2:-/var/backups}"
KEEP="${3:-7}"

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }
die() { printf '[ERROR] %s\n' "$*" >&2; exit 1; }

# --- 校验 ---
[[ -d "$SRC" ]] || die "源目录不存在: $SRC"
[[ "$KEEP" =~ ^[0-9]+$ ]] || die "保留份数必须是整数"
[[ "$KEEP" -ge 1 ]] || die "保留份数至少为 1"
command -v tar >/dev/null 2>&1 || die "找不到 tar"

SRC_ABS=$(readlink -f "$SRC")
NAME=$(basename "$SRC_ABS")
STAMP=$(date +%Y%m%d-%H%M%S)
ARCHIVE="$BACKUP_DIR/${NAME}-${STAMP}.tar.gz"

mkdir -p "$BACKUP_DIR"

# --- 清理函数：失败时删除半成品 ---
cleanup() {
    local rc=$?
    if [ "$rc" -ne 0 ] && [ -f "$ARCHIVE" ]; then
        log "备份失败 (退出码 $rc)，删除不完整的归档"
        rm -f "$ARCHIVE"
    fi
    exit "$rc"
}
trap cleanup EXIT

# --- 打包 ---
log "开始备份 $SRC_ABS -> $ARCHIVE"
# 用 -C 切换目录，这样归档里的路径是相对名（便于异机恢复）
tar -czf "$ARCHIVE" -C "$(dirname "$SRC_ABS")" "$NAME"
log "打包完成，大小: $(du -h "$ARCHIVE" | cut -f1)"

# --- 校验归档完整性 ---
log "校验归档..."
if ! tar -tzf "$ARCHIVE" >/dev/null 2>&1; then
    die "归档校验失败，文件可能已损坏"
fi
file_count=$(tar -tzf "$ARCHIVE" | wc -l)
log "校验通过，归档内含 $file_count 个条目"

# --- 比对条目数（粗粒度验证，防止归档为空）---
if [ "$file_count" -lt 1 ]; then
    die "归档为空，拒绝保留"
fi

# --- 轮转：只保留最近 KEEP 份 ---
log "轮转旧备份（保留最近 $KEEP 份）"
mapfile -t old < <(
    find "$BACKUP_DIR" -maxdepth 1 -type f -name "${NAME}-*.tar.gz" \
        -printf '%T@ %p\n' | sort -rn | awk -v k="$KEEP" 'NR > k {print $2}'
)

if [ "${#old[@]}" -eq 0 ]; then
    log "无需清理"
else
    printf '  删除: %s\n' "${old[@]}"
    printf '%s\0' "${old[@]}" | xargs -0 rm -f
    log "已删除 ${#old[@]} 个旧备份"
fi

log "当前备份列表:"
find "$BACKUP_DIR" -maxdepth 1 -type f -name "${NAME}-*.tar.gz" \
    -printf '%TY-%Tm-%Td %TH:%TM  %10s  %p\n' | sort

log "备份成功"
```

**技术点**：
- **`tar -C "$(dirname ...)"`**：切换目录后再打包，归档内的路径是相对路径，恢复到其他机器时不依赖原绝对路径；
- **`trap cleanup EXIT`**：失败时删除半成品归档，避免留下一个"看起来存在但无法使用"的备份——**这比备份失败更危险**，因为监控可能只看文件是否存在；
- **`tar -tzf` 校验**：确认归档可读、条目数合理（防止空归档）；
- **`find -printf '%T@ %p'` + `sort -rn` + `awk 'NR > k'`**：按 mtime 排序后取"第 k 条之后的"，即要删除的旧备份。`%T@` 是 mtime 的 Unix 时间戳，便于数值排序；
- **`${1:?用法: ...}`**：参数缺失时自动打印用法并退出，配合 `set -u` 很好用。

### 9.4 服务健康检查与自愈

```bash
#!/usr/bin/env bash
# watch-service.sh —— 轮询健康端点，连续失败则重启服务
set -uo pipefail                 # 不用 -e：失败是预期内的情况，要自己处理

SERVICE="${1:?用法: $0 <systemd-unit> [健康URL] [连续失败阈值]}"
HEALTH_URL="${2:-http://127.0.0.1:8080/health}"
MAX_FAILS="${3:-3}"
INTERVAL="${INTERVAL:-30}"       # 检查间隔（秒）
TIMEOUT="${TIMEOUT:-5}"          # 单次请求超时

STATE_DIR="${STATE_DIR:-/var/lib/watch-service}"
STATE_FILE="$STATE_DIR/${SERVICE}.fails"

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }

command -v curl >/dev/null 2>&1 || { echo "需要 curl" >&2; exit 1; }
command -v systemctl >/dev/null 2>&1 || { echo "需要 systemctl" >&2; exit 1; }

mkdir -p "$STATE_DIR"
fails=0
[ -f "$STATE_FILE" ] && fails=$(cat "$STATE_FILE" 2>/dev/null || echo 0)
[[ "$fails" =~ ^[0-9]+$ ]] || fails=0

log "开始监控 $SERVICE"
log "健康端点: $HEALTH_URL  间隔: ${INTERVAL}s  阈值: $MAX_FAILS  当前连续失败: $fails"

# 退出时保存状态，这样重启监控脚本不会丢失计数
trap 'echo "$fails" > "$STATE_FILE"; log "退出，已保存失败计数 $fails"' EXIT

while true; do
    code=$(curl -s -o /dev/null -m "$TIMEOUT" -w '%{http_code}' "$HEALTH_URL" 2>/dev/null || echo "000")

    if [[ "$code" =~ ^2[0-9][0-9]$ ]]; then
        if [ "$fails" -gt 0 ]; then
            log "健康检查恢复 (HTTP $code)，重置失败计数"
        fi
        fails=0
    else
        fails=$((fails + 1))
        log "健康检查失败 ${fails}/${MAX_FAILS} (HTTP $code)"

        if [ "$fails" -ge "$MAX_FAILS" ]; then
            log "达到阈值，重启 $SERVICE"
            if systemctl restart "$SERVICE"; then
                log "重启命令已执行，等待服务稳定"
                sleep 5
                new_code=$(curl -s -o /dev/null -m "$TIMEOUT" -w '%{http_code}' "$HEALTH_URL" 2>/dev/null || echo "000")
                if [[ "$new_code" =~ ^2[0-9][0-9]$ ]]; then
                    log "重启后恢复健康 (HTTP $new_code)"
                    fails=0
                else
                    log "重启后仍不健康 (HTTP $new_code)，保留计数等待下次尝试"
                fi
            else
                log "重启失败，请检查: journalctl -u $SERVICE -n 50"
            fi
        fi
    fi

    echo "$fails" > "$STATE_FILE"
    sleep "$INTERVAL"
done
```

**技术点**：
- **`curl -w '%{http_code}'`**：只取状态码，用 `-o /dev/null` 丢弃响应体。`-m` 设超时（**必须设**，否则一个卡住的连接会让检查永久挂起）；
- **`|| echo "000"`**：curl 失败时不输出状态码，用 `000` 表示"没连上"，与其他情况区分；
- **状态持久化**：把连续失败计数写到文件，这样监控脚本自己重启后不会重置计数（避免"重启监控脚本导致永远达不到阈值"）；
- **不用 `set -e`**：健康检查失败是预期情况，脚本必须继续；
- **指数退避没做，但阈值机制足够**：真正生产环境应该用成熟的监控系统（Prometheus + Alertmanager、systemd 的 `Restart=`、或进程管理器如 supervisord）而不是自己写轮询脚本。**这个脚本的价值在于理解机制，而不是替代监控系统。**

最后这一点值得强调：**这个脚本是本篇里我唯一会明确说"不要拿到生产用"的例子**。自己写的自愈脚本容易造成"重启风暴"、掩盖真实问题、并且没有告警。生产环境应当用 systemd 的 `Restart=on-failure` + `RestartSec=` + `StartLimitBurst=`（systemd 内建的重启频率限制），或交给专门的监控告警系统。**把它当成一个理解 curl/journalctl/systemctl 配合的练习更合适。**

## 10. 脚本编写检查清单

写完后逐条自查（这些都能被 shellcheck 抓出大半）：

| # | 检查项 |
| --- | --- |
| 1 | 有 shebang 吗？用 `#!/usr/bin/env bash` 更可移植 |
| 2 | 有 `set -euo pipefail` 吗？（除非有理由不加，且加了注释说明） |
| 3 | 所有变量引用都加双引号了吗？转发参数用 `"$@"` 而不是 `$@` 或 `$*` |
| 4 | 整数比较用 `-eq`/`-gt` 而不是 `>`/`<` 吗？ |
| 5 | 有 `trap ... EXIT` 清理临时文件/锁吗？ |
| 6 | 危险操作（删除、覆盖）有 dry-run 模式吗？有路径校验吗？ |
| 7 | 函数内的临时变量都加了 `local` 吗？ |
| 8 | 关键步骤检查了退出码吗？（不要依赖 `set -e` 覆盖一切） |
| 9 | 向用户报告错误时输出到 stderr（`>&2`）吗？ |
| 10 | 出错信息包含足够定位的上下文（哪一步、哪个文件、什么值）吗？ |
| 11 | 用 `command -v` 检查了所有外部依赖吗？ |
| 12 | 过了 shellcheck 吗？ |
| 13 | 路径、变量都做了引号与空格保护吗？（用带空格的文件名测一遍） |
| 14 | 脚本的退出码有意义吗？（监控要靠它） |
| 15 | 处理了"没有匹配文件"（`nullglob`）和"输入为空"的情况吗？ |

## 小结

- shebang 决定用哪个解释器；`./script.sh` 需要执行位并读 shebang，`bash script.sh` 忽略 shebang。`source` 会在当前 shell 执行，会污染环境。
- `set -euo pipefail` 是推荐开头，但 **`set -e` 有大量例外**（`if` 条件、`&&`/`||` 左侧、`!` 取反、函数在条件里被调用），关键步骤要显式检查。
- 赋值时**等号两边不能有空格**；引用时**不确定就加双引号**。`'单引号'` 完全不展开，`"双引号"` 展开变量但不做通配符展开。
- **bash 算术只支持整数**，小数必须用 `bc` 或 `awk`。`$((10/3))` 是 `2`。
- 转发参数永远用 `"$@"`；`"$*"` 会把所有参数合成一个字符串，不加引号则会被词分隔。
- bash 脚本推荐用 `[[ ]]`（不做词分隔、支持 `&&`/`||`/`=~`）；用 `[ ]` 时**变量必须加引号**。
- **`[ "$a" > "$b" ]` 是错的**——`>` 被当成重定向。整数比较必须用 `-gt` 等运算符。
- `${VAR##*/}` 取文件名、`${VAR%/*}` 取目录、`${VAR%.*}` 去扩展名；`#`/`%` 管前后缀，双写表示最长匹配。
- 读文件用 `while IFS= read -r line; do ... done < file`；**`cat file | while read` 会让循环在子 shell 里，变量修改丢失**。需要接命令输出时用进程替换 `< <(cmd)`。
- `for f in *.log` 在没有匹配时会展开成字面量 `*.log`，用 `shopt -s nullglob` 或 `[[ -e "$f" ]] || continue` 处理。
- 函数**没有具名参数**，`return` 只能返回 0-255 的整数；要"返回字符串"就用 `echo` + `$(...)` 捕获。函数内一律用 `local`。
- 脚本里优先用 `printf` 而不是 `echo -e`/`echo -n`（后者不是 POSIX 标准，行为因 shell 而异）。
- `<<'EOF'`（定界符加引号）**阻止变量展开**；内容里有 `$` 时必须用这种形式。`<<-` 只去掉行首的 **Tab**，不去空格。
- **`((var++))` 在 `var` 为 0 时返回假（退出码 1），配合 `set -e` 会让脚本意外退出**；用 `((var++)) || true` 或 `var=$((var+1))`。
- 关联数组 `declare -A` 需要 bash 4+，macOS 自带的 bash 3.2 不支持。
- 调试用 `bash -x` 或 `set -x`，配合 `PS4` 带上文件名与行号；用 `trap ... EXIT` 保证清理。
- **`shellcheck` 是必备工具**，它抓的问题（SC2086 未加引号、SC2181 多余检查 `$?`、SC2155 `local x=$(cmd)` 吞掉退出码）正是脚本 bug 的高发区。
- 危险脚本的纪律：**默认 dry-run**、**先校验参数再操作**、**用 `-print0`/`mapfile -d ''` 处理任意文件名**、**`trap` 清理**、**失败时删除半成品**（一个损坏但存在的备份比没有备份更危险）。
- 自写的"自愈/轮询"脚本不适合生产：优先用 systemd 的 `Restart=` + `RestartSec=` + `StartLimitBurst=`，以及成熟的监控告警系统。
