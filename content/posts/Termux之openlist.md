---

title: "Termux之openlist"

author: "xxsky"

type: "posts"

date: 2026-04-23T19:40:41+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

openlist的安装与设置

<!--more-->

## 一、openlist的安装
### 1. Termux中删除alist
1.1 如果Termux中原安装了alist先删除

1.2 停止 AList 后台进程
```
pkill -9 alist
```
1.3 查找并删除文件
通常文件夹名就叫 alist。你可以输入 ls 查看当前目录下的文件夹。
```
rm -rf ~/alist
```
你可以现在去 Termux 里执行这几条命令试试。

如果在执行 netstat 或者 pgrep 时依然有输出，说明还有残留进程。

1.4 检查是否还有残留的快捷命令：
```
which alist
```
* 如果返回空行（直接跳到下一行），说明很干净。

* 如果返回了路径（比如 /data/.../bin/alist），请执行 rm 加上那个路径，例如：rm /data/data/com.termux/files/usr/bin/alist。
* 最后确认：
输入 ls 查看当前目录，确认没有 alist 相关的文件夹了。

1.5 输入which alist后输出：/data/data/com.termux/files/usr/bin/which: this version of which' is deprecated; use command -v' in scripts instead.

/data/data/com.termux/files/usr/bin/alist
看来 AList 还没“搬家”干净，它现在正躲在你的系统指令路径里。

那个输出告诉你了两件事：

AList 在哪： 它位于 /data/data/com.termux/files/usr/bin/alist。

小插曲： which 命令在 Termux 里有点老了，建议以后用 command -v，但这不影响咱们现在的操作。
1.6 继续清理
删除系统路径下的 AList：
```
rm /data/data/com.termux/files/usr/bin/alist
```
再次确认：
```
command -v alist
```
如果这次没有任何输出（直接跳回新的一行），那么恭喜你，AList 已经从你的 Termux 里彻底消失了！

### 2. Termux中安装openlist

2.1 你想把 OpenList 安装在哪个文件夹里？ 通常建议建一个专门的文件夹，方便以后管理。你可以直接运行下面这串命令来创建目录并进入：
```
mkdir -p ~/openlist && cd ~/openlist
```
2.2 更新并升级所有软件包
```
pkg update && pkg upgrade -y
```
2.3 使用curl下载（换个更稳的方法）
```
curl -L -O https://github.com/OpenListTeam/OpenList/releases/latest/download/openlist-android-arm64.tar.gz
```
2.4 解压并运行
```
tar -zxvf openlist-android-arm64.tar.gz && chmod +x openlist
```
2.5 如果下载实在太慢可直接浏览器下载
2.5.1 掐断当前下载：
在手机键盘上按下 Ctrl（Termux 工具栏上的按钮）然后按 C。你会看到光标回到了 ~/openlist $。
2.5.2 清理残余：
为了防止文件损坏，先把刚才下载了一半的文件删掉：
```
rm openlist-android-arm64.tar.gz
```

2.5.3 点击这个链接直接下载到手机：https://github.com/OpenListTeam/OpenList/releases/latest/download/openlist-android-arm64.tar.gz（或者把这个链接贴进你的手机浏览器）。
2.5.4 下载完成后，在 Termux 里输入以下命令把文件从手机下载目录移动过来：
```
termux-setup-storage
# 弹出权限请求时点“允许”
cp /sdcard/Download/openlist-android-arm64.tar.gz ~/openlist/
```
2.5.5 验证下载是否成功
不管是哪个方法，下载完后输入：
```
ls -lh
```
2.5.6 解压文件
```
tar -zxvf openlist-android-arm64.tar.gz
```
2.5.7 授予运行权限
```
chmod +x openlist
```
2.5.8 启动 OpenList 服务器
```
./openlist server
```
2.5.9 获取管理员密码
运行命令后，屏幕会飞快地滚动很多日志信息。请在这些信息中仔细寻找下面这一行：

Successfully create admin user, username: admin, password: XXXXXXXX

那个 XXXXXXXX 就是你的随机初始密码。

如果你错过了这一行，或者没记住，别担心：

按 Ctrl + C 停止程序。

输入命令：./openlist admin set 123456（这会把密码强行改为 123456）。

重新输入 ./openlist server 启动。
2.5.10 扫尾工作

既然已经解压成功，那个 47MB 的压缩包就没用了，可以删掉它省点空间：
```
rm openlist-android-arm64.tar.gz
```

### 3.开启 Termux 的 Wake Lock（唤醒锁）

这是最简单也最直接的方法。它会告诉系统：“哪怕锁屏了，也请给我的 CPU 留一口气”。

操作方法： 下拉你的手机通知栏，找到 Termux 的通知。

点击： 点击通知上的 Acquire Wake Lock。

或者在命令行输入：
```
termux-wake-lock
```

### 4. 创建本地strm存放目录

```
# 建立一个在手机下载目录里可见的文件夹
mkdir -p ~/storage/downloads/strm_files
```
### 5. 登陆openlsit,创建strm存储
### 6. 索引搜索
第一次需全面索引一次
### 7. 全局勾选



## 附录
一、 Aria2
1.Aria2 终极完美安装脚本
```
# 1. 杀掉可能在后台带病运行的旧进程，清空旧配置
pkill aria2c
rm -rf ~/.config/aria2

# 2. 重新安装并创建必要的文件夹和文件
pkg install aria2 -y
mkdir -p ~/.config/aria2
mkdir -p /storage/emulated/0/Movies
touch ~/.config/aria2/aria2.session

# 3. 写入完美无瑕的配置文件 (修复了 input-file 错误)
cat > ~/.config/aria2/aria2.conf << "EOF"
# --- 基础设置 ---
dir=/storage/emulated/0/Movies
continue=true
disk-cache=32M

# --- 断点续传与进度保存 ---
input-file=/data/data/com.termux/files/home/.config/aria2/aria2.session
save-session=/data/data/com.termux/files/home/.config/aria2/aria2.session
save-session-interval=60

# --- RPC 控制设置 (给 openlist 用的接口) ---
enable-rpc=true
rpc-listen-all=true
rpc-allow-origin-all=true
rpc-listen-port=6800
rpc-secret=xxsky1127

# --- 核心下载提速优化 ---
max-concurrent-downloads=5
max-connection-per-server=16
split=16
min-split-size=10M

# --- BT/磁力链接专属优化 ---
bt-enable-lpd=true
enable-dht=true
enable-peer-exchange=true
bt-seed-unverified=true
bt-save-metadata=true
EOF

# 4. 启动 Aria2 (以太静默守护模式运行)
aria2c --conf-path=$HOME/.config/aria2/aria2.conf -D

echo "✅ Aria2 完美配置完毕并已在后台稳定运行！"
```
2.在后台绑定 Aria2 (建立通讯)
2.1在你的主力机浏览器里，输入服务器的地址进入 openlist 后台（例如：http://192.168.0.117:5244），并点击底部登录管理后台。

2.2在左侧菜单栏，点击 设置 (Settings)。

2.3在顶部的选项卡里，找到并点击 其他 (Other) 标签页。

2.4往下划，找到关于 Aria2 的设置框，对照着填：

* Aria2 URI：填 http://127.0.0.1:6800/jsonrpc （这代表让它找住在自己这台手机里的 Aria2）。

* Aria2 密钥：填 xxsky1127 （咱们刚才在代码里写好的暗号）。

2.5填完后，一定要点击底部的 保存 (Save)。

(现在，openlist 已经成功连上底层的 Aria2 苦力了！)

