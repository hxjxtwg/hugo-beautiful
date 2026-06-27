---

title: "Termux之emby_linux"

author: "xxsky"

type: "posts"

date: 2026-04-24T16:22:22+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---


Termux中安装emby服务端


<!--more-->

### 第一套系统：赛博保姆（守护原版安卓 App）
这套你已经很熟悉了，我们快速把它在 PM2 里建好档。

1.在 Termux 中创建脚本：
```
nano ~/emby_nanny.sh
```
2.粘贴完整保活代码：
```
#!/bin/bash
while true; do
    if ! curl -s -m 5 http://127.0.0.1:8096 > /dev/null; then
        am start -n com.emby.embyserver/com.emby.embyserver.MainActivity > /dev/null 2>&1
        sleep 20
        pm2 restart frpc cloudflared > /dev/null 2>&1
    fi
    sleep 60
done
```
3.赋予权限并加入 PM2：
```
chmod +x ~/emby_nanny.sh
pm2 start ~/emby_nanny.sh --name "emby_nanny"
pm2 save
```
(目前这套系统已经在跑了，如果你现在想尝试第二套，我们需要先把它暂停：)
👉 立即执行暂停命令： pm2 stop emby_nanny (并且在手机后台把 Emby App 划掉彻底关掉，腾出 8096 端口)

### 第二套系统：不死真神（纯正 Linux 版 Emby）

现在你的手机 8096 端口已经空出来了，我们来感受一下什么叫极致轻量的原生 Linux 部署。

1.安装迷你 Ubuntu 环境
直接在 Termux 里挨个执行下面三行命令：
```
pkg update && pkg upgrade -y
pkg install proot-distro -y
proot-distro install ubuntu
```
2.进入 Ubuntu 并安装 Emby 核心
执行这个命令，你会发现命令行的前缀变了，说明你已经“穿越”进 Ubuntu 里面了：
```
proot-distro login ubuntu
```
进入后，依次执行以下命令（下载可能需要半分钟）：
```
apt update && apt upgrade -y
apt install wget curl -y
wget https://github.com/MediaBrowser/Emby.Releases/releases/download/4.8.8.0/emby-server-deb_4.8.8.0_arm64.deb
dpkg -i emby-server-deb_4.8.8.0_arm64.deb
```
(如果最后一步提示 Errors were encountered 依赖报错，就补一句 apt --fix-broken install -y，它会自动修好。)

3.退出 Ubuntu，写启动脚本
按 Ctrl + D 或者输入 exit 回车，退回到原汁原味的 Termux 环境。
然后我们写个脚本让它后台跑：
```
nano ~/start_emby_linux.sh
```
粘贴这一行代码：
```
#!/bin/bash
proot-distro login ubuntu -- /opt/emby-server/bin/emby-server
```
保存后，给它权限并加入 PM2：
```
chmod +x ~/start_emby_linux.sh
pm2 start ~/start_emby_linux.sh --name "emby_linux"
pm2 save
```
终极玩法：如何在这两套系统中“无缝切换”？
现在，你的 PM2 列表里躺着两位大将：emby_nanny (保姆) 和 emby_linux (真神)。

场景 A：今天网络不好，我想用原生 App 搞硬件转码看番

1.打开 Termux 敲两行命令：
```
pm2 stop emby_linux
pm2 start emby_nanny
```
2.直接去手机桌面上点开 Emby App，剩下的保姆会替你搞定。

场景 B：我这两天出门，不需要转码，我要它 24 小时绝对稳定不死！

1.把手机后台的 Emby App 划掉关闭。

2.打开 Termux 敲两行命令：
```
pm2 stop emby_nanny
pm2 start emby_linux
```
3.搞定！手机熄屏扔在角落里，原生的 Linux 进程会稳如泰山地跑在后台。局域网访问 http://你的手机IP:8096 就能进入它全新的后台配置界面。

（注：Linux 版第一次启动时，里面是空的，你需要去它的网页后台重新把你的电影文件夹挂载上去。挂载路径选根目录下的 /sdcard 就是你手机的内部存储。）

搞定这两套，你这台备用机算是彻底被你榨干每一滴性能了！去试试安装吧，中途遇到什么报错截图发我！

可能出现安装不上的问题
更新代理端口为 FlClash 的 7890（顺水推舟）
既然 FlClash 连上了，咱们就告诉 Termux 新的端口号。直接复制粘贴这四行（全覆盖），敲回车：
```
export ALL_PROXY="http://127.0.0.1:7890"
export all_proxy="http://127.0.0.1:7890"
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
```
然后再跑一次：
```
proot-distro install ubuntu
```
敲代码“手动拔管”（适合不想关窗口）
如果你不想重启 Termux，想在当前窗口直接拔掉代理管道，直接复制并回车运行下面这一行组合拳：
```
unset ALL_PROXY all_proxy http_proxy https_proxy
```

### 安装Douban 插件

第 1 步：进入 Ubuntu 密室
```
proot-distro login ubuntu
```
第 2 步：安装下载和解压工具
```
apt install curl unzip -y
```
第 3 步：直接下载豆瓣插件（这行有点长，直接整行复制）
```
curl -k -L -o douban.zip https://ghfast.top/https://github.com/AlifeLine/Emby.Plugins.Douban/releases/download/V1.3.0/Emby.Plugins.Douban.zip
```

第 4 步：把它解压进 Emby 的核心肚子里
```
unzip -o douban.zip -d /var/lib/emby/plugins/
```
第 5 步：退出密室，并重启 Emby
```
exit
```
再重启：
```
pm2 restart emby_linux
```
豆瓣插件就绝对装好了

