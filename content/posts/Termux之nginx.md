---

title: "Termux之nginx"

author: "xxsky"

type: "posts"

date: 2026-08-07T17:40:07+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

Termux中安装nginx,实现播放的302重定向。

<!--more-->
### 一、Nginx安装与配置
1.在 Termux 中安装 Nginx
```
pkg update -y
pkg install nginx -y
```
2.替换 Nginx 配置文件

安装完成后，Nginx 的配置文件默认藏在 Termux 的深处。我们需要把里面的内容换成咱们的“劫持分流”规则。

2.1直接在 Termux 里用 nano 编辑配置文件：
```
nano /data/data/com.termux/files/usr/etc/nginx/nginx.conf
```
进入编辑界面后，把里面原有的代码全部删干净（如果你连着蓝牙键盘，狂按删除；或者长按屏幕选全选删除）。

把下面这段我为你写好的终极分流配置，原封不动地粘贴进去：
```
worker_processes  1;
events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;

    # 这里就是你的“新 VIP 车道”
    server {
        listen       8097;
        server_name  localhost;

        # 🚨 核心劫持区：看到 Infuse 请求播放流，立刻拦下！
        # 把它转交给本地 5001 端口的 Python 劫持脚本（咱们下一步来搞定它）
        location ~* /videos/(.*)/(stream|original) {
            proxy_pass http://127.0.0.1:5001;
            # 增加下面这两行，强制把真实的域名传给 Python！
            proxy_set_header Host $http_host;
            proxy_set_header X-Forwarded-Host $http_host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 🟢 普通通行区：海报、菜单、刮削，统统老老实实交给 8096 的 Emby 本尊
        location / {
            proxy_pass http://127.0.0.1:8096;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Protocol $scheme;
            proxy_set_header X-Forwarded-Host $http_host;
            
            # 必须加上这段，否则 Emby 网页端的 WebSocket 会断，导致进度条不同步
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```
粘贴好之后，保存退出（在 nano 里按 Ctrl + X，然后按 Y，最后按 回车）。

2.2或者用其它方法从电脑传递文件然后CP

首先在电脑新建nginx.conf，然后粘贴如上代码保存。通过openlist上传至termux根目录下，再运行如下操作。

把根目录的配置，强行覆盖到 Nginx 真正的老巢去：
```
cp ~/nginx.conf $PREFIX/etc/nginx/nginx.conf
```
核心大招：一键清洗 Windows 的隐形换行符（\r）：
```
sed -i 's/\r$//' $PREFIX/etc/nginx/nginx.conf
```
3.启动并测试 Nginx

```
# 检查配置文件语法有没有错
nginx -t
```
如果输出显示 syntax is ok 和 test is successful，说明完美！

接着启动 Nginx：
```
nginx
```
(如果以后修改了配置，只需要运行 nginx -s reload 重载即可)

验收成果
现在，你的交通枢纽已经建好了！
你可以把你手机穿透工具（Cloudflared 或 frpc）里原本指向 8096 的端口，改成 8097。

你现在去外网用浏览器打开你的域名，你会发现：网页能完美打开 Emby，海报墙刷得飞快！
这就证明 Nginx 已经成功把正常流量代理给 Emby 了，而且你完全感觉不到 Nginx 的存在。

你先走到这一步，测试一下海报墙能不能正常刷出来。确认没问题了跟我说，我们来写最后那个跑在 5001 端口的 极简 Python 302 劫持脚本，彻底打通 Infuse 的任督二脉！

### 二、把启动服务加入PM2进程管理

Nginx 的默认性格是“深藏功与名”。一启动，它就会自己潜入系统后台（这叫 Daemon 守护进程），把控制权交还给终端。
但是 PM2 是个强迫症保安，它必须一直盯着进程的肉身（前台运行）。如果 Nginx 潜入了后台，PM2 就会以为它死了！然后 PM2 就会疯狂地去拉起它，拉起又潜入后台，PM2 又以为它死了……最终导致 Nginx 无限重启报错

完美解决方案（只需 3 步）
要让 PM2 乖乖接管 Nginx，我们只需要在启动时加一个参数：daemon off;，强迫 Nginx 站在前台让 PM2 盯着。

1.杀掉目前在后台偷偷跑的 Nginx（清理战场）：
```
pkill nginx
```
2.让 PM2 用“前台模式”接管 Nginx：
```
pm2 start "nginx -g 'daemon off;'" --name nginx
```
(注意看引号，外面是双引号，里面是单引号，原封不动复制进去执行)

3.保存 PM2 的进程列表（防重启丢失）：
```
pm2 save
```
以后如果改了 Nginx 的配置文件，你再也不用去敲原生的 Nginx 命令了，直接用 PM2 统一管理：

重启 Nginx： pm2 restart nginx

看 Nginx 日志： pm2 logs nginx

停止 Nginx： pm2 stop nginx

你的整个家庭影院底层架构，到这一步已经彻底完成了“大一统”，完美闭环！