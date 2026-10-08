# Linux 文本处理：grep / sed / awk 与管道组合

本篇整理标准流与重定向的语义、grep/sed/awk 三大文本工具的核心用法，以及 sort/uniq/wc/cut/tr 这些配合工具，最后用完整的管道实例把它们串起来。

## 1. 标准流与重定向

### 1.1 三个标准流

每个进程启动时默认打开三个文件描述符 (file descriptor, fd)：

| fd | 名称 | 默认去向 | 说明 |
| --- | --- | --- | --- |
| 0 | stdin | 键盘 | 程序读取输入的来源 |
| 1 | stdout | 终端 | 程序的正常输出 |
| 2 | stderr | 终端 | 错误与警告 |

**为什么要区分 stdout 和 stderr**：这样"正常结果"和"错误信息"可以被分别处理。在管道里，只有 stdout 会流向下一个命令，stderr 仍然直接打到终端——所以即使管道里的某个命令报错，错误信息也不会污染数据流。这是一个有意设计的分离。

### 1.2 重定向

```bash
cmd > file            # stdout 覆盖写入 file（文件已存在则先清空！）
cmd >> file           # stdout 追加到 file 末尾
cmd 2> err.log        # stderr 写入 err.log
cmd > out.log 2>&1    # stdout 和 stderr 都写入 out.log（正确写法）
cmd &> out.log        # 上面这条的 bash 简写（不是 POSIX sh 语法）
cmd > /dev/null 2>&1  # 彻底静默（丢弃正常输出和错误）
cmd < input.txt       # 从文件读 stdin
cmd <<< "a string"    # here-string，把字符串作为 stdin（bash）
```

### 1.3 重定向顺序的语义 —— 必须理解这一点

`2>&1` 的意思是"**把 fd 2 变成 fd 1 当前所指的那个东西**"。关键词是"当前"。因此顺序决定了结果：

```bash
# 情况 A：先重定向 stdout，再把 stderr 指向 stdout 的目标
cmd > file 2>&1
# 步骤：1 → file；然后 2 → 1 当前所指（即 file）
# 结果：正常输出和错误都进 file ✅

# 情况 B：先把 stderr 指向当前 stdout（此时还是终端），再重定向 stdout
cmd 2>&1 > file
# 步骤：2 → 1 当前所指（终端！）；然后 1 → file
# 结果：错误进终端，正常输出进 file ❌（与意图相反）
```

用一张表记住：

| 写法 | stdout 去哪 | stderr 去哪 |
| --- | --- | --- |
| `> file 2>&1` | file | file |
| `2>&1 > file` | file | **终端** |
| `> file 2> err.log` | file | err.log |
| `&> file` | file | file |

`2>&1 > file` 是新手最常见的错误之一，而且它**不会报错**，只是行为不对，所以特别难发现。

另外两个常用技巧：

```bash
# 同时输出到终端和文件（tee）
cmd | tee output.log
cmd | tee -a output.log          # 追加而非覆盖

# 交换 stdout 与 stderr
cmd 3>&1 1>&2 2>&3 3>&-
```

`tee` 的用途是"既想看又想存"，例如跑一个长时间构建时把输出同时留档。注意：只写 `cmd | tee log` 时 stderr **不会**进 tee，要写成 `cmd 2>&1 | tee log`。

### 1.4 管道 `|`

```bash
cmd1 | cmd2 | cmd3
```

把 `cmd1` 的 stdout 接到 `cmd2` 的 stdin，依此类推。要点：

- **stderr 不经过管道**（除非显式 `2>&1`）；
- 管道里的每个命令**在各自的子进程里运行**（bash 中默认如此），因此在管道内部修改的变量在父 shell 里看不到。这是 `cat file | while read` 会丢失变量的原因（见 [06-shell-scripting.md](06-shell-scripting.md)）；
- 管道有缓冲区（通常 64 KiB），缓冲区满时写端会阻塞——这决定了管道是"流式"处理的，不是等前一个命令全部结束；
- **`SIGPIPE`**：如果下游提前退出（例如 `... | head -5`），上游继续写就会收到 `SIGPIPE` 并被终止。这就是为什么 `head` 之后有时会看到 `Broken pipe` 报错。

