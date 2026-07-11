---

title: "Termux之魔改版openlist"

author: "xxsky"

type: "posts"

date: 2026-07-07T20:08:45+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

主要针对魔改版openlist一些错误解决方法，Go 语言写的程序在解析域名时，会去找系统的 /etc/resolv.conf 文件。老版本能用：因为它很可能是专门为 Termux 编译或打过补丁的，它知道去 Termux 的专属目录（$PREFIX/etc/resolv.conf）找 DNS，

<!--more-->

一、安装步骤
1.新建目录
```
mkdir -p ~/oplist && cd ~/oplist
```
2.把下载的压缩包放入oplist目录里

3.解压缩 
```
tar -zxvf openlist-guangyapan_linux_arm64.tar.gz && chmod +x openlist-guangyapan
```
4.启动 OpenList 服务器
```
./openlist-guangyapan server
```
5.记住上面输出的面板

如果错过可以按 Ctrl + C 停止程序

输入命令：./openlist admin set 123456（这会把密码强行改为 123456）

6.修改端口
在data目录下的config.json文件里把"scheme":"http_port": 5244改成想要端口。

二、解决错误

1.Go 语言写的程序在解析域名时，会去找系统的 /etc/resolv.conf 文件。

老版本能用：因为它很可能是专门为 Termux 编译或打过补丁的，它知道去 Termux 的专属目录（$PREFIX/etc/resolv.conf）找 DNS。

这个魔改版报错：这玩意儿大概率是你直接从网上下载的标准 Linux 编译版。它根本不知道自己运行在 Termux 里！ 它傻乎乎地去系统根目录 /etc/ 找配置，发现找不到，就只能向安卓系统底层求救。而安卓底层（因为你装了 Flclash，即使排除了 Termux，系统全局的 DNS 属性依然被改写了）告诉它：“去问 [::1]:53”。结果就是你看到的：撞墙，连接被拒绝。

因为前面那条 GODEBUG 命令对静态编译的 Go 程序无效，所以原样报错。

你只需要用 termux-chroot 命令把它包裹起来运行即可：

第一步：确保你安装了 proot 扩展（如果没装过的话，执行一次就行）

```
pkg install proot -y
```
第二步：用虚拟环境启动你的魔改版
在原有的启动命令前面，加上 termux-chroot：
```
termux-chroot ./openlist-guangyapan server
```
为什么这招管用？
termux-chroot 会在当前终端启动一个虚拟的文件系统。在这个环境里，这个“瞎子”程序去读取 /etc/resolv.conf 时，Termux 会悄悄把正确的文件递给它。

2.由于这是一个编译好的独立程序，它在建立 HTTPS 加密连接时，需要去系统里找“根证书清单”来核对移动云盘的身份。但在 termux-chroot 虚拟环境里，它没找对 Termux 存放证书的路径，变成了个“脸盲”，不敢信任移动的服务器，于是自己把连接掐断了。

终极解决办法（零污染）
我们依然保持之前的原则：绝不乱改系统配置。既然它找不到证书，我们就直接在启动命令里，把 Termux 自带的合法证书文件“塞”给它。

请按顺序执行以下两步：

第一步、确保 Termux 已安装最新的根证书包
（大多数情况已经自带，执行一下以防万一更新）：
```
pkg install ca-certificates -y
```
第二步、携带证书路径再次启动
在刚才的命令前面，加一个专门的环境变量 SSL_CERT_FILE，强行指路：
```
SSL_CERT_FILE=$PREFIX/etc/tls/cert.pem termux-chroot ./openlist-guangyapan server
```
3.pm2启动

请在 ~/oplist 目录下按以下步骤操作：

```
cd oplist
```
创建一个名为 oplist.sh 的文件：
```
nano oplist.sh
```
将下面这两行代码粘贴进去（先把环境变量导出，再执行启动命令）：

```
#!/bin/bash
export SSL_CERT_FILE=$PREFIX/etc/tls/cert.pem
termux-chroot ./openlist-guangyapan server
```
按 Ctrl + O 保存，回车确认，然后按 Ctrl + X 退出。

必须让这个脚本拥有可执行权限，PM2 才能跑得起来：

```
chmod +x oplist.sh
```
加入 PM2 管理

```
pm2 start ./oplist.sh --name "oplist"
```
保存 PM2 进程状态
```
pm2 save
```

4.调整oplist.sh

```
#!/bin/bash
export SSL_CERT_FILE=$PREFIX/etc/tls/cert.pem

# 让这个程序的网络直接走本地代理，绕开那个瞎眼的自带 DNS
export HTTP_PROXY="http://127.0.0.1:7890"
export HTTPS_PROXY="http://127.0.0.1:7890"

# 直接原生启动，没有任何性能损耗
./openlist-guangyapan server
```

