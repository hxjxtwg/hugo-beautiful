---

title: "Termux之openlist"

author: "xxsky"

type: "posts"

date: 2026-04-23T19:40:41+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

openlist的安装与设置

<!--more-->

## 一、openlist的安装
### 1. Termux中删除alist
1.1 如果Termux中原安装了alist先删除

1.2 停止 AList 后台进程
```
pkill -9 alist
```
1.3 查找并删除文件
通常文件夹名就叫 alist。你可以输入 ls 查看当前目录下的文件夹。
```
rm -rf ~/alist
```
你可以现在去 Termux 里执行这几条命令试试。

如果在执行 netstat 或者 pgrep 时依然有输出，说明还有残留进程。

1.4 检查是否还有残留的快捷命令：
```
which alist
```
* 如果返回空行（直接跳到下一行），说明很干净。

* 如果返回了路径（比如 /data/.../bin/alist），请执行 rm 加上那个路径，例如：rm /data/data/com.termux/files/usr/bin/alist。
* 最后确认：
输入 ls 查看当前目录，确认没有 alist 相关的文件夹了。

1.5 输入which alist后输出：/data/data/com.termux/files/usr/bin/which: this version of which' is deprecated; use command -v' in scripts instead.

/data/data/com.termux/files/usr/bin/alist
看来 AList 还没“搬家”干净，它现在正躲在你的系统指令路径里。

那个输出告诉你了两件事：

AList 在哪： 它位于 /data/data/com.termux/files/usr/bin/alist。

小插曲： which 命令在 Termux 里有点老了，建议以后用 command -v，但这不影响咱们现在的操作。
1.6 继续清理
删除系统路径下的 AList：
```
rm /data/data/com.termux/files/usr/bin/alist
```
再次确认：
```
command -v alist
```
如果这次没有任何输出（直接跳回新的一行），那么恭喜你，AList 已经从你的 Termux 里彻底消失了！

### 2. Termux中安装openlist

2.1 你想把 OpenList 安装在哪个文件夹里？ 通常建议建一个专门的文件夹，方便以后管理。你可以直接运行下面这串命令来创建目录并进入：
```
mkdir -p ~/openlist && cd ~/openlist
```
2.2 更新并升级所有软件包
```
pkg update && pkg upgrade -y
```
2.3 使用curl下载（换个更稳的方法）
```
curl -L -O https://github.com/OpenListTeam/OpenList/releases/latest/download/openlist-android-arm64.tar.gz
```
2.4 解压并运行
```
tar -zxvf openlist-android-arm64.tar.gz && chmod +x openlist
```
2.5 如果下载实在太慢可直接浏览器下载
2.5.1 掐断当前下载：
在手机键盘上按下 Ctrl（Termux 工具栏上的按钮）然后按 C。你会看到光标回到了 ~/openlist $。
2.5.2 清理残余：
为了防止文件损坏，先把刚才下载了一半的文件删掉：
```
rm openlist-android-arm64.tar.gz
```

2.5.3 点击这个链接直接下载到手机：https://github.com/OpenListTeam/OpenList/releases/latest/download/openlist-android-arm64.tar.gz（或者把这个链接贴进你的手机浏览器）。
2.5.4 下载完成后，在 Termux 里输入以下命令把文件从手机下载目录移动过来：
```
termux-setup-storage
# 弹出权限请求时点“允许”
cp /sdcard/Download/openlist-android-arm64.tar.gz ~/openlist/
```
2.5.5 验证下载是否成功
不管是哪个方法，下载完后输入：
```
ls -lh
```
2.5.6 解压文件
```
tar -zxvf openlist-android-arm64.tar.gz
```
2.5.7 授予运行权限
```
chmod +x openlist
```
2.5.8 启动 OpenList 服务器
```
./openlist server
```
2.5.9 获取管理员密码
运行命令后，屏幕会飞快地滚动很多日志信息。请在这些信息中仔细寻找下面这一行：

Successfully create admin user, username: admin, password: XXXXXXXX

那个 XXXXXXXX 就是你的随机初始密码。

如果你错过了这一行，或者没记住，别担心：

按 Ctrl + C 停止程序。

输入命令：./openlist admin set 123456（这会把密码强行改为 123456）。

重新输入 ./openlist server 启动。
2.5.10 扫尾工作

既然已经解压成功，那个 47MB 的压缩包就没用了，可以删掉它省点空间：
```
rm openlist-android-arm64.tar.gz
```

2.6加入pm2y启动管理
```
cd ~/openlist && pm2 start ./openlist --name "openlist" -- server
```

### 3.开启 Termux 的 Wake Lock（唤醒锁）