### 1.5 `xargs` —— 把 stdin 变成参数

有些命令不从 stdin 读数据，只接受参数（`rm`、`mkdir`、`kill`）。这时用 `xargs` 转换：

```bash
# 删除所有 .tmp 文件（rm 不接受 stdin，必须用 xargs 或 find -exec）
find . -name "*.tmp" | xargs rm -v

# 处理含空格的文件名：用 -0 配合 find -print0（关键！）
find . -name "*.tmp" -print0 | xargs -0 rm -v

# 控制每次传递的参数个数
find . -name "*.log" | xargs -n 10 gzip

# 用 -I 指定替换占位符（每次处理一个）
cat urls.txt | xargs -I {} curl -sO {}
cat files.txt | xargs -I {} sh -c 'echo "=== {} ==="; head -3 "{}"'

# -t 打印将要执行的命令（调试 xargs 的标准手段）
find . -name "*.bak" | xargs -t rm

# -P 并行执行（4 个并发）
find . -name "*.png" -print0 | xargs -0 -P 4 -n 1 optipng
```

> ⚠️ **空格与引号陷阱**：文件名含空格时，`xargs` 默认按空白切分会把 `my file.txt` 拆成 `my` 和 `file.txt` 两个参数。**只要文件名可能含空格，就必须用 `-print0` + `xargs -0`**（NUL 分隔，NUL 是文件名里唯一不允许出现的字节）。此外，如果文件名里含引号，`xargs` 可能把它当语法处理；`-0` 也能避免这个问题。

`find -exec ... +` 与 `find | xargs` 的取舍：`-exec ... +` 不需要第二个进程，且天生安全（find 直接传参，不经过字符串解析）。能用 `-exec ... +` 时优先用它。

## 2. `grep` —— 按模式筛选行

### 2.1 常用选项

| 选项 | 含义 |
| --- | --- |
| `-i` | 忽略大小写 (ignore case) |
| `-v` | 反向匹配：输出**不**匹配的行 (invert) |
| `-n` | 显示行号 |
| `-r` / `-R` | 递归搜索目录（`-R` 会跟随符号链接） |
| `-l` | 只输出**含匹配的文件名**，不输出内容 |
| `-L` | 只输出**不含匹配的文件名** |
| `-c` | 只输出匹配的**行数** |
| `-w` | 全词匹配 (word)：`grep -w cat` 不匹配 `category` |
| `-E` | 使用扩展正则 (ERE)，等价于 `egrep` |
| `-F` | 固定字符串 (fixed)，不做正则解释，等价于 `fgrep`，最快 |
| `-A n` | 匹配行之后额外显示 n 行 (After) |
| `-B n` | 匹配行之前额外显示 n 行 (Before) |
| `-C n` | 前后各 n 行 (Context) |
| `-o` | 只输出匹配到的部分，而不是整行 |
| `-q` | 静默，只用退出码表示是否匹配（脚本里最有用） |
| `--include` / `--exclude` | 限定/排除文件模式 |
| `-h` | 不显示文件名（多文件搜索时） |

### 2.2 实例

```bash
# 在日志里找错误（不区分大小写），带行号和上下文
grep -in -C 3 "error" /var/log/app.log

# 递归搜索源码里的某个函数调用，只列出文件名
grep -rl "TODO" ~/project/src

# 统计出现次数（注意 -c 数的是"行数"，不是"出现次数"）
grep -c "404" access.log

# 统计真正的"出现次数"，用 -o 再数行
grep -o "404" access.log | wc -l

# 排除空行和注释行（读配置文件的标准手法）
grep -v -E '^\s*(#|$)' /etc/nginx/nginx.conf

# 全词匹配，避免子串误匹配
grep -w "admin" /etc/passwd

# 扩展正则：匹配 IPv4 的粗粒度模式
grep -E '^([0-9]{1,3}\.){3}[0-9]{1,3} ' access.log

# 只要匹配的片段（例如提取所有邮箱）
grep -oE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' contacts.txt

# 只看 /etc 下的配置文件，跳过二进制和其他类型
grep -rn --include="*.conf" "listen" /etc/

# 脚本里用退出码判断
if grep -q "FAILED" test-report.txt; then
    echo "测试失败"
fi
```

