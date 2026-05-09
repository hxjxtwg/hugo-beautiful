---

title: "Termux之PM2监工管理"

author: "xxsky"

type: "posts"

date: 2026-04-22T17:52:13+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

PM2监工模式能够启动服务、自动重启、日志查看等

<!--more-->

### 一、安装“监工” (PM2)
1.Termux 终端里，直接复制运行下面这两行命令（安装 Node.js 环境和 PM2）：
```
pkg install nodejs -y
```
```
npm install -g pm2
```
2.彻底改造start.sh
```
cat > ~/start.sh << 'EOF'
#!/bin/bash
echo "🚀 正在唤醒矩阵服务 (修正版)..."

# 1. 启动 OpenList (补上 server 参数)
cd ~/openlist && pm2 start ./openlist --name "openlist" -- server

# 2. 启动 Cloudflare 隧道 (修正参数传递语法)
pm2 start cloudflared --name "cloudflared" -- tunnel --protocol http2 --config /data/data/com.termux/files/home/.cloudflared/config.yml run emby_tunnel

# 3. 启动 FRP 穿透
pm2 start /data/data/com.termux/files/home/frp_0.56.0_linux_arm64/frpc --name "frpc" -- -c /data/data/com.termux/files/home/frp_0.56.0_linux_arm64/frpc.toml

# 4. 启动 Aria2
pm2 start aria2c --name "aria2" -- --conf-path=/data/data/com.termux/files/home/.config/aria2/aria2.conf

# 5. 启动 Python 脚本 (使用全路径更稳)
pm2 start ~/tg_bridge.py --name "tg_bridge" --interpreter python
pm2 start ~/auto_189.py --name "auto_189" --interpreter python

# 6. crond
pkill -9 crond  # 先杀掉旧的防止锁死
crond
echo "✅ 定时任务 (crond) 已重置启动"

# 保存状态
pm2 save

echo "==================================================="
echo "🎉 矩阵全面复活！所有服务已纳入不死守护！"
echo "👉 输入 'pm2 list' 可查看所有服务运行状态。"
echo "👉 输入 'pm2 logs' 可查看所有服务的实时报错日志。"
echo "==================================================="
EOF

chmod +x ~/start.sh
```
这样改完之后，你获得了什么神级体验？
绝对的不死之身： 如果 Cloudflare 隧道因为网络波动断开了（导致进程退出），或者你的 Python 脚本报错闪退了，PM2 会瞬间捕获，然后自动执行重启，根本不需要你手动去管。

极简的启动方式： 你只需要运行一次 ./start.sh。因为最后有一句 pm2 save，以后如果你手机重启了，或者 Termux 被强退了，你重新打开 Termux，只需要输入一句 pm2 resurrect，所有的服务就会瞬间按原样全部复活，连 ./start.sh 都不用再敲了。

抓虫（排错）神器： 以前你排查日志要去各个文件里找（比如 tunnel.log、run.log）。现在，你只需要敲一句 pm2 logs，所有服务的报错和运行日志会像监控屏幕一样花花绿绿地排在你面前，谁出了问题一目了然。

你先把旧的服务全部杀掉（或者直接重启一下 Termux），然后跑一次新版的 ./start.sh，输入 pm2 list 看看那张华丽的进程监控表，你就知道这套方案有多爽了！

3.功能说明
PM2 （黑心监工模式）
你用 PM2 启动服务，就像是你不仅雇了工人，还雇了一个不睡觉的“黑心监工”死死盯着他们。

自动复活（最核心的区别）： 如果 Cloudflare 因为网络波动自己断开退出了，或者 Python 脚本报错闪退了。PM2 发现工人倒下了，会在 1 秒钟之内自动把他“抽醒”，强行让他重新开始干活！ 根本不需要你插手，服务自己就恢复了。

永不罢工： 只要 Termux 软件还没被系统彻底杀掉，你的服务就永远不会因为报错而彻底停机。
除了“不死之身”，PM2 还有这三大吊打 nohup 的爽点：
对比一：你想看服务有没有在跑？

nohup： 你得输入恶心的 ps -aux | grep cloudflared，在一堆乱码里找进程号。

PM2： 你敲一句 pm2 list，屏幕上直接弹出一个排版极其整齐的表格。哪个服务在线（绿色的 online），哪个服务重启了几次，消耗了多少内存，一目了然。

对比二：你想看报错日志？

nohup： 你得满世界找日志文件，一会儿 cat tunnel.log，一会儿 cat run.log，找起来极其痛苦。

PM2： 你敲一句 pm2 logs，所有服务的实时输出和报错，会用不同颜色按时间顺序在屏幕上滚动。排错极其舒服。

对比三：你想单独停掉某个服务（比如只停掉 TG 机器人）？

nohup： 你得先用 ps 查到进程号，然后小心翼翼地 kill -9 进程号，搞不好还杀错了。

