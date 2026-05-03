# Linux 常用指令笔记

> CentOS 7 环境，面向 IC 验证工作流。按场景组织，不求面面俱到，重点覆盖日常使用频率最高的操作。

---

## Vim 基础操作

Vim 有三种基本模式：**普通模式**（默认，敲命令）、**插入模式**（编辑文本）、**命令模式**（按 `:` 进入，执行保存退出等操作）。

### 模式切换

```
Esc       → 从任何模式回到普通模式
i         → 在光标处进入插入模式
o         → 在下一行插入并进入插入模式
:         → 进入命令模式
```

### 常用命令（普通模式下）

| 操作 | 按键 |
|------|------|
| 保存 | `:w` |
| 退出 | `:q` |
| 不保存强制退出 | `:q!` |
| 保存并退出 | `:wq` |
| 撤销 | `u` |
| 反撤销 | `Ctrl+R` |
| 复制当前行 | `yy` |
| 粘贴 | `p`（光标后） / `P`（光标前） |
| 删除一行 | `dd` |
| 跳转到行首 | `gg` |
| 跳转到行尾 | `G` |
| 查找 | `/关键字` 后按 `n` 下一个，`N` 上一个 |

---

## 文件与目录操作

### 文件管理

`cp`（copy）复制文件或目录。复制目录需加 `-r`（recursive）选项。为防止覆盖已有文件时被反复询问，可用 `\cp` 跳过确认提示。`-i` 选项则会在覆盖前逐一询问。

```bash
cp file1 file2                   # 文件复制
cp -r dir1 dir2                  # 目录递归复制
cp file1 dir1/                   # 复制到目标目录
```

`mv`（move）移动或重命名文件。在同一目录下操作就是重命名，跨目录操作则是移动。

```bash
mv oldname newname               # 重命名
mv file1 dir1/                   # 移动到目录
```

`rm`（remove）删除文件或目录。删除目录需加 `-r`，默认会逐个询问是否确认删除，加 `-f` 可强制删除不询问。**rm 删除后不可恢复，使用 `-rf` 组合时务必确认路径正确。**

```bash
rm file1                         # 删除文件（会询问确认）
rm -f file1                      # 强制删除，不询问
rm -rf dir1/                     # 递归强制删除目录
```

`mkdir` 创建目录，`-p` 选项可一次性创建多层不存在的父目录。这在创建深层目录结构时非常方便，比如 `mkdir -p a/b/c` 会同时创建 a、a/b、a/b/c 三层。

### 目录导航

`ls` 列出目录内容。不加路径时默认为当前目录。常用组合中，`-l` 显示详细信息（权限、大小、修改时间），`-a` 显示隐藏文件（以 `.` 开头的文件），`-h` 将文件大小以人类可读格式显示（K、M、G），`-t` 按修改时间排序。

```bash
ls -l                           # 详细信息（权限、大小、时间）
ls -a                           # 包含隐藏文件
ls -lh                          # 文件大小显示为 K/M/G
ls -lt                          # 按时间排序
```

`cd` 切换目录。不带参数时回到当前用户家目录。`cd -` 回到上一个所在目录，在两个目录间来回切换时非常高效。

```bash
cd /etc/yum.repos.d/            # 进入绝对路径
cd ..                           # 返回上一级
cd ~                            # 回到家目录
cd -                            # 回到上一个目录
```

### 查找文件

`find` 按条件搜索文件。格式为 `find <路径> <条件>`，条件可以是文件名、类型、大小、修改时间等。`-name` 按文件名匹配，`-type f` 只查普通文件，`-type d` 只查目录。`-iname` 忽略大小写。

```bash
find . -name "*.v"              # 当前目录找所有 .v 文件
find / -type f -name "*.swp"    # 全盘找交换文件
find . -name "*.log" -mtime -1  # 过去24小时修改的 .log 文件
```

---

## 文件内容查看与处理

### 查看内容

`cat` 直接输出文件全部内容到终端，适合查看短文件。`head` 和 `tail` 分别查看文件开头和末尾，默认各显示 10 行，`-n` 可指定行数。`tail -f` 是追踪模式，文件有新内容写入时会实时输出，查看仿真日志时非常常用。

```bash
cat file                        # 输出全部内容
head -n 20 file                 # 前 20 行
tail -f sim.log                 # 实时追踪日志更新
tail -n 100 sim.log             # 最后 100 行
```

`less` 分页查看文件，支持上下翻页和搜索。打开后按 `/` 输入关键字向下搜索，按 `?` 向上搜索，`n` 跳到下一个匹配。按 `q` 退出。查看大日志文件时 `less` 比 `cat` 更合适，因为它只加载当前屏幕的内容。

```bash
less large_file.log
```

### 文本搜索与处理

