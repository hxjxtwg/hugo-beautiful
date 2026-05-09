---

title: "Termux之内外穿透"

author: "xxsky"

type: "posts"

date: 2026-04-23T19:47:40+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

内外穿透用了两种方法：Cloudflare Tunnel与FRP

<!--more-->

### 一、Cloudflare Tunnel

内外穿透，使用Cloudflare Tunnel方式安装

#### 1. Termux 官方库里已经自带了 cloudflared（Tunnel 的核心程序），所以安装极其简单，不需要折腾脚本。
在 Termux 里执行：
```
pkg update -y && pkg install cloudflared -y
```
#### 2. 让 Tunnel 认领你的域名
装好之后，我们需要让这台手机和你的 Cloudflare 账号绑定，这样它才能接管你的 363689.xyz。
在 Termux 里输入这行命令：
```
cloudflared tunnel login
```
这个时候，Termux 屏幕上会弹出一长串网址（URL）。 ---

你现在的任务：把 Termux 里弹出的那串网址复制下来，丢到你手机或电脑的浏览器里打开。 网页会让你登录 Cloudflare，你选择你的 363689.xyz 域名点授权就行。
#### 3. 创建一条隧道
直接在 Termux 里输入并回车：
```
cloudflared tunnel create emby_tunnel
```
#### 把你的域名绑到隧道上
这一步会自动去 Cloudflare 把 emby.363689.xyz 的解析改成隧道的专用地址，瞬间顶替掉原来那些乱七八糟的IP
```
cloudflared tunnel route dns emby_tunnel emby.363689.xyz
```
给 openlist 绑一个新域名
```
cloudflared tunnel route dns emby_tunnel olist.363689.xyz
```
#### 4. 生成配置文件
现在我们要让这条隧道同时干两份活：既转发 Emby，又转发 openlist。
假设你 openlist 的端口是默认的 5244（如果不是，请把下面代码里的 5244 改成你的实际端口）
```
CRED_FILE=$(ls ~/.cloudflared/*.json | head -n 1)
cat <<EOF > ~/.cloudflared/config.yml
tunnel: emby_tunnel
credentials-file: ${CRED_FILE}
ingress:
  - hostname: emby.363689.xyz
    service: http://127.0.0.1:8096
  - hostname: olist.363689.xyz
    service: http://127.0.0.1:5244
  - service: http_status:404
EOF
```
#### 5. 启动隧道
```
cloudflared tunnel run emby_tunnel
```
#### 6. 静默启动服务
```
nohup cloudflared tunnel run emby_tunnel >/dev/null 2>&1 &
```
或强行让隧道走传统的 TCP 协议（HTTP2）,UDP 容易被系统掐断。
```
nohup cloudflared tunnel --protocol http2 --config ~/.cloudflared/config.yml run emby_tunnel > ~/tunnel.log 2>&1 &
```
```
cd ~/openlist
```
```
nohup ./openlist server >/dev/null 2>&1 &
```
```
jobs
```
#### 7. PM2启动服务
```
pm2 start cloudflared --name "cloudflared" -- tunnel --protocol http2 --config /data/data/com.termux/files/home/.cloudflared/config.yml run emby_tunnel
```
锁定IPV4、tcp、配置文件启动
```
pm2 start cloudflared --name "cloudflared" -- tunnel --edge-ip-version 4 --protocol http2 --config /data/data/com.termux/files/home/.cloudflared/config.yml run emby_tunnel
```
锁定IPV4、tcp、Zero Trust 控制台管理启动
```
pm2 start cloudflared --name "cloudflared" -- tunnel --edge-ip-version 4 --protocol http2 run <你的超长Token>
```

### 二、FRP
### VPS部署服务端 frps

