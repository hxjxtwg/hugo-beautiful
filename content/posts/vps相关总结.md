---

title: "Vps相关总结"

date: 2025-12-03T21:27:16+08:00

lastmod: 2025-12-03T21:27:16+08:00

draft: false

categories: ["技术"]

tags: ["技术"]

author: "xxsky"

description: "快速搭建节点"

\# cover: /images/default.jpg

---

快速搭建节点

<!--more-->


## 1. 安装unzip
1.1 对于 Debian / Ubuntu / Armbian 系统 (最常见):
```bash
apt-get update && apt-get install -y unzip
```
1.2 对于 CentOS / RHEL / Fedora 系统:
```bash
yum install -y unzip
```
1.3 对于 Alpine Linux 系统:
```bash
apk add unzipz
```
## 2. vps安装节点示例
2.1 勇哥agsbx
[github](https://github.com/yonggekkk/argosbx "agsbx")
```bash
hypt="" tupt="" vmpt="" uuid="" argo="y" agn="域名" agk="token" bash <(curl -Ls https://raw.githubusercontent.com/yonggekkk/argosbx/refs/heads/main/argosbx.sh)
```
2.2 老王
[github](https://github.com/eooce/Sing-box "sing-box")

VPS一键四协议安装脚本
```bash
bash <(curl -Ls https://raw.githubusercontent.com/eooce/sing-box/main/sing-box.sh)
```
ssh综合工具箱一键脚本
```bash
curl -fsSL https://raw.githubusercontent.com/eooce/ssh_tool/main/ssh_tool.sh -o ssh_tool.sh && chmod +x ssh_tool.sh && ./ssh_tool.sh
```
## 3. 哪吒探针
3.1 探针1安装
```bash
curl -L https://raw.githubusercontent.com/nezhahq/scripts/main/agent/install.sh -o agent.sh && chmod +x agent.sh && env NZ_SERVER=92.113.117.22:10851 NZ_TLS=false NZ_CLIENT_SECRET=DIjZMwSkRnqZ40LINw1dMl2fGFT0VzV2 ./agent.sh
```
3.2 探针2安装
```bash
curl -L https://raw.githubusercontent.com/nezhahq/scripts/main/agent/install.sh -o agent.sh && chmod +x agent.sh && env NZ_SERVER=35.212.240.126:8008 NZ_TLS=false NZ_CLIENT_SECRET=liBH1SvrWbcA6p6ZakTBMFNPbgN2wp27 ./agent.sh
```
3.3 探针卸载
```bash
sudo systemctl stop nezha-agent
```
```bash
sudo systemctl disable nezha-agent
```
```bash
sudo rm -f /etc/systemd/system/nezha-agent.service
```
```bash
sudo rm -rf /opt/nezha/agent
```
```bash
sudo systemctl daemon-reload
```
3.4 重装UUID
```bash
curl -L https://raw.githubusercontent.com/nezhahq/scripts/main/agent/install.sh -o agent.sh && chmod +x agent.sh && env NZ_SERVER=92.113.117.22:10851 NZ_TLS=false NZ_CLIENT_SECRET=DIjZMwSkRnqZ40LINw1dMl2fGFT0VzV2 NZ_UUID=931aac24-0982-997c-ed7e-12839b63f870 ./agent.sh
```
## 4. komari探针
4.1 通用安装
```bash
bash <(curl -sL https://raw.githubusercontent.com/komari-monitor/komari-agent/refs/heads/main/install.sh) -e https://km.363689.xyz --auto-discovery 65XYUFUXSTekd98agpiDfW5L
```
4.2 卸载探针
```bash
sudo systemctl stop komari-agent
```
```bash
sudo systemctl disable komari-agent
```
```bash
sudo rm -f /etc/systemd/system/komari-agent.service
```
```bash
sudo rm -rf /opt/komari
```
```bash
sudo systemctl daemon-reload
```