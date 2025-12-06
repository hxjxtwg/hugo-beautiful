---

title: "Warp自动重连脚本部署指南"

date: 2025-12-03T21:25:29+08:00

lastmod: 2025-12-03T21:25:29+08:00

draft: false

categories: ["技术"]

tags: ["技术"]

author: "xxsky"

description: ""

\# cover: /images/default.jpg

---



<!--more-->

## 1. warp安装
首次运行
```bash
wget -N https://gitlab.com/fscarmen/warp/-/raw/main/menu.sh && bash menu.sh
```
日常维护
```bash
warp
```
手动重连
```bash
wg-quick down warp &>/dev/null; sed -i "s/Endpoint.*/Endpoint = engage.cloudflareclient.com:4500/" /etc/wireguard/warp.conf; warp o
```
## 2. Warp 自动重连脚本部署
本指南将帮助您在 VPS 上部署一个脚本，以自动修复 Warp 的 IPv4 出口掉线问题
#### 2.1 创建并编辑脚本文件
首先，在您的 VPS 上创建一个名为 reconnect_warp.sh 的文件
```bash
nano reconnect_warp.sh
```
打开编辑器后，将下面的完整脚本代码复制并粘贴进去。粘贴完成后，按 Ctrl + X，然后按 Y，最后按 Enter 保存并退出。

完整脚本代码：
```bash
#!/bin/bash

# Warp 自动重启脚本
# 脚本会检查 IPv4 网络连通性，如果发现网络不通，就会自动修复 Warp。

# 尝试 ping 一个可靠的 IPv4 地址，只发送一个数据包，超时 3 秒。
ping -c 1 -w 3 8.8.8.8 > /dev/null 2>&1

# $? 不为 0 表示 ping 失败，执行修复逻辑。
if [ $? -ne 0 ]; then
    echo "$(date): IPv4 network connection failed, repairing Warp..."

    # 强制 Warp 隧道关闭
    wg-quick down warp &>/dev/null

    # 关键修复：使用 | 作为分隔符，确保 Endpoint 被正确替换为标准的 WireGuard 格式。
    sed -i 's|Endpoint.*|Endpoint = engage.cloudflareclient.com:4500|' /etc/wireguard/warp.conf

    # 重新连接 Warp
    warp o
else
    echo "$(date): IPv4 network connection is active. No action needed."
fi
```
或者一键脚本创建reconnect_warp.sh
```bash
cat > /root/reconnect_warp.sh <<'EOF'
#!/bin/bash

# Warp 自动重启脚本
# 脚本会检查 IPv4 网络连通性，如果发现网络不通，就会自动修复 Warp。

# 尝试 ping 一个可靠的 IPv4 地址，只发送一个数据包，超时 3 秒。
ping -c 1 -w 3 8.8.8.8 > /dev/null 2>&1

# $? 不为 0 表示 ping 失败，执行修复逻辑。
if [ $? -ne 0 ]; then
    echo "$(date): IPv4 network connection failed, repairing Warp..."

    # 强制 Warp 隧道关闭
    wg-quick down warp &>/dev/null

    # 关键修复：使用 | 作为分隔符，确保 Endpoint 被正确替换为标准的 WireGuard 格式。
    sed -i 's|Endpoint.*|Endpoint = engage.cloudflareclient.com:4500|' /etc/wireguard/warp.conf

    # 重新连接 Warp
    warp o
else
    echo "$(date): IPv4 network connection is active. No action needed."
fi
EOF
```
#### 2.2 赋予脚本执行权限
接下来，您需要让系统知道这个文件是一个可执行的脚本。在终端中运行以下命令：
```bash
chmod +x reconnect_warp.sh
```
#### 2.3 将脚本添加到定时任务 (Cron)
将脚本添加到定时任务 (Cron)
```bash
crontab -e
```
在打开的文件末尾，新开一行并添加以下命令。这会使脚本每 10 分钟自动运行一次
```bash
*/5 * * * * /root/reconnect_warp.sh
```
或
```bash
*/10 * * * * /bin/bash /root/reconnect_warp.sh
```
完成以上步骤后，即使您的 Warp 再次掉线，系统也会自动检测并修复它，您无需再手动干预。
### 3. 测试
3.1 手动断开warp连接
```bash
wg-quick down warp
```
3.2 运行脚本测试自动重连
```bash
/bin/bash /root/reconnect_warp.sh
```