1.登录并下载 FRP
用 SSH 连上vps。在命令行里依次执行下面两句命令（这是下载并解压目前最新的 FRP）：
```
wget https://github.com/fatedier/frp/releases/download/v0.56.0/frp_0.56.0_linux_amd64.tar.gz
tar -zxvf frp_0.56.0_linux_amd64.tar.gz
cd frp_0.56.0_linux_amd64
```
2.修改服务端配置文件
我们现在要编辑 frps.toml 文件。执行：
```
nano frps.toml
```
把里面的内容全部删掉，改成下面这段代码（注意修改你的专属密码）：
```
# 服务端与客户端通信的端口
bindPort = 7000

# 身份验证密码（自己编一个复杂的，比如 88888888）
auth.token = "这里换成你自己的密码"

# 穿透出来的网页访问端口（因为 80 端口可能被占用，咱们用 8080）
vhostHTTPPort = 8080
```

如果屏幕上跳出 frps started successfully，说明成功了！按 Ctrl+C 先停掉它。
咱们让它在后台永远运行，执行这个命令：
```
nohup ./frps -c ./frps.toml >/dev/null 2>&1 &
```
(云端搞定！记住vps公网 IP，下面要用)

3.vps防火墙
以GCP为例

谷歌云的防火墙默认是全部拦截的，你必须去网页后台放行咱们刚才设置的 7000 和 8080 端口。

3.1登录谷歌云网页后台 ➡️ 左侧菜单找 VPC 网络 ➡️ 防火墙。

3.2点击顶部的 创建防火墙规则。

3.3按下面这样填：

* 名称： allow-frp

* 流量方向： 入站

* 对匹配项执行的操作： 允许

* 目标： 网络中的所有实例

* 来源 IPv4 范围： 0.0.0.0/0

* 协议和端口： 勾选 指定的协议和端口，勾选 TCP，在框里输入 7000,8080。

* 点击 创建。
### Terxum部署客户端 frpc
1.下载同样的 FRP
执行命令（如果你的手机是 ARM 架构，链接得换成 arm64，我默认你用的是这个）：
```
wget https://github.com/fatedier/frp/releases/download/v0.56.0/frp_0.56.0_linux_arm64.tar.gz
tar -zxvf frp_0.56.0_linux_arm64.tar.gz
cd frp_0.56.0_linux_arm64
```
2.域名添加A记录

把需要穿透的服务在cloudflare域名中添加A记录,ipv4为vps的IP，并关闭小云朵。

3.修改客户端配置文件
编辑 frpc.toml 文件：
```
nano frpc.toml
```
清空里面的内容，换成下面这段：
```
# 填你谷歌云的公网 IP
serverAddr = "35.212.240.126"
serverPort = 7000

# 必须和刚才云端设置的密码一模一样！
auth.token = "xxsky1127"

# ==================================
# 1. 把本地的OpenList 穿透出去
# ==================================
[[proxies]]
name = "openlist_web"
type = "http"
localIP = "127.0.0.1"
localPort = 5244
customDomains = ["op.363689.xyz"]

# ==================================
# 2. 把本地的 Emby 穿透出去
# ==================================
[[proxies]]
name = "emby_web"
type = "http"
localIP = "127.0.0.1"
localPort = 8096       # 这是 Emby 的默认端口，如果你改过，请换成你的真实端口
customDomains = ["huu.cc.cd"]  # 给 Emby 专属的新域名
```
保存退出（Ctrl+O, 回车, Ctrl+X）。
4.启动本地客户端
```
./frpc -c ./frpc.toml
```
如果屏幕显示 start proxy success，恭喜你，穿透成功了！

如果toml文件格式错误，用电脑传过去再
```
cp ~/storage/downloads/frpc.toml ~/frp_0.56.0_linux_arm64/
```
三、vpn设置
以flclash为例
1.添加 Fake-IP 过滤（防止 DNS 被劫持）
这一步是为了告诉 FlClash：“这两个域名你别瞎管，用我本地真实的 DNS 去解析出谷歌云的真实 IP！”

打开 FlClash，找到 配置 / 代理 / 高级设置（根据你的界面，找到 Fake-IP 过滤 / Fake-ip Filter）。

在里面添加下面这两行：

```
op.363689.xyz
huu.cc.cd
```
(注：如果你想图省事，连带着以后可能加的子域名一起直连，也可以写成 *.363689.xyz 和 *.huu.cc.cd，但我建议精准放行，就填上面那两行。)