PM2： 你敲一句 pm2 stop tg_bridge 就行了。想再开，敲 pm2 start tg_bridge。随用随停。

总结
你以前的脚本没问题，只是在安卓这种极其恶劣的后台环境里，nohup 过于脆弱，它只管“生”，不管“养”。引入 PM2，就是为了给你的矩阵服务套上一件“复活甲”，彻底告别那种需要你频繁手动去擦屁股、重跑脚本的日子。

怎么样，现在搞明白这个“监工”的含金量了吗？要不要在你的 Termux 里敲一下 pm2 list 看看效果？

4.进程清理
```
pm2 delete all
```
5.启动步骤

第一次启动
```
./start.sh
```
以后只需
手动一键复活
当你重新打开 Termux 时，只需要输入这一句：
```
pm2 resurrect
```
6.彻底自动化（开机即用）

如果你连 pm2 resurrect 都不想打，想让 Termux 一启动就自动复活所有服务，可以把这行指令写进 Termux 的启动配置文件里：

6.1.编辑配置文件：
```
nano ~/.bashrc
```
6.2.在文件的最末尾，另起一行加上：
```
# 自动复活 PM2 管理的服务
pm2 resurrect
# 启动定时任务
crond
# 锁定 CPU 唤醒，防止息屏断网
termux-wake-lock
```
6.3.按 Ctrl + O 保存，Enter 确认，CTRL+X -> Y -> 回车。 退出。

6.4.在vscode终端有提示：
```
The program crond is not installed. Install it by executing:
 pkg install cronie
```
可以直接选择安装：
```
pkg install cronie -y
```


7.日常维护常用指令

既然用了“正规军”管理，这几个指令建议记一下，排查问题非常方便：

查看谁在跑： pm2 list （看状态是否是绿色的 online）。

看报错日志： pm2 logs （如果哪个服务没连上，这里会实时显示报错原因）。

单独重启某项： pm2 restart frpc （只重启 FRP，不影响别的）。

停止某项： pm2 stop auto_189 （临时关掉追剧脚本）。

为了让你更直观地理解 PM2 是如何保护你的“矩阵”不掉线的，你可以通过下面的模拟器体验一下“自动重启”和“保存复活”的逻辑。

8.PM2一键管理所有服务,重新配置

8.1.先清掉当前卡死的 PM2（这一步逃不掉）

必须先把那个坏掉的管家和损坏的管道清理掉，才能重新开始。
```
pkill -9 node
pkill -9 pm2
rm -rf ~/.pm2
```
8.2 VS Code 起草“总名单”

现在 PM2 空了，为了方便你写配置，咱们先临时手动把网页版 VS Code 跑起来（让它在后台跑）：
```
nohup code-server --bind-addr 0.0.0.0:8088 --auth none >/dev/null 2>&1 &
```
去电脑浏览器打开 http://你的手机IP:8088，在左侧的资源管理器里，新建一个文件，命名为 ecosystem.config.js。

8.3编写你的专属配置表（复制修改）

把你所有的服务像下面这样分门别类地填进去。我已经把你目前的 4 个核心服务写好了框架，你只需要检查一下 Python 脚本的绝对路径对不对：
```
module.exports = {
  apps: [
    {
      name: "sshd",
      script: "/data/data/com.termux/files/usr/bin/sshd",
      args: "-D",
    },
    {
      name: "vscode",
      script: "/data/data/com.termux/files/usr/bin/code-server",
      args: "--bind-addr 0.0.0.0:8088 --auth none",
    },
    {
      name: "auto_189",
      script: "/data/data/com.termux/files/home/tcloud/auto_189.py", // ⚠️请确认这里是你脚本的真实路径
      interpreter: "python" // 告诉 PM2 用 Python 运行它
    },
    {
      name: "nginx_302,
      script: "/data/data/com.termux/files/home/tcloud/nginx_302.py", // ⚠️请确认劫持脚本的真实路径
      interpreter: "python"
    }
    // 以后还有别的服务，照着这个格式往下加就行！
  ]
};
```
写完后，按下 Ctrl + S 保存这个文件。

8.4见证“一键点将”的奇迹！

直接让 PM2 照着你的“总名单”干活：
```
pm2 start ecosystem.config.js
```
再敲：
```
pm2 save
pm2 list
```
搞定！ 这时候你再看 pm2 list，你会发现你的 sshd、vscode、auto_189 等等全都在列表里，整整齐齐，全是绿色的 online！

💡 这个方法最爽的地方在于：
以后你不管加多少个服务，只需要在 VS Code 里打开 ecosystem.config.js 这个文件加上几行代码。遇到任何 PM2 抽风崩溃，你直接 rm -rf ~/.pm2 删光，然后重新 pm2 start ecosystem.config.js，所有服务一秒钟满血复活！这才是真正的服务器管理艺术！快去试试！