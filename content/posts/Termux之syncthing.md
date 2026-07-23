---

title: "Termux之syncthing"

author: "xxsky"

type: "posts"

date: 2026-07-23T16:54:22+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

局域网文件共享

<!--more-->

### 一、更新系统并安装 Syncthing

1.更新 Termux 的包列表和现有软件：
```
pkg update && pkg upgrade -y
```
2.安装 Syncthing 本体：
```
pkg install syncthing -y
```
3.首次运行与生成配置

在终端中直接输入启动命令：
```
syncthing
```
程序会自动生成密钥和配置文件。当你在终端日志中看到类似 INFO: Syncthing is ready to host core services 或 Access the GUI via the following URL: [http://127.0.0.1:8384/](http://127.0.0.1:8384/) 时，说明启动成功。

4.4.访问 Web 管理界面：
不要关闭 Termux。直接打开手机上的任意浏览器（如 Chrome 或 Edge），在地址栏输入：
```
http://127.0.0.1:8384
```
你将看到 Syncthing 的可视化控制面板。在这里你可以添加需要同步的文件夹，并扫描电脑或其他设备的 ID 二维码进行配对。

### 二、进阶操作指南

1.如何让 Syncthing 在后台持续运行？

如果你在 Termux 中按 Ctrl + C，Syncthing 就会退出。为了让它在后台默默工作：

后台启动： 下次启动时，在命令后加上 & 符号：
```
syncthing &
```
止系统杀后台（重要）： 安卓系统具有严格的电池管理策略。下拉手机通知栏，找到 Termux 的通知，点击 Acquire wakelock（获取唤醒锁）。这样即使手机息屏，Termux 也会保持运行。

2.如何局域网内用电脑访问手机的 Web 界面？
默认情况下，Syncthing 的控制面板只能在本机（手机）浏览器访问。如果你想在电脑的大屏幕上配置手机的 Syncthing：

停止当前的 Syncthing 进程。

编辑配置文件：
```
nano ~/.local/state/syncthing/config.xml
```
或者想确认具体配置文件的确切位置，可以通过这个命令一键查看：
```
syncthing -paths
```
终端会输出包括 config.xml 和数据库在内的所有完整路径。

找到 <gui enabled="true" tls="false" debugging="false"> 下方的这一行：
<address>127.0.0.1:8384</address>

将其修改为：
<address>0.0.0.0:8384</address>

保存并退出（按 Ctrl+O，回车确认，然后 Ctrl+X）。重启 Syncthing 后，你就可以在电脑浏览器中输入 http://手机局域网IP:8384 访问控制面板了。

3.启动并托管进程
在 Termux 中运行以下命令将 Syncthing 交给 PM2 管理：
```
pm2 start syncthing --name "syncthing" -- --no-browser
```
保存进程状态
```
pm2 save
```