`grep` 按模式（支持正则表达式）搜索文件内容，输出匹配的行。`-r` 递归搜索目录，`-i` 忽略大小写，`-n` 显示行号，`-v` 反向匹配（输出不包含模式的行），`-A`/`-B`/`-C` 同时输出匹配行的后/前/前后若干行。

```bash
grep "error" sim.log            # 搜索关键字
grep -r "module" . --include="*.sv"  # 递归搜索 SV 文件
grep -rn "FIFO" .               # 递归搜索并显示行号
grep -C 5 "fatal" log           # 搜索并显示前后各 5 行
```

`sed`（stream editor）按行处理文本，最常用的场景是替换。基本格式为 `sed 's/原字符串/新字符串/'`。不加 `-i` 时只输出结果不修改原文件，加 `-i` 才原地修改。末尾加 `g`（global）替换一行内的所有匹配，否则只替换每行第一个。

```bash
sed 's/dhcp/static/' file       # 替换（只输出，不修改文件）
sed -i 's/dhcp/static/' file    # 替换并保存到原文件
sed -i 's/foo/bar/g' file       # 全局替换
```

### 统计与格式化

`wc`（word count）统计行数、单词数、字节数。`wc -l` 只统计行数，是最常用的用法。

```bash
wc -l file                      # 文件行数
wc file                         # 行数 单词数 字节数
```

---

## 进程与作业管理

### 查看进程

`ps` 查看当前进程快照。`ps -ef` 显示所有进程的完整信息（PID、父进程、启动时间、命令等），`ps aux` 显示类似信息但格式略有不同。`ps -ef | grep xxx` 是查找特定进程的经典组合。

```bash
ps -ef                          # 所有进程
ps -ef | grep vnc               # 查找 VNC 进程
```

`top` 实时显示系统进程和资源占用，按 `q` 退出，按 `P` 按 CPU 排序，按 `M` 按内存排序。

### 后台运行

在命令末尾加 `&` 可将该命令放入后台执行，终端继续可用。`jobs` 列出当前 Shell 的后台任务及其编号。`fg %编号` 将后台任务调回前台。

`nohup`（no hang up）使命令在终端关闭后继续运行。格式为 `nohup 命令 &`。输出会追加到 `nohup.out` 文件中，也可以用重定向指定输出文件。在长时间仿真场景中，`nohup` 是最常用的方式之一。

```bash
sleep 100 &                     # 后台运行
jobs                            # 查看后台任务
fg %1                           # 调回前台
nohup vcs -full64 top.sv &      # 仿真编译后台跑，退出终端也不中断
```

### 终止进程

`kill` 按 PID 终止进程。默认发送 SIGTERM（15）信号，请求进程正常退出。进程不响应时可加 `-9`（SIGKILL）强制杀死。`killall` 按进程名批量终止。

```bash
kill 1234                       # 终止 PID 1234
kill -9 1234                    # 强制终止
killall vcs                     # 终止所有 vcs 进程
```

---

## 权限管理

### 文件权限

`chmod` 修改文件权限。用三位数字表示（u=所有者, g=所属组, o=其他人），每位数字是 r(4)+w(2)+x(1) 之和。`chmod +x` 不加数字时表示给所有用户增加可执行权限。

```bash
chmod +x script.sh              # 添加可执行权限
chmod 644 file                  # rw-r--r--（所有者读写，其他人只读）
chmod 755 file                  # rwxr-xr-x（所有者完全，其他人只读执行）
```

### 文件归属

`chown` 修改文件的所有者和所属组。格式为 `chown 用户:组 文件`。

```bash
chown zh:zh file                # 将所有者改为 zh，组改为 zh
chown -R zh:zh dir/             # 递归修改整个目录
```

---

## 网络相关

### 网络配置

`ifconfig` 查看和配置网络接口。CentOS 7 中已不是默认安装，但 `yum install net-tools` 可安装。不带参数时列出所有活动接口及其 IP、MAC、收发数据量。

`ip` 是较新的网络配置工具，`ip addr` 查看 IP 地址，`ip link` 查看链路状态。

```bash
ifconfig                        # 查看网络接口信息
ip addr                         # 查看 IP 地址
```

### 网络连通性

`ping` 测试到目标主机的连通性。`Ctrl+C` 停止，否则会一直发下去。

```bash
ping 8.8.8.8                    # 测试外网连通性
ping -c 3 192.168.91.25         # 只发 3 个包
```

### 远程连接

`ssh` 远程登录另一台 Linux 机器。`scp` 在机器之间复制文件，`-r` 复制目录。

```bash
ssh zh@192.168.91.25            # 远程登录
scp file zh@192.168.91.25:~/    # 将文件传到远程目录
scp zh@192.168.91.25:~/file .  # 从远程下载文件
```

### 防火墙