### FRPS容器安装 
以GCP为例说明
```
cat > docker-compose.yml <<EOF
services:
  caddy:
    image: caddy:latest
    container_name: caddy
    ports:
      - "80:80"
      - "443:443"
      - "443:443/udp"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./caddy_data:/data 
    restart: unless-stopped

  # 哪吒面板服务
  nezha:
    image: ghcr.io/nezhahq/nezha:latest
    container_name: nezha
    ports:
      - "8008:8008"
    volumes:
      - ./nezha_data:/dashboard/data 
    restart: unless-stopped

  # openlist服务  
  openlist:
    image: openlistteam/openlist:latest
    container_name: openlist
    ports:
      - "5244:5244"
    environment:
      - TZ=Asia/Shanghai
      - OPENLIST_ADMIN_PASSWORD=xxsky1127 # 请务必修改
    volumes:
      - ./oplist_data:/opt/openlist/data 
    restart: unless-stopped
  
# Typecho 博客服务
  typecho:  
    image: joyqi/typecho:nightly-php8.2-apache
    container_name: typecho
    ports:
      - "8383:80"
    environment:
      TZ: Asia/Shanghai
    volumes:
      # 重点：挂载到 /var/www/html，这样配置文件才会存在你的硬盘上
      - ./typecho_data:/var/www/html
    depends_on:
      - db
    restart: always

  # 数据库服务
  db:  
    image: mariadb:10.6
    container_name: typecho-db
    environment:
      MYSQL_ROOT_PASSWORD: xxsky1127
      MYSQL_DATABASE: typecho
      MYSQL_USER: typecho
      MYSQL_PASSWORD: xxsky1127
      TZ: Asia/Shanghai
    volumes:
      # 重点：指向你那个存有 ibdata1 的 db 文件夹
      - ./db:/var/lib/mysql
    restart: always

  # ==========================================
  # 🌟 新增：FRP 服务端 (用于内网穿透 Emby) 🌟
  # ==========================================
  frps:
    image: snowdreamtech/frps:0.56.0
    container_name: frps
    ports:
      - "7000:7000" # 只暴露底层通讯端口给手机
    volumes:
      - ./frps.toml:/etc/frp/frps.toml # 挂载我们写好的 FRP 配置文件
    restart: unless-stopped

networks:
  default:
    name: app_network
EOF
```
```
cat > Caddyfile <<EOF
{
    # 使用 Google Trust Services (GTS)
    acme_ca https://dv.acme-v02.api.pki.goog/directory
    acme_eab {
        key_id b33a63ed1f543dc239aeb8ee19968fa4
        mac_key da4XKFaLRa9-oURHGEB8yf2i7rawv0ozDJK99nEkYbkDJdkBAR-fGWmdSLz4EpUtAHUq4ADNFC53EgKhKxLkBg
    }
    email xuhxjxhk@gmail.com
}

# 哪吒面板的反向代理配置
mb.hxjx.hidns.co {
    reverse_proxy nezha:8008
}

# 哪吒面板的反向代理配置2
nzha.cc.cd {
    reverse_proxy nezha:8008
}

# Openlist 的反向代理配置
oplist.cc.cd {
    reverse_proxy openlist:5244
}

# Openlist 的反向代理配置2
oplist.hxjx.hidns.co {
    reverse_proxy openlist:5244
}

# Typecho博客的反向代理配置
tych.cc.cd {
    reverse_proxy typecho:80
}

# Typecho博客的反向代理配置2
tych.hxjx.hidns.co {
    reverse_proxy typecho:80
}


# Emby、openlist、后台管理域名套上HTTPS
# 可以直接把两个域名写在一起，用逗号隔开
huu.cc.cd, op.363689.xyz, on.363689.xyz, xxsky.cc.cd {
    # 全部丢给 FRP 的 8080 接收端
    reverse_proxy frps:8080
}
EOF
```
```
cat <<EOF > /root/frps.toml
bindPort = 7000
vhostHTTPPort = 8080
auth.method = "token"
auth.token = "xxsky1127"
EOF
```
```
docker compose up -d
```
