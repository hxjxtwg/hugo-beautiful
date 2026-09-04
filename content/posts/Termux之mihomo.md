---

title: "Termux之mihomo"

author: "xxsky"

type: "posts"

date: 2026-09-04T17:59:22+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

mihomo内核安装、配置、使用

<!--more-->

### 一、在 Termux 中原生部署内核，我们将直接获取官方的 ARM64 架构文件，并将其接入你现有的 PM2 守护体系中。

1.下载与解压内核

环境准备
```
# 呼出你之前写好的快捷指令，接通代理网络
proxy

mkdir -p ~/.config/mihomo && cd ~/.config/mihomo

# 自动抓取官方最新版本号 (例如 v1.18.9)
VERSION=$(curl -s https://api.github.com/repos/MetaCubeX/mihomo/releases/latest | grep '"tag_name":' | head -n 1 | awk -F '"' '{print $4}')
echo "获取到最新内核版本: $VERSION"

# 下载最新版内核包
curl -L -o mihomo.gz "https://github.com/MetaCubeX/mihomo/releases/download/${VERSION}/mihomo-linux-arm64-${VERSION}.gz"

# 解压、赋予权限并验证
gunzip -c mihomo.gz > mihomo
chmod +x mihomo
rm mihomo.gz
./mihomo -v
```
如果最后一步正确输出了包含 Mihomo 和版本号的信息，说明核心准备完毕。

2.拉取并修改配置文件

准备鉴权参数

2.1由于你目前终端依然挂着代理，直接通过命令将云端的订阅配置拉取下来：
```
# 将引号内的链接替换为你的真实订阅链接
curl -A "Clash/1.0" -o config.yaml "https://sub.hxjx.hidns.co/xu"

# 进入编辑器修改配置
nano config.yaml
```
在文件的最顶端（如果已有同名参数则覆盖），强行写入这两行网络安全控制端点：
```
# 面板控制与鉴权
external-controller: '0.0.0.0:9090'
secret: '你的复杂密码'

# 开启物理隔离多端口
listeners:
  - name: bt-port
    type: mixed
    port: 7892
```
按 Ctrl + O 保存，回车确认，按 Ctrl + X 退出。

2.2可以直接脚本自动拉取与更新订阅并配置好

每次覆盖更新都会导致本地安全配置丢失。在完全脱离了图形客户端的 Linux 环境中，解决这个问题的标准做法是编写一个自动化拼接脚本，让机器代替你完成“下载 -> 注入 -> 重启”的全套流程。

请按照以下步骤，在内核目录创建一个智能更新脚本：
第一步：一键生成更新脚本
在终端直接复制并运行以下整段代码（必须先把 你的复杂密码 换成你真实设置的密码）。这会在内核目录下生成一个名为 update_sub.sh 的脚本文件。
```
cat << 'EOF' > ~/.config/mihomo/upsub.sh
#!/bin/bash
cd ~/.config/mihomo

curl -s -A "clash.meta" -o temp_sub.yaml "https://sub.hxjx.hidns.co/xu"

if ! grep -q "proxies:" temp_sub.yaml; then
    echo "❌ 获取配置失败，保留原配置。"
    rm temp_sub.yaml
    exit 1
fi

sed -i '/^external-controller:/d' temp_sub.yaml
sed -i '/^secret:/d' temp_sub.yaml

cat << 'INJECT' > config.yaml
external-controller: '0.0.0.0:9090'
secret: 'xxsky1127'
listeners:
  - name: bt-port
    type: mixed
    port: 7892
INJECT

# 这里是核心：直连规则必须排在 IN-NAME 的上方！
cat << 'PTRULES' > pt_rules.txt
  - DOMAIN-KEYWORD,tracker,DIRECT
  - DOMAIN-KEYWORD,m-team,DIRECT
  - DOMAIN-KEYWORD,longbt,DIRECT
  - IN-NAME,bt-port,🔮 专用下载
PTRULES

sed -i '/^rules:/r pt_rules.txt' temp_sub.yaml
rm pt_rules.txt

cat temp_sub.yaml >> config.yaml
rm temp_sub.yaml
pm2 restart mihomo-core
echo "🎉 更新成功，PT 已直连，BT 数据已隔离走代理！"
EOF
```
第二步：赋予脚本执行权限
刚才只是把剧本写好了，现在需要给它运行的权利。执行一次这行命令：
```
chmod +x ~/.config/mihomo/upsub.sh
```
第三步：以后如何更新？
从现在开始，无论你以后在云端的 mini.ini 怎么折腾节点、怎么改规则，只要你想让挂机手机同步最新配置，你只需要在终端敲这一行命令：
```
~/.config/mihomo/upsub.sh
```
脚本会在一秒钟内，自动去云端把最新的节点拉下来，把原配置里可能会冲突的参数删掉，强制把你专属的 7892 端口和控制面板密码“钉”在文件最顶端，最后自动让 PM2 重启内核。你再也不需要手动去 nano 任何东西了。