CentOS 7 默认使用 firewalld 管理防火墙。`firewall-cmd` 是配置命令，`--permanent` 使规则持久化，`--reload` 重新加载配置使其生效。

```bash
systemctl status firewalld      # 查看防火墙状态
firewall-cmd --add-port=5901/tcp --permanent  # 放行端口
firewall-cmd --reload           # 重载使配置生效
```

---

## 系统管理

### 包管理

CentOS 7 用 `yum` 管理软件包。常用操作包括安装、卸载、搜索和更新。`yum clean all && yum makecache` 在更换源后需要执行以刷新缓存。

```bash
yum install -y 包名              # 安装（-y 自动确认）
yum remove 包名                  # 卸载
yum search 关键字                 # 搜索包
yum repolist                    # 查看可用源
yum clean all && yum makecache  # 清理并重建缓存
```

CentOS 7 的默认源因已 EOL（End of Life）而失效，需要更换为 vault 源或国内镜像源（如阿里云 mirrors.aliyun.com）。

### 服务管理

`systemctl` 管理系统服务（CentOS 7 及以上）。`enable` 设置开机自启，`disable` 取消自启，`start`/`stop` 立即启动/停止，`status` 查看运行状态。

```bash
systemctl status sshd           # 查看 ssh 服务状态
systemctl start vncserver@:1    # 启动 VNC 服务
systemctl enable vncserver@:1   # 设置开机自启
```

### 磁盘与内存

`df -h` 查看磁盘分区使用情况（`-h` 以 GB/MB 显示），`du -sh 目录` 查看指定目录的总大小，`free -h` 查看内存使用情况。

```bash
df -h                           # 磁盘使用情况
du -sh /home/                   # 目录总大小
free -h                         # 内存使用情况
```

### 用户切换

`su` 切换用户。不加用户名时默认切换到 root。`su - 用户名` 同时加载目标用户的环境变量（`-` 表示 login shell），`su 用户名` 则不加载。

```bash
su                              # 切到 root（需要 root 密码）
su - zh                         # 切到 zh 并加载环境
```

---

## Shell 实用技巧

### Tab 补全

输入命令、路径或文件名时按 `Tab` 自动补全。如果有多个匹配项，按两次 `Tab` 列出所有可能项。这是最常用的效率提升手段，建议养成习惯。

### 命令历史

按 `↑`/`↓` 浏览之前执行过的命令。`history` 列出所有历史记录，每条有编号。`!编号` 可快速重新执行某条历史命令。

```bash
history                         # 查看历史命令
!123                            # 重新执行编号 123 的命令
```

### 管道与重定向

`|`（管道）将一个命令的输出作为下一个命令的输入，串联多个命令形成数据处理链。这是 Linux 中最核心的组合方式。

`>` 将命令输出写入文件（覆盖原有内容），`>>` 追加到文件末尾，不覆盖。`2>` 重定向错误输出（stderr），`&>` 同时重定向标准输出和错误输出。

```bash
grep "error" log | wc -l        # 统计 log 中 error 出现次数
ps -ef | grep vnc               # 查找 VNC 进程
make > build.log 2>&1           # 编译输出（含错误）全部保存到文件
echo "hello" >> file            # 追加内容到文件末尾
```

### 通配符

`*` 匹配任意多个字符，`?` 匹配单个字符。在批量操作文件时非常实用。

```bash
ls *.sv                         # 列出所有 .sv 文件
rm *.log                        # 删除所有 .log 文件
ls test_?.v                     # 匹配 test_1.v, test_a.v 等
```

### 别名

`alias` 为长命令设置简短别名。不加参数时列出当前所有别名。临时生效，退出终端消失。要永久生效需写入 `~/.bashrc`。

```bash
alias ll='ls -lh'               # 设置别名
alias g='gvim'                  # gvim 缩写
unalias g                       # 取消别名
```

---

## 特殊文件路径说明

| 符号 | 含义 |
|------|------|
| `.` | 当前目录 |
| `..` | 上级目录 |
| `~` | 当前用户的家目录（root 为 /root，普通用户为 /home/用户名） |
| `/` | 根目录（文件系统最顶层） |
| `-` | `cd -` 中表示上一个所在目录 |

### 家目录的注意点

root 用户的家目录是 `/root/`，普通用户（如 zh）的家目录是 `/home/zh/`。`~` 始终指向当前登录用户的家目录。跨用户操作时（比如 root 用户用 `~` 操作 zh 的文件），需要明确写出完整路径 `/home/zh/` 而非 `~`。

---

## .vimrc 配置参考

```vim
syntax on
set hlsearch
set nu
set tabstop=4
set expandtab
set shiftwidth=4
set softtabstop=4
set autoindent
set smartindent
set backspace=indent,eol,start
set nocompatible
```

将上述内容保存到 `~/.vimrc` 后，重启 vim 或执行 `:source ~/.vimrc` 即可生效。