3.把手机本地的“电影放映室”挂载出来
虽然 Aria2 知道要把电影下到 /storage/emulated/0/Movies 这个文件夹里，但咱们还得让 openlist 把这个文件夹显示在网页上，你才能看到。

3.1还在 openlist 的管理后台，点击左侧菜单的 存储 (Storage)。

3.2点击 添加 (Add)。

3.3驱动 选择：本机存储 (Local)。

3.4在弹出的设置页面，填这两个最关键的：

* 挂载路径：填 /本地高清影院 （或者随便你起个好听的名字，这是在首页显示的文件夹名）。

* 根文件夹路径：填 /storage/emulated/0/Movies （这是 Aria2 的真实下载老巢，一字不差地复制过去）。

3.5其他的什么都不用管，直接划到最下面点击 添加 (Save)。

4.见证奇迹的终极下载测试！
所有管道都已打通，现在咱们来体验一把“全自动遥控下载”的爽快感：

4.1退出管理后台，回到 openlist 的主页 (Home)。

4.2你会看到首页多了一个名叫 “本地高清影院” 的文件夹，点进去。

4.3在这个文件夹的右下角，你会看到一个 离线下载 (Offline Download) 的按钮（或者类似云朵/箭头的图标）。

4.4点开它，把你在网上找好的任意一个磁力链接（magnet:?xt=...）粘贴进去，点击确定！

接下来会发生什么？
openlist 会立刻把这个链接顺着刚才的暗号发给后台的 Aria2。Aria2 会开始满速疯狂下载。下载完成后，电影文件会自动出现在这个“本地高清影院”的文件夹里！

二、定时任务
1.安装定时任务服务
```
pkg install cronie -y
```
2.脚本
```
cat > ~/refresh.sh << "EOF"
#!/bin/bash

# 你的真实管理员 Token
TOKEN="openlist-4ba3bee7-2112-42bd-92e7-6f3a7c67a83a7CIOwhmfYpB9Hi8HX1NYfRJUn1iWvyPnyuoOUPC5Fqn5FGxTwNuKHwRSYCT6OZC2"

# 这里你直接写最外层的大目录就行了！不管里面嵌套多少层它都能找到
PATHS=(
    "/177/177-电影"
    "/177/177-动漫"
    "/177/177-电视剧"
)

# 定义一个“无限深度探测”的核心引擎
scan_dir() {
    local CURRENT_PATH="$1"
    echo "🚀 正在刷新并探测: $CURRENT_PATH"
    
    # 敲门获取当前目录内容
    local JSON=$(curl -s -X POST "http://127.0.0.1:5244/api/fs/list" \
    -H "Authorization: $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"path\":\"$CURRENT_PATH\",\"password\":\"\",\"page\":1,\"per_page\":0,\"refresh\":true}")
    
    # 找出当前目录下的所有子文件夹，存起来（完美解决名字带空格的问题）
    local dirs=()
    while IFS= read -r line; do
        if [ -n "$line" ]; then
            dirs+=("$line")
        fi
    done < <(echo "$JSON" | jq -r '.data.content[]? | select(.is_dir==true) | .name')
    
    # 如果里面还有文件夹，就挨个钻进去继续挖！
    for folder in "${dirs[@]}"; do
        sleep 2 # 停顿2秒，防止请求过快被网盘拉黑封号
        scan_dir "$CURRENT_PATH/$folder"
    done
}

# 开始挨个巡逻你设置的最外层大目录
for ROOT_PATH in "${PATHS[@]}"; do
    echo "==================================="
    echo "🌟 开始执行总任务: $ROOT_PATH"
    scan_dir "$ROOT_PATH"
done

echo "==================================="
echo "🎉 所有网盘的每一个角落都已彻底刷新完毕！"
EOF
```
3.安装“解析眼” (jq)
```
pkg install jq -y
```
4.赐予执行权限
```
chmod +x ~/refresh.sh
```
5.定时服务在后台活着
```
crond
```
6.手动测试
```
~/refresh.sh
```
7.把脚本加入定时任务（这一步会打开一个文本编辑器）
```
crontab -e
```
随后进入编辑模式粘贴以下代码：
```
0 * * * * /data/data/com.termux/files/home/refresh.sh
```
按Ctrl+O 和 Ctrl+X保存退出。
或者直接盲写强插大法
```
echo "0 * * * * /data/data/com.termux/files/home/refresh.sh" > mycron.txt
crontab mycron.txt
rm mycron.txt
```
(这三行代码的意思是：在外面建个文本把任务写好 -> 直接强行塞给 crontab 系统 -> 销毁临时文本。干净利落，全程不需要打开任何编辑器！)
8.检查写成功没
```
crontab -l
```
如果屏幕上成功显示出 0 * * * * /data/data/com.termux/files/home/refresh.sh 这一行字，那就说明定时任务已经完美生效了！
9.定时分目录分时间扫描脚本
9.1 改造脚本
```
cat > ~/refresh.sh << "EOF"
#!/bin/bash

# 你的真实管理员 Token
TOKEN="openlist-4ba3bee7-2112-42bd-92e7-6f3a7c67a83a7CIOwhmfYpB9Hi8HX1NYfRJUn1iWvyPnyuoOUPC5Fqn5FGxTwNuKHwRSYCT6OZC2"

# 检查有没有告诉它要去哪（如果没有路径，就罢工报错）
if [ -z "$1" ]; then
    echo "❌ 错误：请告诉我你要扫哪个目录！(例如: ~/refresh.sh \"/177/177-动漫\")"
    exit 1
fi

TARGET_PATH="$1"

# 核心无限深挖引擎（和之前一样）
scan_dir() {
    local CURRENT_PATH="$1"
    echo "🚀 正在刷新并探测: $CURRENT_PATH"
    
    local JSON=$(curl -s -X POST "http://127.0.0.1:5244/api/fs/list" \
    -H "Authorization: $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"path\":\"$CURRENT_PATH\",\"password\":\"\",\"page\":1,\"per_page\":0,\"refresh\":true}")
    
    local dirs=()
    while IFS= read -r line; do
        if [ -n "$line" ]; then
            dirs+=("$line")
        fi
    done < <(echo "$JSON" | jq -r '.data.content[]? | select(.is_dir==true) | .name')
    
    for folder in "${dirs[@]}"; do
        sleep 2 
        scan_dir "$CURRENT_PATH/$folder"
    done
}

echo "==================================="
echo "🌟 接收到调度指令，开始执行: $TARGET_PATH"
scan_dir "$TARGET_PATH"
echo "🎉 专项目录任务已彻底完成！"
# 1. 踢一脚 Emby，命令它立刻开始扫描（API密钥去Emby后台生成一个填进来）
curl -X POST "http://127.0.0.1:8096/Library/Refresh?api_key=你的EmbyAPI密钥" > /dev/null

# 2. 给你的微信发通知，告诉你活儿干完了！
curl -s "http://www.pushplus.plus/send?token=你的Token&title=私人影院更新&content=云盘深度扫描已完成，Emby正在同步最新资源！" > /dev/null
# 3. # 触发 Telegram 推送
curl -s -X POST "https://api.telegram.org/bot把你的BOT_TOKEN粘贴在这里/sendMessage" -d "chat_id=把你的CHAT_ID粘贴在这里&text=🎬 您的私人影院云盘已扫描完毕，Emby正在入库！" > /dev/null
EOF
```
或者
```
cat > /data/data/com.termux/files/home/refresh.sh << "EOF"
#!/bin/bash

# 你的真实管理员 Token
TOKEN="openlist-4ba3bee7-2112-42bd-92e7-6f3a7c67a83a7CIOwhmfYpB9Hi8HX1NYfRJUn1iWvyPnyuoOUPC5Fqn5FGxTwNuKHwRSYCT6OZC2"

if [ -z "$1" ]; then
    echo "❌ 错误：请告诉我你要扫哪个目录！"
    exit 1
fi

TARGET_PATH="$1"

scan_dir() {
    local CURRENT_PATH="$1"
    echo "🚀 正在刷新并探测: $CURRENT_PATH"
    
    local JSON=$(curl -s -X POST "http://127.0.0.1:5244/api/fs/list" \
    -H "Authorization: $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"path\":\"$CURRENT_PATH\",\"password\":\"\",\"page\":1,\"per_page\":0,\"refresh\":true}")
    
    local dirs=()
    while IFS= read -r line; do
        if [ -n "$line" ]; then
            dirs+=("$line")
        fi
    done < <(echo "$JSON" | jq -r '.data.content[]? | select(.is_dir==true) | .name')
    
    for folder in "${dirs[@]}"; do
        sleep 2 
        scan_dir "$CURRENT_PATH/$folder"
    done
}

echo "==================================="
echo "🌟 接收到调度指令，开始执行: $TARGET_PATH"
scan_dir "$TARGET_PATH"
echo "🎉 OpenList 专项目录任务已下发完毕！"

# ==========================================
# 🌟 修复核心：等本地硬盘同步 strm 文件
# ==========================================
echo "⏳ 给硬盘 30 秒钟时间生成 strm 文件..."
sleep 30

echo "🎬 呼叫 Emby 进行扫描..."
# 捕获 Emby 的真实反应 (HTTP 状态码)
STATUS=$(curl -s -o /dev/null -w "%{http_code}" -X POST "http://127.0.0.1:8096/Library/Refresh?api_key=fcf90c63fd1343cf9e6c66a62f73cfe5")

if [ "$STATUS" == "204" ]; then
    echo "✅ Emby 已成功接收扫描指令！请等待刮削完成推送..."
else
    echo "❌ Emby 呼叫失败！HTTP 状态码: $STATUS (检查 IP、端口或 API Key)"
fi
echo "==================================="
EOF

chmod +x /data/data/com.termux/files/home/refresh.sh
```
9.2终极排班表
```
cat > mycron.txt << "EOF"
# 电影：每天 1 次 (22:00)
0 22 * * * /data/data/com.termux/files/home/refresh.sh "/177/177-电影"

# 动漫：每天 2 次 (11:30、19:30)
30 11,19 * * * /data/data/com.termux/files/home/refresh.sh "/177/177-动漫"

# 电视剧：每天 1 次 (21:00)
0 21 * * * /data/data/com.termux/files/home/refresh.sh "/177/177-电视剧"
EOF
crontab mycron.txt
rm mycron.txt
crontab -l
```
9.3立刻单独扫一下动漫
```
~/refresh.sh "/177/177-动漫"
```
10.电报/微信媒体通知
10.1.Termux 装上“翻译工具”
回到你的 Termux 命令行（~ $ 界面），依次执行下面两行命令：
```
pkg install python -y
```
```
pip install flask requests
```
(这会给系统装上 Python 和专门用来建网站接收数据的 Flask 工具包，大概需要一两分钟。)
10.2.写入咱们的“翻译官”核心代码