**`grep` 的退出码**（脚本里依赖它，必须记住）：

| 退出码 | 含义 |
| --- | --- |
| 0 | 至少有一行匹配 |
| 1 | 没有匹配行 |
| 2 | 出错（文件不存在、正则非法等） |

所以 `if grep -q pattern file` 才是正确写法；用 `if [ $(grep -c ...) -gt 0 ]` 不但多余，还会在文件不存在时得到混乱的结果。

### 2.3 BRE 与 ERE

| 类型 | 元字符 | 用法 |
| --- | --- | --- |
| BRE (基本正则) | `^ $ . * [ ] \` | `grep`（不带 `-E`） |
| ERE (扩展正则) | 多了 `+ ? { } ( ) \|` | `grep -E`（即 `egrep`） |

在 BRE 里 `+`、`?`、`|`、`()` **不是元字符**，要写成 `\+`、`\?`、`\|`、`\(\)`。日常使用几乎总是加 `-E` 更省心。`grep -F`（`fgrep`）则完全关闭正则，把模式当纯字符串——搜含 `.` `[` `*` 的内容时用它最安全也最快。

## 3. `sed` —— 流编辑器

`sed` 按行读入、按脚本处理后输出，适合**逐行的替换、删除、插入、提取**。

### 3.1 替换 `s///`

```bash
sed 's/old/new/' file              # 每行只替换第一次出现
sed 's/old/new/g' file             # 每行替换所有出现 (global)
sed 's/old/new/2' file             # 只替换每行的第 2 次出现
sed 's/old/new/gi' file            # 全局 + 忽略大小写
sed -i 's/old/new/g' file          # 直接修改文件 (in place)
sed -i.bak 's/old/new/g' file      # 修改同时留一个 .bak 备份
sed -E 's/([a-z]+)@([a-z.]+)/\2:\1/' file   # 扩展正则 + 反向引用 \1 \2
```

**`-i` 的可移植性问题（重要）**：GNU `sed`（Linux）的 `-i` 可以不带参数直接跟脚本，可选的备份后缀要**紧贴** `-i`（写 `-i.bak`）。而 BSD/macOS 的 `sed` 要求 `-i` **必须**带一个参数，不留备份要显式写空字符串：

```bash
# Linux (GNU sed)
sed -i 's/a/b/g' file
sed -i.bak 's/a/b/g' file

# macOS (BSD sed)
sed -i '' 's/a/b/g' file
sed -i '.bak' 's/a/b/g' file
```

在跨平台脚本里，要么用临时文件自己实现，要么检测系统。用临时文件的方式最稳：

```bash
tmp=$(mktemp) && sed 's/a/b/g' file > "$tmp" && mv "$tmp" file
```

### 3.2 寻址 (addressing)

`sed` 的脚本可以加地址前缀，限定作用于哪些行：

| 地址形式 | 含义 |
| --- | --- |
| `3` | 第 3 行 |
| `2,5` | 第 2 到第 5 行 |
| `$` | 最后一行 |
| `/pattern/` | 匹配该模式的行 |
| `/start/,/end/` | 从匹配 start 的行到匹配 end 的行 |
| `2~3` | 从第 2 行开始每隔 3 行（GNU 扩展） |

```bash
sed -n '10,20p' file              # 打印第 10-20 行（-n 抑制默认输出，p 才是打印）
sed -n '5p' file                  # 打印第 5 行
sed -n '$p' file                  # 打印最后一行
sed -n '/BEGIN/,/END/p' file      # 打印两个标记之间的内容
sed '1d' file                     # 删除第 1 行（即"跳过表头"）
sed '/^$/d' file                  # 删除所有空行
sed '/^#/d' file                  # 删除所有以 # 开头的行
sed '$d' file                     # 删除最后一行
sed '3i\插入的文字' file          # 在第 3 行之前插入
sed '3a\追加的文字' file          # 在第 3 行之后追加
sed '2c\替换掉的文字' file        # 替换第 2 行
```

### 3.3 多个命令与一次做完

```bash
sed -e 's/a/b/' -e 's/c/d/' file          # 多个 -e
sed 's/a/b/; s/c/d/' file                 # 或者分号分隔
sed -n -e '/START/,/END/{ /SKIP/d; p }' file   # 花括号组合命令
```

### 3.4 几个真正有用的 sed 单行

```bash
# 1. 删除所有空行（含只有空格的行）
sed '/^[[:space:]]*$/d' file

# 2. 删除注释行与空行，得到"有效配置"
sed -e 's/#.*$//' -e '/^[[:space:]]*$/d' config.conf

# 3. 在每行行尾加一个字符串
sed 's/$/  # reviewed/' list.txt

# 4. 在每行行首加一个前缀
sed 's/^/PREFIX: /' list.txt

# 5. 把 Windows 换行 (CRLF) 转成 Unix 换行 (LF)
sed -i 's/\r$//' file.txt

# 6. 交换两列（用制表符分隔）
sed -E 's/^([^\t]+)\t([^\t]+)$/\2\t\1/' file.tsv

# 7. 只对匹配的行做替换
sed '/^server/ s/8080/9090/' config.conf

# 8. 把匹配行里的内容提取出来（配合 -n 与反向引用）
sed -n 's/.*version=\([0-9.]*\).*/\1/p' app.log
```

第 5 条（去掉 `\r`）在 Windows 与 Linux 之间传文件时几乎每次都会用到。

## 4. `awk` —— 按字段处理与计算

`awk` 是一门小语言：它**自动逐行读取，并把每行拆成字段**，然后执行 `模式 { 动作 }`。这是它与 `sed` 的根本区别——`sed` 面向"整行文本"，`awk` 面向"结构化字段"。

### 4.1 字段模型与内置变量

| 变量 | 含义 |
| --- | --- |
| `$0` | 当前整行 |
| `$1` `$2` ... | 第 1、2... 个字段 |
| `$NF` | 最后一个字段（`NF` 是字段数量） |
| `$(NF-1)` | 倒数第二个字段 |
| `NF` | 当前行的字段数 |
| `NR` | 当前处理到的行号（所有文件累计） |
| `FNR` | 当前文件内的行号 |
| `FS` | 输入字段分隔符 (Field Separator)，默认空白（空格或 Tab，且**多个连续空白算一个**） |
| `OFS` | 输出字段分隔符，默认空格 |
| `ORS` | 输出记录分隔符，默认换行 |
| `RS` | 输入记录分隔符，默认换行 |
| `FILENAME` | 当前文件名 |

```bash
awk '{print $1, $3}' file              # 打印第 1 和第 3 个字段
awk '{print $NF}' file                 # 打印每行最后一个字段
awk '{print NR": "$0}' file            # 加行号
awk -F: '{print $1, $7}' /etc/passwd   # 用 : 作分隔符
awk 'BEGIN{FS=":"; OFS="\t"} {print $1,$3}' /etc/passwd
awk -F'\t' '{print $2}' file.tsv       # Tab 分隔
```

**关于 `FS` 的一个关键细节**：默认的 `FS` 是 `" "`，它有特殊含义——**匹配任意数量的空白（空格和 Tab 混用也行），并且会裁掉行首行尾的空白**。而如果你显式写 `-F' '` 或 `FS=" "`，在 POSIX awk 里仍然有那个特殊行为；但如果你写 `-F'  '`（两个空格）或正则，就变成精确匹配了。所以解析"对齐的表格输出"时，默认行为通常正是你想要的。

### 4.2 模式与动作

```bash
awk '/ERROR/ {print}' file                     # 只打印含 ERROR 的行（print 可省略）
awk '$3 > 100 {print $1, $3}' data.txt         # 第 3 列大于 100
awk '$2 == "active" {c++} END {print c}' f     # 统计
awk '!/^#/ && NF > 0 {print}' config.conf      # 排除注释行和空行
awk 'NR >= 10 && NR <= 20' file                # 行号范围（等价于 sed -n '10,20p'）
awk 'NR % 2 == 0' file                         # 偶数行
```

`BEGIN` 与 `END` 是两个特殊模式：`BEGIN` 在处理任何输入**之前**执行一次（用于初始化、打印表头），`END` 在全部输入处理**之后**执行一次（用于输出汇总）。

```bash
awk 'BEGIN{print "name\tcount"} {c[$1]++} END{for (k in c) print k"\t"c[k]}' file
```

### 4.3 计算：求和、平均、最大

```bash
# 求和
awk '{sum += $3} END {print sum}' data.txt

# 平均值（注意除零保护）
awk '{sum += $3; n++} END {if (n > 0) print sum/n; else print 0}' data.txt

# 最大值及其所在行
awk 'NR==1 || $3 > max {max = $3; line = $0} END {print max, "|", line}' data.txt

# 按第一列分组求和
awk '{sum[$1] += $3} END {for (k in sum) print k, sum[k]}' data.txt

# 每行各字段求和
awk '{s=0; for (i=1; i<=NF; i++) s += $i; print s}' numbers.txt
```

`awk` 的数组是**关联数组 (associative array)**（哈希表），所以 `count[$1]++` 这种"按某列计数"的写法极其方便。遍历用 `for (k in arr)`，**注意遍历顺序是不确定的**（哈希序），需要有序输出时要自己 `| sort`。

### 4.4 条件、格式化与传参

```bash
# if/else 分类
awk '{ if ($3 >= 90) g="A"; else if ($3 >= 80) g="B"; else g="C"; print $1, $3, g }' scores.txt

# printf 精确控制格式
awk '{printf "%-10s %6.2f\n", $1, $2}' data.txt
# %-10s: 左对齐占 10 字符  %6.2f: 宽 6、保留 2 位小数

# 把 shell 变量传进 awk（必须用 -v，不要用单引号插值）
threshold=90
awk -v t="$threshold" '$3 > t {print $1}' scores.txt

# 把一个 shell 变量的值作为 awk 的一部分
name="alice"
awk -v n="$name" '$1 == n {print $0}' file

# 修改分隔符，输出 CSV
awk 'BEGIN{FS=":"; OFS=","} {print $1,$3,$7}' /etc/passwd

# 只保留第一列，并去掉重复（保持顺序）
awk '!seen[$0]++' file
```

最后那条 `awk '!seen[$0]++' file` 是**保持顺序去重**的经典写法：`seen[$0]++` 先返回 0（假）再自增，因此第一次出现时 `!0` 为真，输出该行；之后再出现时返回值非 0，`!` 为假，不输出。它比 `sort -u` 的优势是**不改变行的顺序**。

### 4.5 一个贴近实战的日志解析

假设 `access.log` 是常见的 Combined Log Format（每行大致如下）：

```text
192.168.1.10 - - [12/Mar/2025:10:30:15 +0800] "GET /api/users HTTP/1.1" 200 1234 "-" "Mozilla/5.0"
203.0.113.7  - - [12/Mar/2025:10:30:16 +0800] "POST /api/login HTTP/1.1" 401 88 "-" "curl/8.0"
```

```bash
# 1. 提取所有请求方法与路径（第 6、7 字段）
awk '{print $6, $7}' access.log

# 2. 统计各 HTTP 状态码出现次数，按次数降序
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 3. 统计每个 IP 的请求数，取前 10
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# 4. 计算所有 200 响应的总字节数（第 10 字段）
awk '$9 == 200 {sum += $10} END {printf "total bytes: %d (%.2f MB)\n", sum, sum/1024/1024}' access.log

# 5. 每个路径的平均响应字节数
awk '{bytes[$7] += $10; cnt[$7]++} END {for (p in bytes) printf "%-30s %10.0f\n", p, bytes[p]/cnt[p]}' access.log | sort -k2 -rn | head

# 6. 找出 4xx/5xx 且耗时可疑的请求（这里假设第 10 列是耗时而非字节）
awk '$9 ~ /^[45]/ {print $1, $7, $9}' access.log | head -20

# 7. 统计每小时请求量（从时间字段的 :HH 部分提取）
awk -F'[:[]' '{print $3}' access.log | sort | uniq -c
```

第 7 条用到了多分隔符：`-F'[:[]'` 表示冒号或左方括号都算分隔符（正则字符类）。

## 5. `sort` —— 排序

```bash
sort file                     # 按字典序（默认按整行）
sort -n file                  # 按数值排序（-n 数值，否则 10 会排在 9 前面！）
sort -r file                  # 逆序
sort -u file                  # 排序并去重
sort -k2 file                 # 按第 2 列排序
sort -k2,2n file              # 只按第 2 列、按数值排序
sort -t: -k3,3n /etc/passwd   # 指定分隔符，按第 3 列（UID）数值排序
sort -h file                  # 人类可读的数值（1K < 1M < 1G）
sort -f file                  # 忽略大小写
sort -V file                  # 版本号排序（v1.10 排在 v1.9 之后）
sort -k1,1 -k3,3nr file       # 第一列升序，同组内第三列数值降序
```

**`-k2` 与 `-k2,2` 的区别**（重要的坑）：`-k2` 表示"从第 2 列开始，直到行尾"作为排序键；`-k2,2` 表示"只用第 2 列"。当你想"按第 2 列排序，相同则按原顺序"时，必须写 `-k2,2`，否则会用后面的列做次级比较。

**locale 的影响**：`sort` 的默认排序受 `LC_ALL` / `LC_COLLATE` 影响（例如是否区分大小写、标点怎么排）。这会导致同一个脚本在不同机器上结果不同。需要确定性的字节序排序时：

```bash
LC_ALL=C sort file            # 按字节值排序，忽略 locale，也更快
```

`LC_ALL=C` 在处理日志（含大量 ASCII）时不仅能保证一致性，速度也通常明显更快。

## 6. `uniq` —— 去重（必须配合 sort）

> ⚠️ **`uniq` 只能去掉**相邻**的重复行**。输入没有排序时，不相邻的重复行不会被合并。所以 `uniq` 几乎总是跟在 `sort` 后面。

```bash
sort file | uniq              # 去掉相邻重复（先 sort 才彻底）
sort file | uniq -c           # 加上出现次数（最常用！）
sort file | uniq -d           # 只输出重复出现的行（至少出现 2 次）
sort file | uniq -u           # 只输出只出现 1 次的行
sort file | uniq -i           # 忽略大小写
sort file | uniq -f1          # 忽略前 1 个字段再比较
sort file | uniq -s3          # 跳过前 3 个字符再比较
```

### 6.1 "Top N" 流水线 —— 记住这一条

```bash
sort file | uniq -c | sort -rn | head -10
```

逐段解释：

| 阶段 | 作用 |
| --- | --- |
| `sort file` | 排序，让相同行相邻（`uniq` 的前提） |
| `uniq -c` | 合并相邻重复行，并在前面加上计数 |
| `sort -rn` | 按数值（`-n`）逆序（`-r`）排，让计数最大的排最前 |
| `head -10` | 只取前 10 条 |

这条流水线可以解决大量实际问题：访问量前 10 的 IP、最常见的错误信息、日志里出现最多的 URL、命令历史里最常用的命令……只要把第一部分换成"提取你关心的那个字段"，剩下的三段永远一样。

## 7. `wc` —— 计数

```bash
wc file            # 行数、词数、字节数
wc -l file         # 只数行
wc -w file         # 只数词（以空白分隔）
wc -c file         # 数字节
wc -m file         # 数字符（与 -c 在 UTF-8 中文下差别很大）
wc -L file         # 最长行的长度
```

**`wc -l` 数的其实是换行符个数**，不是"行数"。因此一个结尾没有换行符的文件，实际有 N 行时 `wc -l` 会报 N-1。这是一个真实的坑：

```bash
$ printf 'a\nb\nc' | wc -l
2                       # 有三个"行"但只有两个换行符
$ printf 'a\nb\nc\n' | wc -l
3
```

在脚本里比较行数时要注意这一点。另外 `wc -c`（字节）与 `wc -m`（字符）在 UTF-8 下不同：一个中文字符通常占 3 个字节。

**只取数字的写法**（脚本里常用）：

```bash
count=$(wc -l < file.txt)     # 重定向到 stdin，输出就只有数字，没有文件名
```

如果直接 `wc -l file.txt`，输出会是 `  42 file.txt`，取数字还要额外处理。

## 8. `cut` —— 按列切分

```bash
cut -d: -f1 /etc/passwd              # 用 : 分隔，取第 1 列（用户名）
cut -d: -f1,7 /etc/passwd            # 取第 1 和第 7 列
cut -d: -f2-5 /etc/passwd            # 取第 2 到第 5 列
cut -d: -f3- /etc/passwd             # 第 3 列到末尾
cut -c1-10 file                      # 按字符位置取前 10 个字符
cut -c1-8,20- file                   # 字符范围组合
cut -d: -f1 --complement /etc/passwd # 取除第 1 列之外的所有列
cut -d, -f1 --output-delimiter=' | ' data.csv   # 指定输出的分隔符
```

> ⚠️ **`cut` 的致命限制：`-d` 只接受**单个字符**作分隔符。**面对"以多个空格对齐"的表格输出，`cut -d' '` 会把每一段空白都当成分隔符，得到一堆空字段。

替代方案：

```bash
# 用 awk 的默认空白分隔（多个连续空白算一个）
awk '{print $2}' aligned_table.txt

# 或者用 tr 把连续空白压成一个字符再 cut
tr -s ' ' '\t' < aligned_table.txt | cut -f2

# 或者用 sed 只保留需要的列
```

这是"什么时候该用 awk 而不是 cut"的典型判据。

## 9. `tr` —— 字符级替换与删除

`tr` 只做**单字符到单字符**的映射，不能处理正则。

```bash
tr 'a-z' 'A-Z' < file           # 小写转大写
tr -d '\r' < file               # 删除所有回车（CRLF 转 LF）
tr -s ' '                       # 把连续的多个空格压成一个（squeeze）
tr -s '\n'                      # 把多个连续换行压成一个（去掉空行）
tr -d '[:digit:]' < file        # 删除所有数字
tr ':' '\n' < /etc/passwd       # 把冒号换成换行（每个字段单独一行）
tr -c '[:print:]\n' ' ' < file  # 把不可打印字符替换成空格
```

注意：`tr` **不从文件参数读输入**，必须用 `<` 或管道。

## 10. 组合管道实例

下面每一个都是完整的、可直接对应实际任务的一行命令。建议亲手跑一遍。

### 实例 1：日志里访问量前 10 的 IP

```bash
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
```

输出形状：

```text
    482 192.168.1.10
    317 203.0.113.7
    ...
```

### 实例 2：统计各 HTTP 状态码分布（含百分比）

```bash
awk '{print $9}' access.log \
  | sort | uniq -c | sort -rn \
  | awk '{total += $1; lines[NR] = $0} END {for (i=1; i<=NR; i++) print lines[i]}'
```

更简洁的两趟写法：

```bash
total=$(wc -l < access.log)
awk '{print $9}' access.log | sort | uniq -c | sort -rn \
  | awk -v t="$total" '{printf "%-5s %8d  %6.2f%%\n", $2, $1, $1*100/t}'
```

### 实例 3：找出最大的 10 个文件（含隐藏文件）

```bash
find . -type f -printf '%s\t%p\n' 2>/dev/null | sort -rn | head -10 \
  | awk '{printf "%10.2f MB  %s\n", $1/1024/1024, $2}'
```

`find -printf` 是 GNU 扩展（BSD/macOS 的 find 不支持），跨平台时应改用 `find ... -exec du`。

### 实例 4：从 /etc/passwd 提取"可登录用户"并按 UID 排序

```bash
awk -F: '$7 !~ /(nologin|false)$/ && $3 >= 1000 {print $3, $1, $7}' /etc/passwd | sort -n
```

这个例子体现了 `awk` 相对于 `cut` 的优势：能同时做**条件过滤**和**字段重组**。

### 实例 5：去重同时保留出现次数，输出为 CSV

```bash
sort words.txt | uniq -c | sort -rn | awk '{printf "%s,%d\n", $2, $1}' > counts.csv
```

### 实例 6：把某一列的数字求和

```bash
awk '{sum += $3} END {printf "sum = %.2f\n", sum}' numbers.txt

# 或者用 paste + bc（老式写法）
cut -d, -f3 numbers.csv | paste -sd+ | bc
```

### 实例 7：统计文本中出现最多的 5 个单词

```bash
tr -cs '[:alpha:]' '\n' < article.txt \
  | tr 'A-Z' 'a-z' \
  | sort | uniq -c | sort -rn | head -5
```

`tr -cs '[:alpha:]' '\n'` 的意思是"把**不是**字母的字符（`-c` 取补集）压缩并替换成换行"（`-s`），于是文本被拆成一行一个单词。

### 实例 8：找出两个文件的共同行与差异行

```bash
sort a.txt > a.sorted
sort b.txt > b.sorted
comm -12 a.sorted b.sorted      # 只输出两边都有的行（交集）
comm -23 a.sorted b.sorted      # 只在 a 中的行
comm -3  a.sorted b.sorted      # 只在其中一边的行
```

`comm` 要求两个输入**都已排序**，这是它容易被误用的原因。也可以用 `grep -Fxv -f b.txt a.txt` 求差集。

### 实例 9：从日志里提取时间范围内的错误

```bash
grep -E '^\[?2025-03-12 1[0-2]:' app.log | grep -i -E 'error|fail|exception' | sort -u
```

### 实例 10：批量重命名（预览模式）

```bash
# 先只打印，确认无误后再执行
for f in *.jpeg; do
    echo mv -- "$f" "${f%.jpeg}.jpg"
done

# 确认后去掉 echo 执行
for f in *.jpeg; do
    mv -- "$f" "${f%.jpeg}.jpg"
done
```

**永远先打印再执行**是批量操作的铁律。

## 11. grep / sed / awk 该用哪个

| 任务 | 首选 | 原因 |
| --- | --- | --- |
| 找出含某模式的行 | `grep` | 专为此设计，选项丰富，最快 |
| 统计含某模式的行数 | `grep -c` | 一行搞定 |
| 测试某模式是否存在（脚本） | `grep -q` | 用退出码，不产生输出 |
| 简单替换（尤其 `-i` 改文件） | `sed` | 语法最短 |
| 删除/插入特定行 | `sed` | `d` / `i` / `a` 命令直接对应 |
| 处理"两个标记之间"的区块 | `sed` | `/start/,/end/` 寻址天然支持 |
| 按列提取/重排 | `awk` | 字段模型 + 可计算 |
| 任何求和、平均、计数、分组 | `awk` | 有变量、数组、算术 |
| 条件复杂、需要 if/else | `awk` | 是小语言，不是替换工具 |
| 单纯按固定字符切列 | `cut` | 比 awk 短（但只支持单字符分隔符） |
| 字符级替换/删除 | `tr` | 一行完成大小写、去 CR |
| 排序、去重、Top N | `sort` + `uniq` | 唯一可靠组合 |

一句话总结：**"找行"用 grep，"改行"用 sed，"算列"用 awk**。三者常在同一条管道里协作。

## 小结

- stdout(1) 与 stderr(2) 是分开的；**只有 stdout 会进管道**，所以需要把错误也纳入处理时必须写 `2>&1`。
- `> file 2>&1` 与 `2>&1 > file` 结果不同，因为 `2>&1` 把 fd 2 指向**当时**的 fd 1。前者两者都进文件，后者 stderr 仍到终端。
- `xargs` 把 stdin 变成参数；**文件名可能含空格时必须用 `find -print0` + `xargs -0`**。
- `grep` 的退出码 0=有匹配、1=无匹配、2=出错；脚本里用 `grep -q`，不要用 `grep -c` 加判断。
- `grep -o` 数的是"出现次数"，`grep -c` 数的是"行数"，两者不同。
- `sed -i` 在 GNU 与 BSD 上语法不同（BSD 必须给 `-i` 一个参数）；跨平台时用临时文件 + `mv`。
- `awk` 的价值在于**字段 + 计算 + 关联数组**；`awk '!seen[$0]++'` 是保持顺序去重；`awk -v` 是向 awk 传 shell 变量的正确方式。
- `uniq` 只去除**相邻**重复，所以必须 `sort` 在前；`sort | uniq -c | sort -rn | head` 是通用的 Top N 模板。
- `wc -l` 数的是换行符，末尾无换行的文件会少算一行；脚本里用 `wc -l < file` 只取数字。
- `cut -d` 只支持**单个字符**分隔符；处理多空格对齐的表格必须改用 `awk`。
- `sort -k2` 与 `sort -k2,2` 语义不同；`LC_ALL=C sort` 可以获得确定性的字节序排序并且更快。
- 结论性的一句话：**找行用 grep、改行用 sed、算列用 awk**，三者与 sort/uniq/wc/cut/tr 组合起来，几乎能解决所有日常文本处理任务。
