---

title: "Termux之vivo Phone"

author: "xxsky"

type: "posts"

date: 2026-09-02T19:05:18+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

termux不休眠网络不断的设置
<!--more-->

### 一、在电脑上执行 USB ADB 的步骤
1.下载电脑端 ADB 工具
在电脑浏览器访问 Android 开发者官网下载 SDK Platform-Tools（免安装包）。将其解压到电脑的任意位置（例如 D:\adb）。

2.手机开启 USB 调试
在 vivo 手机上进入 设置 -> 系统管理 -> 开发者选项。关闭“无线调试”，打开 USB 调试。然后用数据线将手机连上电脑。

3.在电脑端执行命令
打开解压好的 platform-tools 文件夹，在地址栏输入 cmd 并回车，打开电脑的命令行终端。
依次输入以下命令并回车：

3.1检查连接：
```
adb devices
```
（此时手机屏幕会弹出一个“允许 USB 调试”的确认框，必须点击允许，电脑上才会显示 device）

3.2解除 Wi-Fi 休眠限制：
```
adb shell settings put global wifi_sleep_policy 2
```
3.3将 Termux 加入系统深度休眠绝对白名单：
```
adb shell dumpsys deviceidle whitelist +com.termux
```
3.4关闭安卓底层的幽灵进程杀手（防止多子进程被杀）：
你的 pm2 跑了 qbt、sshd、syncthing 和 vscode，极易触发安卓 12+ 的子进程数量限制，建议一并执行：
```
adb shell settings put global settings_enable_monitor_phantom_procs false
```
执行完这三条命令后，拔掉数据线。这相当于从安卓最底层修改了控制权，即便以后 Termux 放在后台或手机锁屏，网络和进程也绝不会再被 vivo 系统强制挂起了。

4.既然只要一切回前台瞬间就能连上，说明不是进程被杀，也不是休眠断网，而是触发了 OriginOS 最底层的“应用冻结 (App Freezing)”或“后台断流防火墙”。系统强行暂停了后台进程的网络收发。

请直接使用以下三种极端的绕过方法，专门对付这种严格的后台冻结：

方法一：利用“悬浮小窗”伪装前台（最有效、最简单的物理外挂）

只要应用还在屏幕上渲染，系统就绝不会判定它为后台程序。

* 将 Termux 打开，从底部上滑进入多任务（近期任务）界面。

* 点击 Termux 卡片顶部的图标或名称，选择小窗（或悬浮窗）。

* Termux 会变成屏幕上的一个小窗口。你可以把它拖到屏幕边缘隐藏成一个小侧边栏。

* 此时你可以正常锁屏或使用其他软件。因为小窗在底层逻辑中属于“绝对前台”，系统防火墙根本不敢切断它的网络，你的 SSH 和离线下载将稳如泰山。

方法二：通过 ADB 强行授予“无限制后台”权限
之前的白名单只解决了休眠问题，但没有解决运行权限问题。再次连接电脑 ADB，强行修改 AppOps 核心权限栈：
```
adb shell cmd appops set com.termux RUN_IN_BACKGROUND allow
```
```
adb shell cmd appops set com.termux RUN_ANY_IN_BACKGROUND allow
```

强烈建议你直接使用方法一（悬浮小窗）配合锁屏，这是对付目前绝大多数国产安卓系统“杀后台/断网”最简单粗暴的终极解法。

一键查询出口 IP
在 Termux 终端中直接输入以下命令并回车：
```
curl cip.cc
```

```
curl ifconfig.me
```
观察返回的结果：如果你看到的是本地运营商（如电信、联通、移动）的真实 IP 和物理位置，说明完全直连；如果显示的是你的代理节点 IP（如香港、美国），说明流量仍然被 FlClash 劫持了。