监控播放与入库通知
```
cat > ~/tg_bridge.py << "EOF"
from flask import Flask, request
import requests
import time
import threading
import queue

app = Flask("emby_bridge")

# ==========================================
# --- 你的配置信息 ---
# ==========================================
BOT_TOKEN = "7548615667:AAHn0ls4aBPKBPI2-gpwykwVdEKd0ywOlsc"
CHAT_ID = "-1002906711199"
EMBY_API_KEY = "fcf90c63fd1343cf9e6c66a62f73cfe5"
PUSHPLUS_TOKEN = "564ba5836b054b1698180dbab49892c9"

# 🌟 老板免打扰特权账号
BOSS_NAME = "xxsky" 
# ==========================================

msg_queue = queue.Queue()
recent_events = {}  # 🌟 新增：去重记忆库

def worker():
    while True:
        try:
            task = msg_queue.get()
            if not task: continue

            title, msg, target_img_id = task['title'], task['msg'], task['img_id']
            full_msg = f"{title}\n\n{msg}"
            
            photo_stream = None
            if target_img_id and EMBY_API_KEY:
                try:
                    img_resp = requests.get(f"http://127.0.0.1:8096/Items/{target_img_id}/Images/Primary?api_key={EMBY_API_KEY}", timeout=5)
                    if img_resp.status_code == 200: photo_stream = img_resp.content
                except: pass

            try:
                if photo_stream:
                    requests.post(f"https://api.telegram.org/bot{BOT_TOKEN}/sendPhoto", data={'chat_id': CHAT_ID, 'caption': full_msg}, files={'photo': ('poster.jpg', photo_stream, 'image/jpeg')}, timeout=15)
                else:
                    requests.get(f"https://api.telegram.org/bot{BOT_TOKEN}/sendMessage", params={"chat_id": CHAT_ID, "text": full_msg}, timeout=10)
            except: pass

            if PUSHPLUS_TOKEN:
                try: requests.get("http://www.pushplus.plus/send", params={"token": PUSHPLUS_TOKEN, "title": title, "content": msg}, timeout=5)
                except: pass
            
            time.sleep(3)
            msg_queue.task_done()
        except Exception:
            time.sleep(3)

threading.Thread(target=worker, daemon=True).start()

@app.route('/emby', methods=['POST'])
def receive_emby():
    data = request.json
    if not data: return "OK", 200

    if data.get('Title') == 'Test Notification':
        msg_queue.put({'title': "系统测试", 'msg': "✅ 报告老板：入库与播放双通道测试成功！", 'img_id': None})
        return "OK", 200
        
    if 'Item' not in data: return "OK", 200
        
    item = data['Item']
    item_type = item.get('Type', '')
    event = data.get('Event', '')
    user_name = data.get('User', {}).get('Name', '某人')
    item_id = item.get('Id', 'unknown')

    # 1. 过滤无关事件，只保留“新入库”和“开始播放”
    if event not in ['library.new', 'playback.start']:
        return "OK", 200

    # 🌟 2. 核心拦截：60秒内同样的事件直接静默丢弃！(防 Emby 连发)
    current_time = time.time()
    event_signature = f"{event}_{item_id}_{user_name}" # 记忆特征码

    # 自动清理过期的记忆（保持内存干净）
    for key in list(recent_events.keys()):
        if current_time - recent_events[key] > 60:
            del recent_events[key]

    # 如果这个动作 60 秒内刚处理过，直接假装没看见！
    if event_signature in recent_events:
        return "Duplicate Ignored", 200
        
    # 记下这次动作的时间戳
    recent_events[event_signature] = current_time

    # 3. 老板免打扰拦截
    if event == 'playback.start' and user_name == BOSS_NAME:
        return "OK", 200

    # ==========================================
    # 场景一：【新片入库】
    # ==========================================
    if event == 'library.new':
        if item_type == 'Movie':
            title, msg = "🎬 电影入库啦！", f"《{item.get('Name')}》 ({item.get('ProductionYear', '')})\n✨ 已准备就绪！"
        elif item_type == 'Series':
            title, msg = "📺 新剧开播啦！", f"《{item.get('Name')}》 ({item.get('ProductionYear', '')})\n✨ 已全量建档！"
        elif item_type == 'Season': return "OK", 200 
        elif item_type == 'Episode':
            season = item.get('SeasonName', '')
            ep = item.get('Name', '')
            ep_text = f"{season} - {ep}" if season else ep
            title, msg = "📺 追剧更新啦！", f"《{item.get('SeriesName')}》 {ep_text}\n✨ 已自动入库！"
        else:
            title, msg = "✨ 影院上新", f"【{item.get('Name')}】 已准备就绪！"

    # ==========================================
    # 场景二：【别人在播放】
    # ==========================================
    elif event == 'playback.start':
        title = "▶️ 播放监控"
        if item_type == 'Episode':
            season = item.get('SeasonName', '')
            ep = item.get('Name', '')
            ep_text = f"{season} - {ep}" if season else ep
            msg = f"👤 【{user_name}】 正在观看：\n《{item.get('SeriesName')}》 {ep_text}"
        else:
            msg = f"👤 【{user_name}】 正在观看：\n《{item.get('Name')}》"

    target_img_id = item.get('SeriesId') if (item_type == 'Episode' and item.get('SeriesId')) else item_id
    msg_queue.put({'title': title, 'msg': msg, 'img_id': target_img_id})
            
    return "OK", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF
```