这是最简单也最直接的方法。它会告诉系统：“哪怕锁屏了，也请给我的 CPU 留一口气”。

操作方法： 下拉你的手机通知栏，找到 Termux 的通知。

点击： 点击通知上的 Acquire Wake Lock。

或者在命令行输入：
```
termux-wake-lock
```

### 4. 创建本地strm存放目录

```
# 建立一个在手机下载目录里可见的文件夹
mkdir -p ~/storage/downloads/strm_files
```
### 5. 登陆openlsit,创建strm存储
### 6. 索引搜索
第一次需全面索引一次
### 7. 全局勾选



## 附录
一、 Aria2
1.Aria2 终极完美安装脚本
```
# 1. 杀掉可能在后台带病运行的旧进程，清空旧配置
pkill aria2c
rm -rf ~/.config/aria2

# 2. 重新安装并创建必要的文件夹和文件
pkg install aria2 -y
mkdir -p ~/.config/aria2
mkdir -p /storage/emulated/0/Movies
touch ~/.config/aria2/aria2.session

# 3. 写入完美无瑕的配置文件 (修复了 input-file 错误)
cat > ~/.config/aria2/aria2.conf << "EOF"
# --- 基础设置 ---
dir=/storage/emulated/0/Movies
continue=true
disk-cache=32M

# --- 断点续传与进度保存 ---
input-file=/data/data/com.termux/files/home/.config/aria2/aria2.session
save-session=/data/data/com.termux/files/home/.config/aria2/aria2.session
save-session-interval=60

# --- RPC 控制设置 (给 openlist 用的接口) ---
enable-rpc=true
rpc-listen-all=true
rpc-allow-origin-all=true
rpc-listen-port=6800
rpc-secret=xxsky1127

# --- 核心下载提速优化 ---
max-concurrent-downloads=5
max-connection-per-server=16
split=16
min-split-size=10M

# --- BT/磁力链接专属优化 ---
bt-enable-lpd=true
enable-dht=true
enable-peer-exchange=true
bt-seed-unverified=true
bt-save-metadata=true
EOF

# 4. 启动 Aria2 (以太静默守护模式运行)
aria2c --conf-path=$HOME/.config/aria2/aria2.conf -D

echo "✅ Aria2 完美配置完毕并已在后台稳定运行！"
```
2.在后台绑定 Aria2 (建立通讯)
2.1在你的主力机浏览器里，输入服务器的地址进入 openlist 后台（例如：http://192.168.0.117:5244），并点击底部登录管理后台。

2.2在左侧菜单栏，点击 设置 (Settings)。

2.3在顶部的选项卡里，找到并点击 其他 (Other) 标签页。

2.4往下划，找到关于 Aria2 的设置框，对照着填：

* Aria2 URI：填 http://127.0.0.1:6800/jsonrpc （这代表让它找住在自己这台手机里的 Aria2）。

* Aria2 密钥：填 xxsky1127 （咱们刚才在代码里写好的暗号）。

2.5填完后，一定要点击底部的 保存 (Save)。

(现在，openlist 已经成功连上底层的 Aria2 苦力了！)

3.把手机本地的“电影放映室”挂载出来
虽然 Aria2 知道要把电影下到 /storage/emulated/0/Movies 这个文件夹里，但咱们还得让 openlist 把这个文件夹显示在网页上，你才能看到。

3.1还在 openlist 的管理后台，点击左侧菜单的 存储 (Storage)。

3.2点击 添加 (Add)。

3.3驱动 选择：本机存储 (Local)。

3.4在弹出的设置页面，填这两个最关键的：

* 挂载路径：填 /本地高清影院 （或者随便你起个好听的名字，这是在首页显示的文件夹名）。

* 根文件夹路径：填 /storage/emulated/0/Movies （这是 Aria2 的真实下载老巢，一字不差地复制过去）。

3.5其他的什么都不用管，直接划到最下面点击 添加 (Save)。

4.见证奇迹的终极下载测试！
所有管道都已打通，现在咱们来体验一把“全自动遥控下载”的爽快感：

4.1退出管理后台，回到 openlist 的主页 (Home)。

4.2你会看到首页多了一个名叫 “本地高清影院” 的文件夹，点进去。

4.3在这个文件夹的右下角，你会看到一个 离线下载 (Offline Download) 的按钮（或者类似云朵/箭头的图标）。

4.4点开它，把你在网上找好的任意一个磁力链接（magnet:?xt=...）粘贴进去，点击确定！

接下来会发生什么？
openlist 会立刻把这个链接顺着刚才的暗号发给后台的 Aria2。Aria2 会开始满速疯狂下载。下载完成后，电影文件会自动出现在这个“本地高清影院”的文件夹里！

