---

title: "Flclash分流"

author: "xxsky"

type: "posts"

date: 2026-08-30T13:25:44+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

订阅配置分流与定时任务

<!--more-->

1.修改订阅后端配置ini文件:
```
[custom]
custom_proxy_group=✔️ 节点选择`select`[]♻️ 自动选择`[]DIRECT`.*
custom_proxy_group=♻️ 自动选择`url-test`.*`http://www.gstatic.com/generate_204`300,,5
custom_proxy_group=🔮 专用下载`load-balance`(专用X)`http://www.gstatic.com/generate_204`300
custom_proxy_group=☀️ 白天电报池`url-test`(TS)`http://www.gstatic.com/generate_204`300,,50
custom_proxy_group=🌙 晚间电报池`url-test`(美国线路)`http://www.gstatic.com/generate_204`300,,50
custom_proxy_group=🔯 电报下载`select`[]☀️ 白天电报池`[]🌙 晚间电报池

; 1. BT 特征规则 (最高优先级拦截)
custom_rule=PROCESS-NAME,qbittorrent-nox,🔮 专用下载
custom_rule=DOMAIN-KEYWORD,tracker,🔮 专用下载
custom_rule=DOMAIN-KEYWORD,torrent,🔮 专用下载

; 2. 专属与直连规则 (必须放前面，防止被下方的大包列表截胡)
ruleset=🔯 电报下载,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/Telegram.list
ruleset=DIRECT,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/LocalAreaNetwork.list
ruleset=DIRECT,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ChinaDomain.list
ruleset=DIRECT,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ChinaCompanyIp.list

; 3. 手机日常主力规则 (包含 YouTube/Netflix 等各类海外应用)
ruleset=♻️ 自动选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/YouTube.list
ruleset=♻️ 自动选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/Google.list
ruleset=♻️ 自动选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ProxyMedia.list
ruleset=♻️ 自动选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ProxyLite.list

; 4. 反转兜底 (剩下的所有毫无特征的未知连接，全部归 BT 负载组)
ruleset=🔮 专用下载,[]FINAL

enable_rule_generator=true
overwrite_original_rules=true
```
2.flclash开启端口控制

工具-基本配置-外部控制器

3.定时任务

既然是在网页版 VS Code 里面操作，浏览器的快捷键经常会和编辑器冲突，我们干脆彻底抛弃文本编辑器。因为你明确表示之前的旧任务不需要了，我们可以用一行命令直接清空旧任务，并把新的切换任务写进去。

第一步：关闭当前卡住的界面
在 VS Code 终端面板的右上角，点击那个垃圾桶图标 (Kill Terminal) 直接强制关掉当前卡住的终端。然后点击 + 号 新建一个干净的终端命令行窗口。

第二步：一键覆盖写入新任务
在新建的终端里，直接全选复制下面这一整段代码，粘贴进去并按下回车键（这会利用 cat 命令直接把纯净的新任务强行覆盖到系统里）：
```
cat << 'EOF' | crontab -
# 每天晚上 20:00 准时切换到夜班优质线路池
0 20 * * * curl --noproxy "*" -s -X PUT "http://127.0.0.1:9090/proxies/🔯 电报下载" -H "Content-Type: application/json" -d '{"name": "🌙 晚间电报池"}' > /dev/null

# 每天早上 06:00 准时恢复为白天常规测速池
0 6 * * * curl --noproxy "*" -s -X PUT "http://127.0.0.1:9090/proxies/🔯 电报下载" -H "Content-Type: application/json" -d '{"name": "☀️ 白天电报池"}' > /dev/null
EOF
```
第三步：验证是否写入成功
粘贴回车后，系统不会有任何提示。你可以直接输入下面这行查询命令来检查：
```
crontab -l
```
如果屏幕上干净利落地只显示了刚才写入的那几行 TG 节点切换代码，说明旧的影视刷新任务已经被成功清理，新的自动化调度也已经完美生效了。

修复后台守护进程卡死报错
你截图最上方暴露了一个致命问题：crond: can't lock /data/data/.../crond.pid, otherpid may be 32038。这说明 Termux 的定时任务守护进程死锁了，如果不清理，到了晚上 20:00 你的自动切换脚本绝对不会被触发。

请直接复制执行以下三行命令，强制清理锁文件并重启进程：
```
killall crond
rm -f /data/data/com.termux/files/usr/var/run/crond.pid
crond
```
crond 是一个单例守护进程。系统通过生成一个 .pid 锁文件来保证全局只能有一个定时管家在工作。当你手动输入了 crond，或者你的环境变量配置文件（如 ~/.bashrc）在你打开新终端时再次触发了该命令，系统发现 PID 为 21757 的进程已经在干活了，就会抛出这个提示拦截你的重复启动请求。

验证守护进程状态
无需进行任何修复。你可以直接运行以下命令，亲眼确认它在后台稳妥挂机：
```
ps aux | grep crond
```
只要输出结果中包含 crond 进程（且进程号包含 21757），就说明它活得好好的。今晚 20:00 的节点自动切换任务绝不会缺席。

消除烦人提示（彻底根治）
如果你每次打开 VS Code 终端都会被这句提示刷屏，说明你的自动启动项写得太生硬了。建议修改你的 Termux 启动配置文件（通常是 ~/.bashrc 或 ~/.profile）：

找到里面单独写着 crond 的那一行。

将其替换为更优雅的静默启动逻辑：
```
pgrep crond > /dev/null || crond
```
这段逻辑会让系统在每次打开终端时先默默检查一次，如果发现管家已经在了就不出声，如果不在才悄悄把它拉起来。这样你的终端界面就能永远保持清爽。

第四步：交由 PM2 终极托管：
你目前是手动拉起 crond 的，一旦设备断电重启，定时管家就没了。既然你的自动化环境已经部署了 PM2 进程守护，把定时任务一并交给它管理是最稳妥的：

停掉刚才手动开启的后台进程：
```
killall crond
```
用 PM2 启动前台守护模式（加 -f 参数）：
```
pm2 start "crond -f" --name "crontab-daemon"
```
保存进程列表供开机自启：
```
pm2 save
```
注意要删掉.bashr里的定时任务