10.3.静默启动
```
nohup python ~/tg_bridge.py > /dev/null 2>&1 &
```
或启动
```
python ~/tg_bridge.py
```
杀掉进程
```
pkill -f tg_bridge.py
```
或
```
pkill -9 -f tg_bridge.py
```
```
pkill -9 python
```
10.4Emby通知设置
设置-admin首选项-通知-添加通知-webhooks
名称：电报/微信入库通知
网址：http://127.0.0.1:8000/emby
请求内容类型：application/json
勾选媒体库

## 自动转存

1.初始化 Termux 环境
打开你的 Termux，把这三行命令依次复制进去回车。这步是为了装好 Python 和需要的底层运行库（如果提示 [Y/n]，直接敲 y 回车）：
```
pkg update -y && pkg upgrade -y
pkg install python clang make libffi -y
pip install requests pycryptodome python-dotenv schedule
```
2.Termux安装必要的 Python 引擎库
```
pip install flask requests
```
3.库出错
这个 ModuleNotFoundError: No module named 'Crypto' 是 Python 界一个极其经典且烦人的“坑”，几乎所有第一次折腾加密库的人都会踩中
原因很简单：代码里调用的名字叫 Crypto，但它对应的现代库其实叫 pycryptodome。有时候 Python 环境会犯傻，或者之前残留了一些废弃的老库（比如老古董 pycrypto），导致它“认错人”了
直接复制下面这两行命令，依次在 Termux 里回车（第一行可能会提示未找到某些库，不用管，直接让它执行完）：
```
pip uninstall crypto pycrypto pycryptodome -y
pip install pycryptodome
```
现在冒出来的这个 No module named 'schedule' 报错，是因为咱们最开始那步批量装库的时候，可能因为网络波动中断了，导致 schedule（用来做定时任务的库）没装上。

咱们现在就玩“打地鼠”，它缺啥咱们补啥！为了防止等会儿它再报别的库没装，咱们干脆把脚本需要的剩下几个第三方库一次性全补齐。

直接复制这行回车：
```
pip install schedule python-dotenv requests
```

4.建立专属工作台与配置文件

```
# 1. 创建专属文件夹并进去

mkdir -p ~/cloud189_bot/db
cd ~/cloud189_bot

# 2. 创建环境变量文件并编辑
cd
nano sys.env
```
执行完 nano sys.env 后，屏幕会变成黑底白字的编辑器。把下面这段内容修改成你自己的真实信息后，粘贴进去（注意等号两边不要有空格）：
```
# 你的天翼云盘账号和密码
ENV_189_CLIENT_ID=13800138000
ENV_189_CLIENT_SECRET=你的天翼密码

# 你的 TG 机器人配置
ENV_TG_BOT_TOKEN=123456789:ABCDEFGHIJKLMNOPQRSTUVWXYZ
ENV_TG_ADMIN_USER_ID=你的TG数字ID
```
填完后，按 Ctrl + O（字母O），回车保存；然后按 Ctrl + X 退出。

5.写入终极神级脚本

