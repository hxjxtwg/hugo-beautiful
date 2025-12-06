---

title: "Docker Compose部署服务与caddy反代"

date: 2025-12-03T21:26:38+08:00

lastmod: 2025-12-03T21:26:38+08:00

draft: false

categories: ["技术"]

tags: ["技术"]

author: "xxsky"

description: "全Docker Compose部署服务"

\# cover: /images/default.jpg

---


全Docker Compose部署服务
<!--more-->


## docker环境安装
### 1. Docker 官方一键安装脚本:
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```
### 2. 验证安装：
```bash
docker version
```
```bash
docker compose version
```
## 反代配置
### 1. 为域名添加A或AAAA记录

### 2.一键配置Caddyfile
```bash
cat > Caddyfile <<EOF
# Komari面板的反向代理配置
km.hxjx.hidns.co {
    reverse_proxy komari:25774
}

# 哪吒面板的反向代理配置
nz.hxjx.hidns.co {
    reverse_proxy nezha:8008
}

# Openlist的反向代理配置
openlist.hxjx.hidns.co {
    reverse_proxy openlist:5244
}

# Typecho博客的反向代理配置
blog.aibomart.com {
    reverse_proxy typecho:80
}
EOF
```
## 安装与配置
### 1. 一键生成docker-compose.yml
```bash
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

  nezha:
    image: ghcr.io/nezhahq/nezha:latest
    container_name: nezha
    ports:
      - "8008:8008"
    volumes:
      - ./nezha_data:/dashboard/data 
    restart: unless-stopped
    
  openlist:
    image: openlistteam/openlist:latest
    container_name: openlist
    user: '0:0'
    ports:
      - "5244:5244"
    environment:
      - TZ=Asia/Shanghai
      - UMASK=022
      - OPENLIST_ADMIN_PASSWORD=xxsky1127 # 请务必修改
    volumes:
      - ./oplist_data:/opt/openlist/data 
    restart: unless-stopped

  komari:
    image: ghcr.io/komari-monitor/komari:latest
    container_name: komari
    ports:
      - "25774:25774"
    environment:
      - TZ=Asia/Shanghai
      - ADMIN_USERNAME=admin
      - ADMIN_PASSWORD=xxsky1127
    volumes:
      - ./komari_data:/app/data 
    restart: unless-stopped
  
  # Typecho 博客服务
  typecho:  
    image: joyqi/typecho:nightly-php8.2-apache  # 官方 Apache 镜像
    container_name: typecho
    ports:
      - "8383:80"  # 宿主机 8080 -> 容器 80
    environment:
      TZ: Asia/Shanghai  # 设置时区为上海
    volumes:
      - ./typecho/app/usr:/app/usr  # 当前目录存放 Typecho 文件
    depends_on:
      - db  # 依赖数据库
    restart: always  # 自动重启策略

  # 数据库服务
  db:  
    image: mariadb:10.6  # MariaDB 镜像
    container_name: typecho-db
    environment:
      MYSQL_ROOT_PASSWORD: xxsky1127  # 数据库 root 密码（请修改）
      MYSQL_DATABASE: typecho  # 默认数据库
      MYSQL_USER: typecho  # 数据库用户
      MYSQL_PASSWORD: xxsky1127  # 用户密码（请修改）
      TZ: Asia/Shanghai  # 时区
    volumes:
      - ./db:/var/lib/mysql  # 数据库数据存放当前目录
    restart: always

networks:
  default:
    name: app_network
EOF
```
### 2. 启动服务
```bash
docker compose up -d
```
### 3. 停止服务
```bash
docker compose down
```
### 4. 设置./oplist_data文件夹的权限，确保 Openlist 容器可以正常读写
```bash
sudo chown -R 1001:1001 ./oplist_data
```
### 5. 启动服务
```bash
docker compose up -d
```
### 6. 查看服务
```bash
docker compose ps  
```
## 其它操作
### 1. 查看日志
```bash
docker compose logs komari  # 查看komari，其它相应的服务名
```
### 2. 停止服务
```bash
docker compose stop komari  # 停止komari，其它相应的服务名
```
### 3. 删除数据 
```bash
sudo rm -rf ./komari_data   # 删除komari数据，其它相应的数据名
```
### 4. 重启服务
```bash
docker compose up -d komari  # 重启komari，其它相应的服务名
```
