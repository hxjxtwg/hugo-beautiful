---

title: "Termux之refresh.sh"

author: "xxsky"

type: "posts"

date: 2026-04-24T17:11:16+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

# 摘要

<!--more-->

一、refresh.sh
1.安装定时任务服务
```
pkg install cronie -y
```
2.安装“解析眼” (jq)
```
pkg install jq -y
```
3.赐予执行权限
```
chmod +x ~/refresh.sh
```
4.定时服务在后台活着
```
crond
```
5.手动测试
```
~/refresh.sh
```
6.把脚本加入定时任务（这一步会打开一个文本编辑器）
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
7.检查写成功没
```
crontab -l
```
如果屏幕上成功显示出 0 * * * * /data/data/com.termux/files/home/refresh.sh 这一行字，那就说明定时任务已经完美生效了！
8.定时分目录分时间扫描脚本
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
或者双emby版本
```
cat > /data/data/com.termux/files/home/refresh.sh << "EOF"
#!/bin/bash
echo "$(date '+%Y-%m-%d %H:%M:%S') - 收到刷新指令，路径: $1" >> /data/data/com.termux/files/home/refresh_log.txt

# 你的真实管理员 Token
TOKEN="openlist-a87614da-32dd-4b80-9150-6447de823da8f33x53ymkrx0aPKG0HUcsFHmjFRYTKFhSADLRhoQLkXa7ogaiByhWRNEXCjpblp9"

# ==========================================
# 🔑 双身份密钥配置区
# ==========================================
API_KEY_LINUX="751c095055f8493d8e63eb755369b9aa" # [首选] Linux 版密钥
API_KEY_APP="fcf90c63fd1343cf9e6c66a62f73cfe5"  # [备选] 原 APP 版密钥
# ==========================================

if [ -z "$1" ]; then
    echo "❌ 错误：请告诉我你要扫哪个目录！"
    exit 1
fi

TARGET_PATH="$1"

# 🌟 新增：判断路径类型
if [[ "$TARGET_PATH" == /storage/* ]]; then
    IS_LOCAL_PATH=true
    echo "📁 检测到本地物理路径 (STRM/CAS)，跳过 OpenList 云端扫描..."
else
    IS_LOCAL_PATH=false
    echo "☁️ 检测到云端直连路径，启动 OpenList 缓存穿透..."
fi

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

# 🌟 智能路由：只有云端路径才走 scan_dir
if [ "$IS_LOCAL_PATH" = false ]; then
    echo "==================================="
    echo "🌟 接收到调度指令，开始执行云端穿透: $TARGET_PATH"
    scan_dir "$TARGET_PATH"
    echo "🎉 OpenList 专项目录任务已下发完毕！"
    
    echo "⏳ 给本地网盘客户端 30 秒钟同步挂载时间..."
    sleep 30
else
    # 物理路径 STRM 生成极快，只需要随便缓 3 秒钟防 Emby 数据库锁即可
    echo "⏳ 本地 STRM 模式，缓冲 3 秒..."
    sleep 3
fi

# ==========================================
# 🌟 下发 Emby 精准刷新
# ==========================================
echo "🎬 开始下发【精准局部刷新】指令，优先呼叫 Linux 主力..."

# 构建只扫描当前变动目录的 JSON 载荷
JSON_DATA="{\"Updates\":[{\"Path\":\"$TARGET_PATH\"}]}"

# 1️⃣ 【首选测试】使用 Linux 密钥触发局部刷新
STATUS_LINUX=$(curl -s -o /dev/null -w "%{http_code}" -H "Content-Type: application/json" -X POST -d "$JSON_DATA" "http://127.0.0.1:8096/Library/Media/Updated?api_key=$API_KEY_LINUX")

if [ "$STATUS_LINUX" == "204" ]; then
    echo "✅ [主路由成功] Linux Emby 局部刷新已触发！"
else
    echo "⚠️ Linux 版未响应(状态码:$STATUS_LINUX)，降级呼叫 APP 备用版..."
    
    # 2️⃣ 【备选兜底】使用 APP 老密钥再次触发
    STATUS_APP=$(curl -s -o /dev/null -w "%{http_code}" -H "Content-Type: application/json" -X POST -d "$JSON_DATA" "http://127.0.0.1:8096/Library/Media/Updated?api_key=$API_KEY_APP")
    
    if [ "$STATUS_APP" == "204" ]; then
        echo "✅ [备用路由成功] 独立 APP 版 Emby 局部刷新已触发！"
    else
        echo "❌ [全线崩溃] 两个版本的 Emby 均未响应！"
    fi
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
cat > ~/tgms.py << "EOF"
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
nohup python ~/tgms.py > /dev/null 2>&1 &
```
或启动
```
python ~/tgms.py
```
杀掉进程
```
pkill -f tgms.py
```
或
```
pkill -9 -f tgms.py
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