5.1在电脑中新建文件auto_189.py内容如下：
```
import os
import json
import time
import requests
import re
import subprocess
import random
from urllib import parse
from Crypto.Cipher import PKCS1_v1_5 as Cipher_pksc1_v1_5
from Crypto.PublicKey import RSA
import logging
import schedule
from dotenv import load_dotenv
from datetime import datetime

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)
load_dotenv(dotenv_path="sys.env", override=True)

ENV_189_CLIENT_ID = os.getenv("ENV_189_CLIENT_ID", "")
ENV_189_CLIENT_SECRET = os.getenv("ENV_189_CLIENT_SECRET", "")
TG_BOT_TOKEN = os.getenv("ENV_TG_BOT_TOKEN", "")
TG_ADMIN_USER_ID = os.getenv("ENV_TG_ADMIN_USER_ID", "")

SUBS_FILE = "db/subscriptions.json"
HISTORY_FILE = "db/history.json"

def load_json(filepath):
    if os.path.exists(filepath):
        with open(filepath, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {}

def save_json(filepath, data):
    with open(filepath, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

def clean_filename(name):
    illegal_chars = '"\\/:*?|<>'
    for char in illegal_chars:
        name = name.replace(char, '')
    return name[:255]

def rsaEncrpt(password, public_key):
    rsakey = RSA.importKey(public_key)
    cipher = Cipher_pksc1_v1_5.new(rsakey)
    return cipher.encrypt(password.encode()).hex()

class TelegramNotifier:
    def __init__(self, bot_token, user_id):
        self.bot_token = bot_token
        self.user_id = user_id
        self.base_url = f"https://api.telegram.org/bot{self.bot_token}/" if self.bot_token else None

    def send_message(self, message):
        if not self.bot_token: return False
        try:
            requests.get(f"{self.base_url}sendMessage", params={"chat_id": self.user_id, "text": message}, timeout=10)
            return True
        except:
            return False

class Cloud189ShareInfo:
    def __init__(self, fileId, shareId, shareMode, cloud189Client, accessCode="", is_folder=True, file_name=""):
        self.shareDirFileId = fileId
        self.shareId = shareId
        self.session = cloud189Client.session
        self.client = cloud189Client
        self.shareMode = shareMode
        self.accessCode = accessCode
        self.is_folder = is_folder
        self.file_name = file_name

    def getAllShareFiles(self, folder_id=None):
        if not self.is_folder and folder_id is None:
            return {"files": [{"id": self.shareDirFileId, "name": self.file_name}], "folders": []}

        if folder_id is None:
            folder_id = self.shareDirFileId
        fileList, folders = [], []
        pageNumber = 1
        while True:
            result = self.session.get("https://cloud.189.cn/api/open/share/listShareDir.action", params={
                "pageNum": pageNumber, "pageSize": "10000", "fileId": folder_id,
                "shareDirFileId": self.shareDirFileId, "isFolder": "true",
                "shareId": self.shareId, "shareMode": self.shareMode,
                "orderBy": "lastOpTime", "descending": "true", "accessCode": self.accessCode,
            }).json()
            if result['res_code'] != 0: break
            fileListAO = result.get("fileListAO", {})
            fileList += fileListAO.get("fileList", [])
            folders += fileListAO.get("folderList", [])
            if fileListAO.get("fileListSize", 0) == 0 and len(fileListAO.get("folderList", [])) == 0: break
            pageNumber += 1
        return {"files": fileList, "folders": folders}

    def saveShareFiles(self, tasksInfos, targetFolderId):
        try:
            response = self.session.post("https://cloud.189.cn/api/open/batch/createBatchTask.action", data={
                "type": "SHARE_SAVE", "taskInfos": str(tasksInfos),
                "targetFolderId": targetFolderId, "shareId": self.shareId,
            }).json()
            if response.get("res_code") != 0: return response.get('res_message', 'UNKNOWN_ERROR')
            
            taskId = response["taskId"]
            while True:
                res = self.session.post("https://cloud.189.cn/api/open/batch/checkBatchTask.action", data={
                    "taskId": taskId, "type": "SHARE_SAVE"
                }).json()
                if res["taskStatus"] != 3 or res.get("errorCode"): break
                time.sleep(1)
            return res.get("errorCode")
        except Exception as e:
            return str(e)

class Cloud189:
    def __init__(self):
        self.session = requests.session()
        self.session.headers = {
            'User-Agent': "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Accept": "application/json;charset=UTF-8",
        }

    def getEncrypt(self):
        return self.session.post("https://open.e.189.cn/api/logbox/config/encryptConf.do", data={'appId': 'cloud'}, timeout=15).json()['data']['pubKey']

    def getRedirectURL(self):
        rsp = self.session.get('https://cloud.189.cn/api/portal/loginUrl.action?redirectURL=https://cloud.189.cn/web/redirect.html?returnURL=/main.action', timeout=15)
        return parse.parse_qs(parse.urlparse(rsp.url).query)

    def login(self, username, password):
        encryptKey = self.getEncrypt()
        query = self.getRedirectURL()
        resData = self.session.post('https://open.e.189.cn/api/logbox/oauth2/appConf.do', data={"version": '2.0', "appKey": 'cloud'}, headers={"Referer": 'https://open.e.189.cn/', "lt": query["lt"][0], "REQID": query["reqId"][0]}, timeout=15).json()
        
        keyData = f"-----BEGIN PUBLIC KEY-----\n{encryptKey}\n-----END PUBLIC KEY-----"
        data = {
            "appKey": 'cloud', "version": '2.0', "accountType": '01', "mailSuffix": '@189.cn',
            "returnUrl": resData['data']['returnUrl'], "paramId": resData['data']['paramId'],
            "clientType": '1', "isOauth2": "false",
            "userName": f"{{NRP}}{rsaEncrpt(username, keyData)}",
            "password": f"{{NRP}}{rsaEncrpt(password, keyData)}",
        }
        result = self.session.post('https://open.e.189.cn/api/logbox/oauth2/loginSubmit.do', data=data, headers={'Referer': 'https://open.e.189.cn/', 'lt': query["lt"][0], 'REQID': query["reqId"][0]}, timeout=15).json()
        if result['result'] == 0:
            self.session.get(result['toUrl'], headers={"Host": 'cloud.189.cn'}, timeout=15)
        else:
            raise Exception(result['msg'])

    def getShareInfo(self, link):
        url = parse.urlparse(link)
        try:
            code = parse.parse_qs(url.query)["code"][0]
        except:
            code = url.path.split('/')[-1]
            
        pwd = parse.parse_qs(url.query).get('pwd', [''])[0]
        
        result = self.session.get("https://cloud.189.cn/api/open/share/getShareInfoByCodeV2.action", params={"shareCode": code}).json()
        if result.get('res_code') != 0: 
            raise Exception(f"获取分享失败，可能掉线: {result}")
            
        file_id = result.get("fileId")
        share_mode = result.get("shareMode", 1)
        share_id = result.get("shareId")
        
        raw_is_folder = result.get("isFolder")
        is_folder = True if raw_is_folder is None else str(raw_is_folder).lower() in ['true', '1']
        file_name = result.get("fileName", "未命名文件")
        
        if pwd:
            verify_res = self.session.get("https://cloud.189.cn/api/open/share/checkAccessCode.action", params={"shareCode": code, "accessCode": pwd}).json()
            if verify_res.get('res_code') != 0:
                raise Exception(f"提取码错误或失效: {verify_res}")
            share_id = verify_res.get("shareId")
            
        if not share_id:
            raise Exception("未能获取到 shareId，疑似掉线拦截。")
            
        return Cloud189ShareInfo(file_id, share_id, share_mode, self, pwd, is_folder, file_name)

    def createFolder(self, name, parentFolderId=-11):
        result = self.session.post("https://cloud.189.cn/api/open/file/createFolder.action", data={"parentFolderId": parentFolderId, "folderName": name}).json()
        return result["id"]

    def getObjectFolderNodes(self, folderId=-11):
        res = self.session.post("https://cloud.189.cn/api/portal/getObjectFolderNodes.action", data={"id": folderId, "orderBy": 1, "order": "ASC"}).json()
        if isinstance(res, dict):
            raise Exception(f"账号掉线或被限制 (获取目录失败): {res}")
        return res

    def mkdirAll(self, path, parentFolderId=-11):
        path = path.strip("/")
        if not path: return parentFolderId
        for name in path.split("/"):
            found = False
            for node in self.getObjectFolderNodes(parentFolderId):
                if node["name"] == name:
                    parentFolderId = node["id"]
                    found = True
                    break
            if not found:
                parentFolderId = self.createFolder(name, parentFolderId)
        return parentFolderId

def get_all_share_files_recursive(info, folder_id=None, current_path=""):
    all_files = []
    result = info.getAllShareFiles(folder_id)
    for f in result.get("files", []):
        f["full_path"] = current_path + "/" + f["name"]
        all_files.append(f)
    for folder in result.get("folders", []):
        new_path = current_path + "/" + folder["name"]
        all_files.extend(get_all_share_files_recursive(info, folder["id"], new_path))
    return all_files

def auto_relogin(client):
    logger.info("🔄 触发自动保活机制：正在重新登录...")
    try:
        client.login(ENV_189_CLIENT_ID, ENV_189_CLIENT_SECRET)
        logger.info("✅ 自动重新登录成功！通行证已刷新。")
        return True
    except Exception as e:
        logger.error(f"❌ 自动重新登录失败: {e}")
        return False

def check_subscriptions(client, force_target_id=None):
    subs = load_json(SUBS_FILE)
    history = load_json(HISTORY_FILE)
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)

    if not subs: return
    
    current_time = time.time()
    logger.info(f"🔍 巡逻开始，总任务数: {len(subs)}")
    
    for target_id, sub_info in subs.items():
        try:
            share_url = sub_info if isinstance(sub_info, str) else sub_info.get("url", "")
            keyword = "" if isinstance(sub_info, str) else sub_info.get("keyword", "")
            path = "" if isinstance(sub_info, str) else sub_info.get("path", "")
            freq = "" if isinstance(sub_info, str) else sub_info.get("freq", "")

            if force_target_id and str(target_id) == str(force_target_id):
                pass
            elif path:
                now = datetime.now()
                curr_h, curr_m, curr_w = now.hour, now.minute, now.weekday()
                
                next_check_time = 0 if isinstance(sub_info, str) else sub_info.get("next_check_time", 0)

                if current_time < next_check_time:
                    continue

                if freq == "剧迷":
                    is_active_window = (10 <= curr_h < 12) or (18 <= curr_h < 24)
                    if not is_active_window: continue
                    
                elif freq == "周更" or "周更" in path or "动漫" in path:
                    update_weekday = 5 if isinstance(sub_info, str) else sub_info.get("update_weekday", 5)
                    if curr_w < update_weekday: continue  
                    
                    is_am = (curr_h == 10 and curr_m >= 30) or (curr_h == 11)
                    is_pm = (curr_h >= 18 and curr_m >= 30) or (curr_h >= 19)
                    if not (is_am or is_pm): continue
                    
                elif freq == "日更" or "日更" in path or "电视剧" in path or "剧" in path:
                    if curr_h < 18: continue
            
            info = client.getShareInfo(share_url)
            all_files = get_all_share_files_recursive(info)
            
            if keyword:
                all_files = [f for f in all_files if keyword.lower() in f["full_path"].lower()]

            existing_names = set(v["name"] for k, v in history.items() if isinstance(v, dict) and v.get("sub_id") == str(target_id))
            new_files = [f for f in all_files if str(f["id"]) not in history and clean_filename(f["name"]) not in existing_names]

            if new_files:
                logger.info(f"🎉 发现 {len(new_files)} 个新文件，开始转存...")
                taskInfos = [{"fileId": f["id"], "fileName": clean_filename(f["name"]), "isFolder": 0} for f in new_files]
                
                batch_size = 50
                for i in range(0, len(taskInfos), batch_size):
                    batch_tasks = taskInfos[i:i+batch_size]
                    code = info.saveShareFiles(batch_tasks, target_id)
                    
                    if not code:
                        file_names = []
                        for task in batch_tasks:
                            history[str(task["fileId"])] = {"name": task["fileName"], "sub_id": str(target_id)}
                            file_names.append(task["fileName"])
                        save_json(HISTORY_FILE, history)
                        notifier.send_message(f"✅【追剧更新】\n🔗 来源: {share_url}\n📂 新增文件:\n" + "\n".join(file_names))
                        
                        try:
                            openlist_target_path = f"/177{path}" if path.startswith("/") else f"/177/{path}"
                            subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path])
                            notifier.send_message(f"🔄 已自动触发 Emby 刮削入库: {openlist_target_path}")
                        except Exception as script_e:
                            logger.error(f"❌ 联动失败: {script_e}")

                    subs_for_update = load_json(SUBS_FILE)
                    if str(target_id) in subs_for_update:
                        if isinstance(subs_for_update[str(target_id)], str):
                            subs_for_update[str(target_id)] = {"url": share_url, "keyword": keyword, "path": path}
                        
                        subs_for_update[str(target_id)]["last_update"] = time.time()
                        
                        if freq == "剧迷":
                            subs_for_update[str(target_id)]["next_check_time"] = time.time() + 1200
                        elif freq == "周更" or "周更" in path or "动漫" in path:
                            target_weekday = subs_for_update[str(target_id)].get("update_weekday", 5)
                            days_ahead = target_weekday - curr_w
                            if days_ahead <= 0: days_ahead += 7
                            today_midnight = datetime(now.year, now.month, now.day).timestamp()
                            subs_for_update[str(target_id)]["next_check_time"] = today_midnight + days_ahead * 86400
                        elif freq == "日更" or "日更" in path or "电视剧" in path or "剧" in path:
                            subs_for_update[str(target_id)]["next_check_time"] = time.time() + 1800
                        else:
                            subs_for_update[str(target_id)]["next_check_time"] = time.time() + 24 * 3600
                            
                        save_json(SUBS_FILE, subs_for_update)
                else:
                    if code: logger.error(f"转存失败，错误码: {code}")
        except Exception as e:
            error_msg = str(e)
            if "掉线" in error_msg or "失败" in error_msg:
                auto_relogin(client)

def main_control_loop(client):
    offset = 0
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)

    def scheduled_task():
        check_subscriptions(client)
        wait_min = random.randint(25, 45)
        logger.info(f"⏰ 本次巡逻结束，随机预约 {wait_min} 分钟后再次发车...")
        schedule.clear('patrol')
        schedule.every(wait_min).minutes.do(scheduled_task).tag('patrol')

    scheduled_task()
    schedule.every(6).hours.do(auto_relogin, client)

    while True:
        schedule.run_pending()
        try:
            url = f"https://api.telegram.org/bot{TG_BOT_TOKEN}/getUpdates?offset={offset}&timeout=10"
            res = requests.get(url, timeout=15).json()
            if res.get('ok'):
                for item in res['result']:
                    offset = item['update_id'] + 1
                    msg = item.get('message', {})
                    text = msg.get('text', '')
                    chat_id = msg.get('chat', {}).get('id')

                    if str(chat_id) == str(TG_ADMIN_USER_ID):
                        text = text.strip()
                        match_bind = re.match(r'^(订阅|绑定)(\d)?\s+', text)
                        match_fill = re.match(r'^补档\s+(.*?)\s+(http[s]?://\S+)', text)
                        match_refresh = re.match(r'^(刷新|入库)\s+(.*)', text)

                        if "取消订阅" in text:
                            target_kw = text.replace("取消订阅", "").strip()
                            target_kw = re.sub(r'^\d+', '', target_kw).strip()
                            if not target_kw:
                                notifier.send_message("❌ 请在关键词后输入剧名。")
                                continue
                            try:
                                subs = load_json(SUBS_FILE)
                                target_id = None
                                matched_path = ""
                                for sid, info in subs.items():
                                    path_in_db = info.get("path", "") if isinstance(info, dict) else ""
                                    if target_kw.lower() in path_in_db.lower() or path_in_db.lower() in target_kw.lower():
                                        target_id = sid
                                        matched_path = path_in_db
                                        break
                                if target_id:
                                    del subs[str(target_id)]
                                    save_json(SUBS_FILE, subs)
                                    history = load_json(HISTORY_FILE)
                                    old_len = len(history)
                                    new_history = {k: v for k, v in history.items() if not (isinstance(v, dict) and str(v.get("sub_id")) == str(target_id))}
                                    save_json(HISTORY_FILE, new_history)
                                    del_count = old_len - len(new_history)
                                    notifier.send_message(f"✅ 取消成功！\n📂 已移除目录: {matched_path}\n🧹 已清理底层记忆: {del_count} 条。")
                                else:
                                    notifier.send_message(f"❓ 未找到匹配路径: {target_kw}")
                            except Exception as e:
                                logger.error(f"取消失败: {e}")

                        elif match_bind:
                            action = match_bind.group(1)
                            season_num = match_bind.group(2)
                            
                            freq_tag = ""
                            if "#周更" in text: freq_tag, text = "周更", text.replace("#周更", "").strip()
                            elif "#双更" in text: freq_tag, text = "双更", text.replace("#双更", "").strip()
                            elif "#剧迷" in text: freq_tag, text = "剧迷", text.replace("#剧迷", "").strip()
                            elif "#日更" in text: freq_tag, text = "日更", text.replace("#日更", "").strip()
                            
                            weekday_map = {"周一": 0, "周二": 1, "周三": 2, "周四": 3, "周五": 4, "周六": 5, "周日": 6}
                            t_weekday = 5
                            for d_name, d_code in weekday_map.items():
                                if f"#{d_name}" in text:
                                    t_weekday = d_code
                                    text = text.replace(f"#{d_name}", "").strip()
                                    break
                                
                            is_bind = (action == "绑定")
                            parts = text.split()
                            
                            share_url = ""
                            keyword = ""
                            target_path = ""

                            url_index = -1
                            for i, p in enumerate(parts):
                                if p.startswith("http"):
                                    share_url = p
                                    url_index = i
                                    break

                            if url_index != -1:
                                target_path = " ".join(parts[1:url_index])
                                if url_index < len(parts) - 1:
                                    keyword = " ".join(parts[url_index+1:])
                            else:
                                target_path = " ".join(parts[1:])

                            if season_num:
                                s_num = int(season_num)
                                if "season" not in target_path.lower(): target_path = f"{target_path.rstrip('/')}/Season {s_num}"
                                if not keyword: keyword = f"S{s_num:02d}"

                            if not share_url:
                                subs = load_json(SUBS_FILE)
                                for tid, info in subs.items():
                                    if isinstance(info, dict) and info.get("path") == target_path:
                                        share_url = info.get("url", "")
                                        break
                                        
                            if not share_url:
                                notifier.send_message(f"❌ 解析失败：没找到网盘链接，且数据库中未找到【{target_path}】的历史记录。\n请检查拼写或带上链接重试。")
                                continue

                            mode_name = "绑定(静默)" if is_bind else "订阅(下载)"
                            tag_msg = f" ⏱️ 频率: {freq_tag}" if freq_tag else ""
                            kw_msg = f" 🎯 过滤: {keyword}" if keyword else ""
                            day_cn = list(weekday_map.keys())[t_weekday]
                            day_msg = f" 📅 盯梢: {day_cn}" if freq_tag == "周更" else ""
                            
                            notifier.send_message(f"⏳ 正在处理{mode_name}目录：\n📁 {target_path}{tag_msg}{kw_msg}{day_msg} ...")
                                
                            try:
                                target_id = client.mkdirAll(target_path)
                                subs = load_json(SUBS_FILE)
                                subs[str(target_id)] = {
                                    "url": share_url, "keyword": keyword, "path": target_path, 
                                    "last_update": 0, "freq": freq_tag, "update_weekday": t_weekday, 
                                    "next_check_time": 0
                                }
                                save_json(SUBS_FILE, subs)
                                
                                if is_bind:
                                    info = client.getShareInfo(share_url)
                                    all_files = get_all_share_files_recursive(info)
                                    if keyword: all_files = [f for f in all_files if keyword.lower() in f["full_path"].lower()]
                                    history = load_json(HISTORY_FILE)
                                    for f in all_files:
                                        history[str(f["id"])] = {"name": f["name"], "sub_id": str(target_id)}
                                    save_json(HISTORY_FILE, history)
                                    notifier.send_message(f"✅ 成功绑定！\n❇️ 已将 {len(all_files)} 个旧文件标记为已存。")
                                else:
                                    notifier.send_message(f"✅ 成功添加订阅！正在为您优先拉取资源...")
                                    check_subscriptions(client, force_target_id=target_id) 
                            except Exception as e:
                                if "掉线" in str(e) or "失败" in str(e) or "限制" in str(e):
                                    notifier.send_message("⚠️ 网盘疑似掉线，正在自动重连，请稍后重试！")
                                    auto_relogin(client)
                                else:
                                    notifier.send_message(f"❌ 处理失败: {str(e)}")

                        elif match_fill:
                            keyword_input = match_fill.group(1).strip()
                            share_url = match_fill.group(2).strip()

                            # ==========================================
                            # 💥 神级升级：自动劈开“名字”和“季数数字”
                            # ==========================================
                            m = re.match(r'^(.*?)\s*[sS第]?0?(\d+)[季]?$', keyword_input)
                            if m and m.group(1).strip():
                                base_kw = m.group(1).strip()
                                s_num = int(m.group(2))
                            else:
                                base_kw = keyword_input
                                s_num = None

                            msg_ext = f"\n🎯 锁定季数: 第 {s_num} 季" if s_num else ""
                            notifier.send_message(f"🔍 启动智能补档...\n🎯 解析剧名: {base_kw}{msg_ext}\n🔗 链接: {share_url}")

                            subs = load_json(SUBS_FILE)
                            matched_target = None

                            for t_id, info in subs.items():
                                path = info.get("path", "") if isinstance(info, dict) else ""
                                # 只要名字对得上
                                if base_kw.lower() in path.lower():
                                    if s_num is not None:
                                        # 如果你输了数字，就去检查路径里有没有 Season X 或者 SX 等标志
                                        s_patterns = [f"season {s_num}", f"s{s_num:02d}", f"s{s_num}"]
                                        # 兼容直接用数字命名的情况（比如路径最后直接是 /2）
                                        if any(p in path.lower() for p in s_patterns) or str(s_num) in path.split('/')[-1]:
                                            matched_target = (t_id, path)
                                            break
                                    else:
                                        # 没输数字，找到第一个匹配的就上
                                        matched_target = (t_id, path)
                                        break

                            if not matched_target:
                                notifier.send_message(f"❌ 补档中止：您的订阅库中未找到匹配的目录。")
                                continue

                            target_id, target_path = matched_target
                            notifier.send_message(f"🎯 命中目录: {target_path}\n🚀 正在免手动物理转存...")

                            try:
                                info = client.getShareInfo(share_url)
                                all_files = get_all_share_files_recursive(info)
                                history = load_json(HISTORY_FILE)

                                existing_names = set(v["name"] for k, v in history.items() if isinstance(v, dict) and v.get("sub_id") == str(target_id))
                                new_files = [f for f in all_files if clean_filename(f["name"]) not in existing_names]

                                if not new_files:
                                    notifier.send_message("⚠️ 补档完毕：链接里没有新文件，或者与已存文件同名（防重复拦截生效）。")
                                    openlist_target_path = f"/177{target_path}" if target_path.startswith("/") else f"/177/{target_path}"
                                    try:
                                        subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path])
                                    except: pass
                                else:
                                    taskInfos = [{"fileId": f["id"], "fileName": clean_filename(f["name"]), "isFolder": 0} for f in new_files]
                                    success_count = 0
                                    for i in range(0, len(taskInfos), 50):
                                        batch = taskInfos[i:i+50]
                                        code = info.saveShareFiles(batch, target_id)
                                        
                                        if not code:
                                            for task in batch:
                                                history[str(task["fileId"])] = {"name": task["fileName"], "sub_id": str(target_id)}
                                                success_count += 1

                                    save_json(HISTORY_FILE, history)
                                    notifier.send_message(f"✅ 补档完美结束！\n📂 自动归档至: {target_path}\n📝 共计抓取 {success_count} 个新文件，底层记忆已更新。")

                                    try:
                                        openlist_target_path = f"/177{target_path}" if target_path.startswith("/") else f"/177/{target_path}"
                                        subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path])
                                        notifier.send_message(f"🔄 已自动触发 Emby 刮削入库: {openlist_target_path}")
                                    except Exception as e:
                                        pass
                            except Exception as e:
                                notifier.send_message(f"❌ 补档发生异常: {e}")

                        elif match_refresh:
                            keyword_input = match_refresh.group(2).strip()
                            
                            # ==========================================
                            # 💥 入库指令同样享受“名字+季数”拆分待遇
                            # ==========================================
                            m = re.match(r'^(.*?)\s*[sS第]?0?(\d+)[季]?$', keyword_input)
                            if m and m.group(1).strip():
                                base_kw = m.group(1).strip()
                                s_num = int(m.group(2))
                            else:
                                base_kw = keyword_input
                                s_num = None

                            msg_ext = f" (第 {s_num} 季)" if s_num else ""
                            notifier.send_message(f"🔍 收到入库指令，正在检索: {base_kw}{msg_ext}...")
                            
                            subs = load_json(SUBS_FILE)
                            matched_paths = []
                            
                            for t_id, info in subs.items():
                                path_in_db = info.get("path", "") if isinstance(info, dict) else ""
                                if base_kw.lower() in path_in_db.lower():
                                    if s_num is not None:
                                        s_patterns = [f"season {s_num}", f"s{s_num:02d}", f"s{s_num}"]
                                        if any(p in path_in_db.lower() for p in s_patterns) or str(s_num) in path_in_db.split('/')[-1]:
                                            if path_in_db not in matched_paths:
                                                matched_paths.append(path_in_db)
                                    else:
                                        if path_in_db not in matched_paths:
                                            matched_paths.append(path_in_db)
                                        
                            if matched_paths:
                                notifier.send_message(f"🎯 共命中 {len(matched_paths)} 个关联目录，准备批量刷新...")
                                for mp in matched_paths:
                                    openlist_p = f"/177{mp}" if mp.startswith("/") else f"/177/{mp}"
                                    try:
                                        subprocess.run(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_p], timeout=120)
                                        notifier.send_message(f"✅ 目录刷新成功: {openlist_p}")
                                    except: 
                                        pass
                                notifier.send_message("🎉 批量指令已全部呼叫 Emby，请前往查看！")
                            else:
                                notifier.send_message(f"❌ 失败：订阅库中未找到匹配记录。")
                        
                        else:
                            notifier.send_message("❌ 格式错误...")
        except Exception as e:
            pass 
        time.sleep(2)

if __name__ == '__main__':
    os.makedirs("db", exist_ok=True)
    
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    notifier.send_message("🤖 私人追剧管家已启动，正在尝试登录...")
    
    client = Cloud189()
    try:
        logger.info("189正在登录 ...")
        client.login(ENV_189_CLIENT_ID, ENV_189_CLIENT_SECRET)
        notifier.send_message("✅ 网盘登录成功！全天候仿生监控已就位。")
    except Exception as e:
        logger.error(f"登录失败: {e}")
        notifier.send_message(f"❌ 首次登录失败: {e}\n(脚本将直接退出，请处理后重启)")
        exit(-1)
        
    main_control_loop(client)
```
5.2电脑文件传到手机storage/downloads目录下
5.3复制脚本
```
cp ~/storage/downloads/auto_189.py ~/auto_189.py
```
6.点火升空 & 日常操作
准备工作全部完成！现在我们要让它在 Termux 后台静默长驻。
敲下这行命令启动：
```
nohup python auto_189.py > run.log 2>&1 &
```
就这么简单！现在你可以把 Termux 切到后台了。
7.其它有可以出现的问题
7.1杀掉后台的“哑巴”进程
```
pkill -f auto_189.py
```
7.2在前台“裸奔”启动
```
python auto_189.py
```
7.3.清理脏数据，正式接客
在 Termux 里直接执行这两行命令（先清空数据库，再重新启动）：
```
# 1. 强制清空订阅数据库和历史记录（把脏数据扬了）
pkill -f auto_189.py
rm -f db/history.json db/subscriptions.json

# 2. 重新启动脚本！
python auto_189.py
```
7.4.专属启动
```
(cd ~ && nohup python3 auto_189.py > run.log 2>&1 &) && echo "✅ TG 追剧管家已启动，日志记录于 ~/run.log"
```
8.订阅格式
8.1核心万能公式：
[动作] [你的网盘保存路径] [大佬的分享链接(?pwd=密码)] [狙击过滤词] [#休眠标签]
8.2无密码订阅：
订阅 /177-电视剧/狂飙 [https://cloud.189.cn/t/xxxxxx]
8.3带密码订阅（链接屁股后加 ?pwd=密码）：
订阅 /177-电视剧/狂飙 [https://cloud.189.cn/t/xxxxxx?pwd=abcd]
8.4静默绑定
指令（把“订阅”换成“绑定”）：
绑定 /177-电视剧/狂飙 [https://cloud.189.cn/t/xxxxxx?pwd=abcd]
(机器人会提示：已将 N 个旧文件标记为已存。)
8.5精准狙击
指令（在最后空一格，写上过滤词）：
绑定 /177-动漫/神墓/Season 03 [https://cloud.189.cn/t/xxxxxx] S03
(如果资源命名是中文，就把 S03 换成 第三季)
8.6智能休眠（告别无效巡逻，防封神技）
适用场景：动漫更新极其规律，你想让它抓完一集后立刻吃安眠药，节省 API 资源。
* 标签说明： #周更 (抓完睡6天) / #双更 (抓完睡3天)。没打标签的动漫默认睡 2 天，电视剧默认睡 12 小时。
* 指令（随便加在结尾即可）：
8.7究极缝合怪（全功能火力全开）
适用场景：带密码 + 只要第三季 + 周更动漫 + 不下旧文件。

终极指令：
绑定 /177-动漫/神墓/Season 3 [https://cloud.189.cn/t/xxxxxx?pwd=abcd](https://cloud.189.cn/t/xxxxxx?pwd=abcd) S03 #周更
绑定 /177-动漫/2602/天庭 [https://cloud.189.cn/t/YRnIjmAZZZre?pwd=ea12](https://cloud.189.cn/t/YRnIjmAZZZre?pwd=ea12) #周更

注意：订阅1-9或绑定1-9对应Season 1-9文件夹并锁定过滤S01-09

8.8拔管取消
适用场景：剧追完了，或者烂尾不想看了。

无脑取消法：
直接把你当时发送的那条超长的指令复制下来，把开头的 绑定 或 订阅 改成 取消订阅 发过去。
取消订阅 /177-动漫/2602/天庭 https://cloud... (后面的它会自动忽略)

精准取消法：
取消订阅 /177-动漫/2602/天庭
7.9老鸟进阶 Tips：
如何修改任务？
如果你发现忘了加 #周更 标签，或者密码换了，不需要先取消！直接用正确的格式再发一遍 绑定 指令（路径保持不变），它就会极其聪明地自动覆盖旧配置。

它的作息规律是什么？

* 动漫： 每天 10点-12点 & 18点-23点 活跃。

* 电视剧： 每天 18点-23点 活跃。

* 剧迷：每天 10点-12点 & 18点-24点 20 分钟后立刻再战

(非活跃期和 CD 休眠期，它会彻底装死，绝对安全！)

新增指令复习：

添加剧迷： 订阅 /177-电视剧/2026/狂飙 https://... #剧迷

智能补档： 补档 狂飙 https://... (只需关键词和备用链接)

手动入库： 入库 /电影/2026/流浪地球2 (只需路径，自动呼叫脚本刷新)

9.所有脚本启动

```
cat > ~/start.sh << 'EOF'
#!/bin/bash
echo "🚀 正在唤醒所有追更矩阵服务..."

cd ~/openlist && nohup ./openlist server >/dev/null 2>&1 &
echo "✅ OpenList 已启动"

nohup cloudflared tunnel --protocol http2 --config ~/.cloudflared/config.yml run emby_tunnel > ~/tunnel.log 2>&1 &
echo "✅ Cloudflare 隧道已启动"

aria2c --conf-path=$HOME/.config/aria2/aria2.conf -D
echo "✅ Aria2 已启动"

crond
echo "✅ 定时任务 (crond) 已启动"

nohup python ~/tg_bridge.py > /dev/null 2>&1 &
echo "✅ TG 监控通知已启动"

nohup python auto_189.py > run.log 2>&1 &
echo "✅ TG 追剧转存已启动"

echo "🎉 矩阵全面复活！可以退出了。"
EOF
```
```
chmod +x ~/start.sh
```
```
~/start.sh
```

替换strm文件域名的方法
```
find /storage/emulated/0/Download/emby_strm -type f -name "*.strm" -exec sed -i 's|http://op.363689.xyz:8080|https://op.363689.xyz|g' {} +
```
