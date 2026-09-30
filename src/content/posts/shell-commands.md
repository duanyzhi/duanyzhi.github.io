---
title: SHELL COMMANDS
description: Shell 常用命令、排障和环境配置笔记。
pubDatetime: 2020-08-14T00:00:00Z
tags: [shell, linux]
---

一份自己攒的 shell 速查。按用途分组，头一栏是分类，进来直接找相应的命令。

## 目录

- [一、环境变量与配置](#一环境变量与配置)
- [二、网络](#二网络)
- [三、文件与文本](#三文件与文本)
- [四、磁盘与内存](#四磁盘与内存)
- [五、进程](#五进程)
- [六、后台常驻与定时任务](#六后台常驻与定时任务)
- [七、权限](#七权限)
- [八、系统信息](#八系统信息)
- [九、写 shell 脚本](#九写-shell-脚本)
- [十、杂项](#十杂项)

## 一、环境变量与配置

### 环境变量

让任意目录都能执行某个可执行文件：把它的目录塞进 `PATH`。

```shell
export PATH=/path/to/your/demo:/$PATH
```

取消环境变量：

```shell
unset PATH
```

### ~/.bashrc

终端彩色，以及把提示符里的当前路径改成只显示最后一层目录（`w` 改成 `W`）。

```bash
# 终端彩色
force_color_prompt=yes

if [ -n "$force_color_prompt" ]; then
    if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then
	# We have color support; assume it's compliant with Ecma-48
	# (ISO/IEC-6429). (Lack of such support is extremely rare, and such
	# a case would tend to support setf rather than setaf.)
	color_prompt=yes
    else
	color_prompt=
    fi
fi

# W 表示终端只显示当前目录，把 PS1 这一行的 w 改成 W 就可以了。
if [ "$color_prompt" = yes ]; then
    PS1='${debian_chroot:+($debian_chroot)}\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\W\[\033[00m\]\$ '
else
    PS1='${debian_chroot:+($debian_chroot)}\u@\h:\W\$ '
fi
unset color_prompt force_color_prompt
```

## 二、网络

### 连 WiFi

命令行连无线，`nmcli` 直接一条搞定。改完一般不用重启。

```shell
nmcli dev wifi list   # 先扫一下有哪些网
sudo nmcli dev wifi connect "Infinigence-Staff" password "InfiniAI!"
```

### ssh 服务与 scp

```shell
sudo apt install openssh-server  # 安装 ssh 服务
service sshd start    # 打开 ssh
service sshd status   # 查看 ssh 状态
service sshd stop     # 关闭 ssh

scp -r file name@ip:/root/            # 拷贝 file 到另一台服务器
scp -P port_id -r file name@ip:/path/ # 带端口号时 -P 必须在 scp 后面
```

### ssh 掉线、终端断连怎么办

连着一会儿就断，看不到最新运行状态，分三层解决。

1. 改服务端保活配置。

```shell
# 服务器修改
vim /etc/ssh/sshd_config
# 把下面几项改大
ClientAliveInterval 60  # 每 60s 自动检查一次，越小越好
ClientAliveCountMax 3
MaxStartups 200
TCPKeepAlive yes

# 重启 ssh
sv restart /opt/devmachine/init/service/sshd
```

2. 或者临时登录时带上参数。

```shell
ssh -o TCPKeepAlive=yes -o ServerAliveInterval=60 -o ServerAliveCountMax=5 \
  -p 41967 -L 7860:localhost:7860 root@219.135.xxx.xxx
```

3. 本地电脑上写成配置，省得每次敲。`~/.ssh/config`：

```shell
Host serve_name
  HostName 219.135.xxx.xxx
  Port 41967
  User root
  TCPKeepAlive yes
  ServerAliveInterval 30
  ServerAliveCountMax 10
```

4. 长任务还是丢后台跑，见 [六、tmux](#六后台常驻与定时任务)。

### 登录别人服务器

把自己机器的公钥贴到对方的 `~/.ssh/authorized_keys` 里，之后免密登录。

```shell
ssh-keygen -t rsa            # 生成密钥对（已有就跳过）
cat ~/.ssh/id_rsa.pub        # 把这行的内容贴到对方服务器
```

### ftp

```shell
root:/workspace   # 在 workspace 目录下登录 ftp
ftp 192.168.1.1   # 登录服务器
# input user name and password
ftp> cd /home
ftp> put file   # 把 workspace 目录下的 file 上传到 ftp 的 /home 目录
```

### wget

```shell
wget -q -O a.zip http://www.XXX.zip   # 下载网页文件
# 参数:
-q  # 不显示下载进程
-O  # 定义输出文件名称为 a.zip
```

## 三、文件与文本

### ls

```shell
ls -lh          # 查看文件具体大小，会显示单位 K/M/G
ls -l | wc -l   # 查看文件个数
ls -ltr         # 按照时间顺序排列
ls /path/to/????.jpg  # 列出所有图片前缀是四个字符的文件
ls -d */        # 只列出所有文件夹
```

### find 查找文件

```shell
find dir/ -iname *.gcda   # 找到 dir 下所有以 .gcda 结尾的文件
find dir/ \( -iname *.gcda -o -name "*.gcno" \)  # 同时满足两个后缀的文件
find / -name "xxx"        # 从根目录按名字找

locate file.zip  # 查找文件位置（走索引，快）
ack "str"        # 递归查找某个字符，比 grep 顺手
```

### 删除符合条件的文件

```shell
# 删除当前目录下所有 .abi3.so
find . -name "*.abi3.so" -type f -delete
```

### grep 搜索内容

```shell
grep -r "ACLRT_LAUNCH_KERNEL" /usr/local/Ascend/ascend-toolkit

grep -Hrn "abc" --include \*.cpp
# --include 只在 cpp 结尾文件中查找
# -r 递归查找
# -n 输出时带行号
# -i 忽略大小写
# -H 输出文件名
```

### sed 和 cat

```shell
sed '/^$/d' old.log > new.log   # 去除 old.log 中所有空行并保存到 new.log
echo "" > log  # 向 log 文件的最后一行添加空行
sed '${/^[[:blank:]]*$/d}' old.log > new.log   # 去除 old.log 最后一行的空行
sed -i 'M,Nd' filename  # 删除文件的 M 到 N 行内容

# sed 替换带 '/' 的字符需要在 / 前面加 \ 转义
sed -i 's/\/usr\/local\/python3.7/\/usr\/python3.7/g' file
sed -i 's/abc.*/aaa/g' file  # 把符合 abc*** 的行全部替换为 aaa

cat -n file        # -n 会显示行号
cat log1 >> log2   # 把 log1 的内容追加到 log2 最后
```

向某一行插入字符：

```shell
sed -i "1i # This is test demo." log  # 向 log 第一行插入字符
```

查看文件前几行：

```shell
cat filename.txt | head -n 10
```

在每一行增删内容：

```shell
sed -e 's/$/abc/' -i file    # 在每行末尾插入 abc
sed -i "s/^/\"/g" file      # 在每行开头插入 "
sed -e "s/$/\"/g" file      # 在每行末尾插入 "
sed -e "/patten str/d" file  # 删除符合条件的行
sed -i "/^#/d" file          # 删除所有以 # 开头的行
sed -i 'N' -e 's/(\n/\(/g' file  # 把 ( 加换行替换成 (
```

### awk

```shell
awk '{print $NF}'  // 输出以空格分割最后一个匹配项

awk -F 'name' '{print $2}' file  # 以 name 为分隔符，打印它后面的内容
```

加了下面这个 log：

```tex
  %output.62 : Float(32:5120, 10:512, 512:1) = aten::matmul(%input.121, %2645)
  %outputs.100 : Float(32:5120, 10:512, 512:1) = aten::add_(%output.62, %3527, %2632)
  %x.24 : Float(32:5120, 10:512, 512:1) = aten::add_(%outputs.100, %input.119, %2632)
  %218 : int = prim::Constant[value=1]()
  %2651 : int[] = prim::ListConstruct(%2628)
  %mean.24 : Float(32:10, 10:1, 1:1) = aten::mean(%x.24, %2651, %2629, %2630),
  %2653 : int[] = prim::ListConstruct(%2628)
  %std.24 : Float(32:10, 10:1, 1:1) = aten::std(%x.24, %2653, %2629, %2629),
  %2655 : Float(32:5120, 10:512, 512:1) = aten::sub(%x.24, %mean.24, %2632)
  %2656 : Float(32:5120, 10:512, 512:1) = aten::mul(%3528, %2655)
  %2657 : Float(32:10, 10:1, 1:1) = aten::add(%std.24, %2631, %2632),
  %2658 : Float(32:5120, 10:512, 512:1) = aten::div(%2656, %2657)
  return (%2658)
```

要拿到所有 `aten::**` 的算子名：

```shell
grep "=" log | sed "/prim::Constant/d" | awk -F " = " '{print $2}' | awk -F "(" '{print $1}'|sort -u
# grep "=" log 把所有含有 = 的行都找出来
# sed "/prim::Constant/d"  删除符合条件的行
# awk -F " = " '{print $2}'  按 = 号切开，取第二段，比如
#  %output.62 : Float(32:5120, 10:512, 512:1) = aten::matmul(%input.121, %2645)
#  输出就是 aten::matmul(%input.121, %2645)
# awk -F "(" '{print $1}'  再按 ( 切开，取第一段
# sort -u  sort 排序，-u 去重
```

最后结果：

```tex
aten::add
aten::add_
aten::div
aten::matmul
aten::mean
aten::mul
aten::std
aten::sub
prim::ListConstruct
```

### head、tail、xargs、tr

```shell
head -n 1 File1; tail -n 1 File1  # 选取文件第一行或者最后一行

echo "a b" | xargs mkdir  # 分别创建 a、b 文件夹

tr '\n' ' '  # 用空格替换换行符，即把所有行变成一行
tr -d '"'    # 删除 " 符号

seq 1 10     # 生成 1 到 10 之间的整数
```

### cp、rm、软链接

```shell
cp a.so ./    # 直接拷贝 .so
cp -r a.so ./ # 拷贝软链接文件 .so

rm -- --filename   # 文件名带横杠时这样删
```

拷贝和某个文件同级目录下的所有文件：

```shell
file=/path/to/file/abc.so
cp ${file%/*}/* dir/to/path
```

### rsync

```shell
rsync -av raw/ des/  # 把 raw 文件夹下面的内容拷贝到 des 目录下

# 假如目录如下
bin/
├── ext
│   ├── remove
│   │   └── empty.txt
│   └── test.cpp
└── torch
    ├── test.gcda
    └── test.gcno
# 将 bin 目录下内容拷贝到 dir 目录下，同时不拷贝 remove 目录下内容
rsync -avAX --exclude=*/remove/\* bin/ dir/
```

### tar、dpkg、rpm

```shell
tar -cf md.tar.gz md/  # 压缩 md
tar -xvf md.tar.gz
# 参数:
-v  # 显示进程

dpkg -i pack.deb      # 安装 deb 包
dpkg --remove pack
dpkg -X pack.deb test # 把 deb 文件解压到 test 文件夹
dpkg -L pack_name     # 查看安装包位置

rpm2cpio name.rpm | cpio -i --make-directories  # 解压 rpm 文件
sh pkg.run -list      # 查看 .run 文件
rpm -e packagename    # 卸载 rpm 安装包
```

## 四、磁盘与内存

### 看磁盘大小

```shell
df -h /                          # 看根分区还剩多少

du -ah / | sort -hr | head -30   # 找出最占地方的目录 / 文件
du -ah --max-depth=1             # 看当前目录下每个文件、文件夹占多少
du -sh                           # 只显示总占用，比上面快
```

### 看内存和 buff/cache

`free` 里 free 小不代表不够用，主要看 `available`。buff/cache 是系统拿来缓存磁盘的，随时能回收，一般不用担心。

```shell
free -h

# free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       993Mi       190Mi        67Mi       2.6Gi       2.7Gi
Swap:             0B          0B          0B
```

真觉得 cache 占得难受，可以手动清（正常没必要）：

```shell
sudo sh -c 'echo 3 > /proc/sys/vm/drop_caches'
```

### 清 cache 和孤儿包

```shell
sudo apt-get autoclean   # 清理旧版本的软件缓存
sudo apt-get clean       # 清理所有软件缓存
sudo apt-get autoremove  # 删除系统不再使用的孤立软件
```

### 清旧内核

```shell
dpkg --get-selections | grep linux-image  # 查看已安装内核
uname -a  # 查看电脑当前用的内核
# 删除没用内核：
sudo apt-get remove linux-image-4.13.0-37-generic
sudo apt-get purge linux-image-extra-4.13.0-37-generic
sudo apt-get autoremove linux-image-extra-4.13.0-37-generic
sudo dpkg -r linux-image-extra-4.13.0-38-generic
```

### 服务器切到纯命令行

机器不用图形界面时，切到多用户模式，省内存。

```shell
sudo systemctl set-default multi-user.target
sudo reboot
```

## 五、进程

### 看进程和占用排行

```shell
ps aux --sort=-%mem | head -n 20   # 按内存占用排序，看谁最吃内存
ps aux
ps -eLf          # 查看所有隐藏进程（含线程）
ps -eLf | wc -l  # wc -l 统计行数，也就是进程数
ulimit -u        # 查看系统进程上限

cat /proc/meminfo | head -n 20   # 查看内存详细分布

ps -aux | grep python   # 找到 python 的进程
```

### kill 进程

```shell
kill -9 PID
pgrep -f python | xargs kill -9   # 按命令行关键字批量 kill
```

### ulimit 限制资源

给跑起来的程序套个上限，防止把机器吃干。软限制只对当前 shell 和它启动的进程生效。

```shell
ulimit -v 30000000   # 限制虚拟内存 30GB（单位 KB）
ulimit -a            # 看看当前所有限制
```

### shell 多线程

用 `&` 丢后台加 `wait` 等全部结束。

```shell
cmd_list="ls "
cmd_a="python test.py"
cmd_b="python test.py"
cmd_list="${cmd_list} & eval ${cmd_a} & eval ${cmd_b}"
eval ${cmd_list}
wait
```

## 六、后台常驻与定时任务

### tmux

```shell
apt-get install tmux
tmux  # 输入 tmux 进入

# 所有 tmux 命令在输入前都要先按键盘上 ctrl+b，之后命令才生效。
# 按了之后屏幕上没变化，然后输入：
"   # 将 terminal 上下分频
%   # 将 terminal 左右分频
;   # 切换到上一个窗口
o   # 切换到下一个窗口
{   # 当前窗口和上一个窗口交换位置
}   # 当前窗口和下一个窗口交换位置
q   # 显示窗口编号

# tmux 启动鼠标
# ctrl+B，然后输入 : 进入命令行模式，再输入 : set -g mouse on
# 或者把上面这句写到 ~/.tmux.conf 文件中

# tmux 粘贴和赋值都可以按住 shift 来执行

# 同时按下 ctrl+b+o 交换两个窗口位置
```

### tmux attach：断连后找回现场

terminal 断了、也看不了日志，直接重新 attach 上去就行。进去就是最新的运行状态，还能继续交互。

```shell
tmux ls   # 查看有哪些后台运行的 tmux

# 1: 1 windows (created Wed Jun 10 20:12:23 2026) (attached)

tmux attach -t 1   # 登陆这个 tmux，恢复现场
```

### at / crontab 定时任务

at：某个时间点跑一次。

```shell
date   # 查看时间
echo "hello world" > log | at 20:10   # 在今天 20:10 向 log 文件写入 hello world
```

crontab：周期性跑。

```shell
apt-get install cron
crontab -e  # 打开 crontab
30 10 * * * /path/to/run.sh   # 每天 10:30 跑 run.sh
sudo /etc/init.d/cron start   # 开始运行 cron
sudo /etc/init.d/cron stop    # 停止运行
```

## 七、权限

### 拿 root 权限

```shell
ctrl+alt+F2   # 进入超级终端
ctrl+alt+F7   # 退出超级终端

chmod -R 777 directory/   # 给目录加读写执行权限
sudo su                   # 切超级用户，谨慎
```

### 改文件属主

把某个目录还给当前用户（比如被 root 建过之后各种写不进去）：

```shell
sudo chown -R $USER:$USER /path/to/project
```

## 八、系统信息

### 看机器配置

```shell
lsb_release -a
uname -a    # 查看是 x86_64(64 位)还是 i686(32 位)
cat /etc/*-release
cat /proc/cpuinfo
cat /proc/driver/nvidia/version  # 驱动版本号
cat /proc/meminfo                # 查看内存
free -m                          # 以 MB 为单位查看 ram 大小
```

### 看 .so 文件

```shell
nm -D *.so   # 查看 .so 中定义的 symbol，列出所有动态链接符号

# strip 去掉符号表，版本发布时一般不能暴露符号表，需要去除：
strip libname.so
nm libname.so   # 会显示 no symbols，说明符号已经没了

readelf -V *.so | grep name      # 从 .so 中读取出筛选内容
readelf -D *.so | grep "R.*PATH" # 查看 so 库设置的 rpath，rpath 优先级大于 LD_LIBRARY_PATH
```

## 九、写 shell 脚本

### .sh 常用写法

```shell
$1       # 运行 .sh 时的输入参数
set -e   # 一般放 .sh 最上面，某个命令返回 error 就终止脚本

pushd $DIR  # 进去 DIR 目录
popd        # 退出目录
```

新建一个字符串列表：

```shell
MY_LIST=(
python
cpp
)

for str in "${MY_LIST[@]}"; do
  echo $str
done

NEW_LIST=(${MY_LIST[*]})   # 把 MY_LIST 赋值给另一个变量

echo "num of MY_LIST" ${#MY_LIST[*]}   # 输出 MY_LIST 元素个数
```

判断文件夹 / 文件是否存在：

```shell
if [ -d dir ]; then
   echo "dir is exist"
fi

if [ -f file ]; then
   echo "file is exist"
fi
```

切分字符串：

```shell
str=path/to/name
dir="$(cut -d'/' -f1 <<< "$str")"   # 按 / 切分并取出 f1，第一个字段
echo $dir
# path
```

赋值、自增：

```shell
num=1
num=$((num+1))   # num+1 并重新赋值给 num
```

逐行读文件：

```shell
while IFS= read -r line || [ -n "$line" ]; do
  echo $line   # 把每一行打印出来
done < log     # 读取 log 文件中每一行
```

加函数：

```shell
FUN() {
echo "do something"
}

FUN   # 运行函数
```

正则在 shell 里的用法：

```shell
# log 文件内容如下
echo "This is log file"
dense=<"0xB5508">

# 用正则把 dense=<"0xB5508"> 替换掉
sed -i "s/dense<\"0\([xX]\)\([0-9a-fA-F]*\)\"/dense<\"{{0[xX][0-9a-fA-F]+}}\"/g" log
```

### 给脚本传参

**方法一**

```shell
# script.sh
usage(){
    echo "Usage of install.sh"
    echo "-p  set param"
    echo "-h  print usage"
}

PARAM=""

while getopts "p:h:" opt
do
  echo "all parameters: "$opt
  case $opt in
    p)
    PARAM=$OPTARG;;    # 把参数值给 PARAM 变量
    h)
    usage
    exit 0;;
    ?)
	echo "there is unrecognized parameter."
	exit 1;;
  esac
done

# 使用用法:
sh script.sh -p true

# 这种方式传参 -p 后面必须要加参数内容
```

**方法二**

```shell
# script.sh
PARAM=""

while [[ $# -gt 0 ]]; do
  key="$1"

  case $key in
    -p|--performance)
      PARAM="true"
      shift
      shift;;
    -h|--help)
      exit 0;;
    *)
      echo "unknown option."
      shift;;
  esac
done

# 使用方法:
sh script.sh -p

# 这种方法传参数 -p 后面不需要跟内容
```

输出所有传入的命令行参数：

```shell
#!/bin/bash
echo ${@:2}   # 输出所有的命令行参数

# a.sh 1 2 3 4
# out: 1 2 3 4
```

### 给函数传参

```shell
fun(){
  arg1=$1  # 这里的 $1 获取的不是 shell 脚本的输入，而是函数 fun 的输入
  echo "arg1:"$arg1
}

fun aaa  # 把 aaa 以参数形式送给函数 fun
```

### 模拟字典

```shell
info_list=(
  "name time"
)

for info in "${info_list[@]}"; do
    info_dict=($info)
    name=${net_dict[0]}
    time=${net_dict[1]}
done
```

### eval

```shell
cmd="ls | grep aaa"
eval $cmd   # 执行 cmd 所表示的语句
```

### 脚本开头 /bin/bash

```powershell
#!/bin/bash
# 一般在 script 文件开头加上这一句。目的是统一脚本在不同系统下运行的环境，
# 保证各个系统都用 /bin/bash 跑这个脚本，因为有些系统默认 bash 不是 /bin/bash，
# 会导致运行脚本出错。
```

### 生成随机数

```shell
echo $(((RANDOM % 10) + 1))   # 生成 1 到 10 之间的随机数
```

## 十、杂项

### shortcut

```shell
ctrl+H          # 查看隐藏文件
df              # 查看 boost 分区空间
ifconfig        # 查看 ip
Ctrl+shift++    # 增加字体，ctrl+- 减小字体
history         # 查看历史 terminal 命令
windows+1/2/3   # 切换不同左侧锁定的应用
ctrl+windows+D  # 显示桌面
alt+tab         # 前后两个应用之间切换 / 可以作为双屏之间切换
# 左侧工具栏可以隐藏
```

### windows 和 ubuntu 时间不一样

```shell
sudo apt-get install ntpdate
sudo ntpdate time.windows.com
sudo hwclock --localtime --systohc
```

### ubuntu 解决 home 空间不足

```markdown
1. windows 下下载 [Gparted](https://sourceforge.net/projects/gparted/files/gparted-live-testing/OldFiles/)
2. 制作 U 盘启动工具：
   打开 UltraISO 工具 - 文件 -> 打开（下载的镜像）> 启动 -> 写入硬盘映像 -> 选择 U 盘驱动器 -> 格式化 -> 写入
3. 我的电脑右键管理 - 磁盘管理 - 点击一个有空余的磁盘 - 压缩卷 - 压缩出一部分空余未分配磁盘空间。
4. 重启 - U 盘启动：
   进入 Gparted 界面 -> Default settings - Don't touch keymap -> 选择语言 -> 输入 26（简体中文）-> 输入 0 进入 X-Windows
5. 弹出 Gparted 磁盘界面，显示自己的磁盘分布 -> 选择 home 分区所在的分区：/ -> 右键点击 Resize/Move -> 点击拖动上方内存条
   使得 New Size 最大，即把未分配的空间分配到 home 下。
6. 点击 Resize/Move -> Apply
7. 重启即可
```

### yum

```shell
yum install -y XXXX   # -y 表示安装过程中让选 yes/no 时默认选 yes
```

### hollywood

```shell
sudo apt-get install hollywood
cmatrix
hollywood
sudo apt-get install oneko
oneko
```