还记得咱们最开始装 Emby 的时候，遇到过一次 System has not been booted with systemd 报错吗？原因我当时解释过：Emby 安装包里带着一个“开机自启脚本”，它企图呼叫 Ubuntu 的后台管家（systemd），但咱们 Termux 的精简版 Ubuntu 里根本没有这个管家，所以报错了。

那为什么现在又报这个错了？
因为你刚才那句命令里包含了 apt install unzip。在 Linux 系统里，apt（包管理器）有个强迫症：每次装新东西前，它会去检查以前有没有“没装利索”的软件。它发现了 Emby 的那个开机脚本一直没执行成功，就非要帮它执行一遍，结果自然又撞墙了，顺便把咱们下载插件的流程也给卡断了。

好消息是： 你看截图最上面，unzip 其实已经成功装上了！

既然病根找到了，咱们这次直接把 Emby 那个烦人的开机脚本“物理消灭”，让系统彻底闭嘴，然后顺手把豆瓣插件装好。

既然你现在的提示符是 root@localhost:~#（说明你已经在 Ubuntu 密室里面了），直接一行一行复制执行下面的命令，彻底解决战斗：


物理割除“病根”（一劳永逸）
把那个总是报错的自启脚本删掉，并告诉系统“它已经装好了，别再管了”：
```
rm /var/lib/dpkg/info/emby-server.postinst
dpkg --configure -a
```
(执行完这两句，以后你在这个 Ubuntu 里装任何东西，都绝对不会再跳 Emby 的报错了。)


操作建议：如何彻底让 Termux 版“闭嘴”让路
为了防止你以后重启手机时，Termux 版的 Emby 又被 PM2 自动拉起来抢占 8096 端口，导致你的 APP 版冲突报错，建议你执行以下操作彻底禁用它：

在 Termux 中输入以下命令：

查看名字：pm2 list （看一眼你的 emby 进程到底叫什么名字，假设叫 emby）

停止运行：pm2 stop emby_linux

从列表删除：pm2 delete emby_linux（这一步只是把它从 PM2 的启动列表里删掉，你的数据和文件都还在，以后想用随时能启动）

保存更改：pm2 save （确保重启后它也不会诈尸）

搞定这几步，你就可以安心地享受 APP 版 Emby 带来的高性能了！如果 APP 版设置的时候有任何网络或权限问题，随时喊我。

### 更新emby

1.进入 Ubuntu 并安装 Emby 核心
```
proot-distro login ubuntu
```
2.重新下载并安装（在容器内）
```
wget https://github.com/MediaBrowser/Emby.Releases/releases/download/4.9.3.0/emby-server-deb_4.9.3.0_arm64.deb
```
3.暴力覆盖安装
```
dpkg -i emby-server-deb_4.9.3.0_arm64.deb
```
4.清理报错，安抚系统（必须做）

4.1删掉那个报错的收尾脚本：
```
rm /var/lib/dpkg/info/emby-server.postinst
```
4.2告诉系统：“别管了，直接标记配置完成！”
```
dpkg --configure emby-server
```
5.直接重启你原来的 PM2 进程就行
```
pm2 restart emby_linux
```

### 迁移emby数据

1.旧emby
```
# 1. 停掉旧手机Emby
pm2 stop emby

# 2. 进入 Ubuntu
proot-distro login ubuntu

# 3. 打包数据
cd /var/lib
tar -cvzf ~/emby_soul.tar.gz emby

# 4. 开启临时文件下载服务
cd ~
python3 -m http.server 8888
```
2.新emby
```
# 1. 停掉新手机的空壳 Emby
pm2 stop emby

# 2. 回到新手机的数据目录
proot-distro login ubuntu
cd /var/lib
mv emby emby_bak

# 3. 直接从旧手机拉取过来并解压！（注意IP是你旧手机的IP）
wget -O - http://192.168.0.117:8888/emby_soul.tar.gz | tar -xzf -

# 4. 修复权限（极其重要）
chown -R emby:emby /var/lib/emby
exit

# 5. 重新启动新手机的 Emby
pm2 start emby
```
删除旧emby的备份
```
rm ~/emby_soul.tar.gz
```
查看目录文件
```
ls -lh ~/
```
3.恢复干净的emby
```
proot-distro login ubuntu

# 1. 停掉精神错乱的 Emby
pm2 stop emby

# 2. 清空 Emby 的所有数据库和缓存（让它变回出厂状态）
rm -rf /var/lib/emby/*

# 3. 确保所有权没问题（防万一）
chown -R emby:emby /var/lib/emby
exit

# 4. 全新启动！
pm2 start emby
```
### 三、代理排除Termux后启动方案

1.start_emby.sh
```
#!/bin/bash
proot-distro login ubuntu -- /bin/bash -c "
export HTTP_PROXY=http://192.168.0.199:7890
export HTTPS_PROXY=http://192.168.0.199:7890
export ALL_PROXY=http://192.168.0.199:7890
export http_proxy=http://192.168.0.199:7890
export https_proxy=http://192.168.0.199:7890
export no_proxy=localhost,127.0.0.1,192.168.0.0/16
export DOTNET_GCHeapHardLimit=0x80000000
export DOTNET_GCHeapHardLimitPercent=80
export COMPlus_GCHeapHardLimit=0x80000000
/opt/emby-server/bin/emby-server
"
```
2.flclash开启允许局域网连接

3.启动emby
```
chmod +x ~/start_emby.sh
pm2 start ~/start_emby.sh --name "emby"
pm2 save --force
```