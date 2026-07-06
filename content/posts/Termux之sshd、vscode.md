---

title: "Termux之sshd、vscode"

author: "xxsky"

type: "posts"

date: 2026-04-22T19:24:45+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

Termux中安装openssh、网页版的vscode

<!--more-->

### 一、openssh
1.安装工具

在 Termux 里输入：

```
pkg install openssh
```
2.设置密码

```
passwd
```
然后按提示输入两次密码（输入时屏幕不显示字符，输完按回车）。

3.查询连接信息

3.1用户名

```
whoami
```

记下返回的字符串（通常是 u0_aXXX 这种格式）

3.2本机 IP
```
ifconfig
```
在 wlan0 那一项里找到 inet 后面的数字（比如 192.168.1.10）。

4.启动服务
```
sshd
```
5.如果曾经安装过或报错缺失配置文件需下面的方法解决：

5.2彻底卸载并连根拔起配置残留
```
apt purge openssh -y
```
注意这里用的是 purge 而不是 remove

5.3全新安装 openssh
```
pkg install openssh -y
```
5.4重新设置你的连接密码
```
passwd
```
5.5见证奇迹的时刻（测试配置）
```
sshd -t
```
(如果这行敲完，屏幕什么都没输出，直接跳到了下一行命令提示符 ~ $，那就说明配置文件成功生成了！)

6.重新让 PM2 接管
```
pm2 start sshd --name "sshd" -- -D
pm2 save
pm2 list
```
或者
```
pm2 start /data/data/com.termux/files/usr/bin/sshd --name "sshd" -- -D
```
7.ssh连接
```
ssh u0_a1443@192.168.0.117 -p 8022
```
有了 SSH： 你可以直接在 Windows 的 PowerShell 或者命令行里敲一句 ssh u0_aXXX@192.168.x.x -p 8022 连进去，敲个 pm2 restart vscode

### 二、部署“网页版 VS Code” (code-server)

既然微软官方的 SSH 插件水土不服，咱们服务器玩家有更高级的玩法：直接在 Termux 里安装一个原生的网页版 VS Code！

这东西叫 code-server，它能在你手机上启动一个一模一样的 VS Code 界面，你只需要在电脑浏览器里输入手机的 IP，就能直接在网页里写代码了，丝滑无比，没有任何兼容性问题！

1.安装扩展源和网页版 VS Code
```
pkg install tur-repo -y
pkg install code-server -y
```
2.让 PM2 把网页版 VS Code 也管起来

我们让它运行在 8088 端口，并且关闭密码验证（因为都是你自己家局域网用）：
```
pm2 start code-server --name "vscode" -- --bind-addr 0.0.0.0:8088 --auth none
pm2 save
```
在电脑上见证奇迹
回到你的 Windows 电脑，不要打开 VS Code 软件，而是打开你的 Chrome 或 Edge 浏览器。
在网址栏输入你手机的 IP 加上 8088 端口（根据你截图里的 IP）：

3.在电脑上见证奇迹

回到你的 Windows 电脑，不要打开 VS Code 软件，而是打开你的 Chrome 或 Edge 浏览器。
在网址栏输入你手机的 IP 加上 8080 端口（根据你截图里的 IP）：
```
http://192.168.0.117:8088
```
4.安装出错

破局三部曲：调虎离山
第一步：暂时关闭 NekoBox
去你的手机里，把 NekoBox 的代理连接暂时断开（点击停止）。让手机完全恢复到正常的 WiFi 或移动数据状态。一般更换节点能解决。

第二步：重新执行安装命令
回到 Termux，因为刚才 tur-repo 其实已经装上一半了，咱们直接更新并安装网页版 VS Code。依次输入这两行：
```
pkg update -y
pkg install code-server -y
```
第三步：重新开启 NekoBox
等屏幕上跑完一堆代码，不再报错并回到 ~ $ 提示符后，就说明安装大功告成了。此时你再去把 NekoBox 重新打开，恢复咱们之前布下的路由大阵。

让 PM2 接管
```
pm2 start code-server --name vscode --interpreter bash -- --bind-addr 0.0.0.0:8080
```
如果启动后日志有错误改成下面的带双引号
```
pm2 start "code-server --bind-addr 0.0.0.0:8088 --auth none" --name "vscode"
```
```
pm2 save
pm2 list
```

### 三、HTTP Server

使用 pm2 来托管 Python 自带的 HTTP Server 是一种非常聪明且常见的做法。它能让你关闭 SSH 终端后，服务依然在后台稳稳运行，还能实现开机自启

```
pm2 start "python3 -m http.server 9999 -d /storage/emulated/0/Download/189cas" --name "fcas"
```
```
pm2 save
pm2 startup
```