3.斩断旧代理并无缝移交 PM2：

后台守护启动

现在所有的“原材料”已经全部部署在你的 .config/mihomo 目录下了。在终端执行 unproxy 关闭终端的环境变量。最关键的一步： 在手机后台彻底划掉/强制停止 Flclash，释放被霸占的 9090 端口。将原生的无头内核挂载到你的自动化常驻体系中：
```
pm2 start ./mihomo --name "mihomo-core" -- -d ~/.config/mihomo
pm2 save
pm2 logs mihomo-core
```
当你看到日志中输出 RESTful API listening at 0.0.0.0:9090，说明这台设备已经完成向完美无头服务器的终极进化了。在同一局域网的浏览器访问 [https://yacd.haishan.me](https://yacd.haishan.me) 即可接管后台。

### 二、mini.ini文件

```
[custom]
custom_proxy_group=✔️ 节点选择`select`[]♻️ 自动选择`[]DIRECT`.*
custom_proxy_group=♻️ 自动选择`url-test`.*`http://www.gstatic.com/generate_204`300,,5
custom_proxy_group=🔮 专用下载`url-test`(专用X)`http://www.gstatic.com/generate_204`900,,50
custom_proxy_group=白天电报池`url-test`(TS)`http://www.gstatic.com/generate_204`300,,50
custom_proxy_group=晚间电报池`url-test`(美国线路)`http://www.gstatic.com/generate_204`300,,50
custom_proxy_group=🔯 电报下载`select`[]白天电报池`[]晚间电报池

; 1. 物理端口强控规则 (只要是从 7892 端口进来的流量，强制走 BT 专用下载组)
custom_rule=IN-PORT,7892,🔮 专用下载

; 2. 专属与直连规则
ruleset=🔯 电报下载,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/Telegram.list
ruleset=DIRECT,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/LocalAreaNetwork.list
ruleset=DIRECT,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ChinaDomain.list
ruleset=DIRECT,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ChinaCompanyIp.list

; 3. 常规应用分流 (目标全部改为 ✔️ 节点选择)
ruleset=✔️ 节点选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/YouTube.list
ruleset=✔️ 节点选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/Google.list
ruleset=✔️ 节点选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ProxyMedia.list
ruleset=✔️ 节点选择,https://raw.githubusercontent.com/ACL4SSR/ACL4SSR/master/Clash/ProxyLite.list

; 4. 常规兜底 (所有未命中规则的剩余流量，走 ✔️ 节点选择)
ruleset=✔️ 节点选择,[]FINAL

enable_rule_generator=true
overwrite_original_rules=true
```

### 三、附定时任务
```
cat << 'EOF' | crontab -
# 1. 每天凌晨 05:00 自动从云端拉取最新订阅并重启内核 (避免打扰夜间下载)
0 5 * * * bash ~/.config/mihomo/upsub.sh > /dev/null 2>&1

# 2. 每天早上 06:00 准时恢复为白天测速池 (带 API 鉴权密码)
0 6 * * * curl --noproxy "*" -s -X PUT "http://127.0.0.1:9090/proxies/🔯%20电报下载" -H "Authorization: Bearer xxsky1127" -H "Content-Type: application/json" -d '{"name": "白天电报池"}' > /dev/null

# 3. 每天晚上 20:00 准时切换到夜间专线池 (带 API 鉴权密码)
0 20 * * * curl --noproxy "*" -s -X PUT "http://127.0.0.1:9090/proxies/🔯%20电报下载" -H "Authorization: Bearer xxsky1127" -H "Content-Type: application/json" -d '{"name": "晚间电报池"}' > /dev/null
EOF
```
```
执行完毕后，你可以输入 crontab -l 检查一下。如果终端输出了这三行任务，就说明系统已经完美接管了。

至此，这两台设备已经彻底化身为免维护的智能网络中枢：

* 7890 端口负责家里的普通手机、电脑，全自动分流且支持手动接管。

* 7892 端口负责死磕 qBittorrent 专属流量，物理隔离保护 Cloudflare 节点。

它们会在每天你熟睡时自己更新配置，并在早晚准点切换最高效的 Telegram 线路。你只需把它们插在充电器上，以后再也不用管了。