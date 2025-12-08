---

title: "Vps拾建节点相关总结"

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


