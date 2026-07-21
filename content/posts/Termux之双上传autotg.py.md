---

title: "Termux之双上传autotg.py"

author: "xxsky"

type: "posts"

date: 2026-07-14T12:22:55+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

双下载双上传

<!--more-->

### 一、机器人指令菜单

1.@BotFather

2.在对话框里发送指令：/mybots

3.屏幕上会弹出你创建过的机器人列表，点击咱们正在用的这个机器人的名字

4.接着点击弹出面板上的 Edit Bot

5.然后点击 Edit Commands

6.把这套菜单直接喂给它
```
sub - ➕ 自动订阅 (可直接回复视频抓取频道)
unsub - 🗑️ 取消订阅 (全网通杀或精准单杀)
go - 🚀 手动发车 (开辟航线并锁死文件夹)
mag - 🧲[磁力链接]
reup - 🚀 [剧名关键字] [分类] [可选:年份] [c139|only139]
list - 📡 查看雷达大盘排班与状态
scan - 🔍 强行触发一次全网补漏扫荡
history - 📋 查看曾经发过车的历史航线记录
rm - ❌ 删除历史航线！ 把不再需要的死档从history 里永久抹除
rmh - ❎ 带剧名删除下载历史记录
clh - ❎ 清除没订阅的下载历史记录
clean - ⭕️ 清除下载碎片
cancel - ⭕️ 终止下载 (all/剧名(+e01/s01)/.mp4)
setdir - ❇️ 设置上传目录
ping - 💓 查看系统是否活着、存活时间、雷达正在盯防多少部剧
help - 📖 查看随身说明书与指令语法
```

### 二、脚本

```
import os
import re
import time
import json
import asyncio
from datetime import datetime
from urllib.parse import quote
import httpx

# 🔥 必须放在导入 pyrogram 之前！修复 Python 3.12+ 兼容性问题
try:
    asyncio.get_running_loop()
except RuntimeError:
    asyncio.set_event_loop(asyncio.new_event_loop())

from pyrogram import Client, filters, enums, idle
import logging
import shutil

# =================================================================
# ⚙️ 核心网关、路径与凭证配置区域
# =================================================================
API_ID = 33349348              
API_HASH = "44bde7f01d2b6001589c28cea93716af"        

COMMAND_CENTER_CHAT = "@xxskyemby_bot"

# --- 189 天翼云配置 (端口 5244) ---
OLIST_URL = "http://127.0.0.1:5244"
OLIST_TOKEN = "openlist-a87614da-32dd-4b80-9150-6447de823da8f33x53ymkrx0aPKG0HUcsFHmjFRYTKFhSADLRhoQLkXa7ogaiByhWRNEXCjpblp9" 

STEWARD_BASE_URL = "http://127.0.0.1:5000" 

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
TG_ROUTE_DB = os.path.join(BASE_DIR, "tg_manual_routes.json")
LOCAL_TEMP_DIR = os.path.join(BASE_DIR, "tg_temp")
TG_LISTENER_DB = os.path.join(BASE_DIR, "tg_listener_config.json") 
TG_HISTORY_DB = os.path.join(BASE_DIR, "tg_download_history.json") 
TG_MAGNET_DB = os.path.join(BASE_DIR, "tg_magnet_tasks.json")
os.makedirs(LOCAL_TEMP_DIR, exist_ok=True)

TMDB_API_KEY = "9c88e18e43543c8ff195c631aaa0d2fa" 
START_TIME = time.time()

TG_SETTINGS_DB = os.path.join(BASE_DIR, "tg_settings.json")

# =================================================================
# 🧲 Aria2 磁力引擎配置
# =================================================================
ARIA2_RPC_URL = "http://127.0.0.1:6800/jsonrpc"
ARIA2_RPC_SECRET = "xxsky1127"
ARIA2_DOWNLOAD_DIR = "/data/data/com.termux/files/home/downloads"  

# ----------------------- 你的代理 -----------------------
TG_PROXY = {
    "scheme": "http",
    "hostname": "127.0.0.1",
    "port": 7890
}

def get_mount_root():
    if os.path.exists(TG_SETTINGS_DB):
        try:
            with open(TG_SETTINGS_DB, "r", encoding="utf-8") as f: 
                return json.load(f).get("mount_root", "/family/177_cas")
        except: pass
    return "/family/177_cas"
    
def set_mount_root(path):
    settings = {}
    if os.path.exists(TG_SETTINGS_DB):
        try:
            with open(TG_SETTINGS_DB, "r", encoding="utf-8") as f: 
                settings = json.load(f)
        except: pass
    settings["mount_root"] = path
    with open(TG_SETTINGS_DB, "w", encoding="utf-8") as f: 
        json.dump(settings, f, ensure_ascii=False, indent=2)

# =================================================================
# 🤫 日志与全局状态
# =================================================================
logging.basicConfig(level=logging.INFO, format='[%(asctime)s] %(message)s', datefmt='%H:%M:%S')
logging.getLogger("pyrogram").setLevel(logging.WARNING)
logging.getLogger("httpx").setLevel(logging.WARNING)
logger = logging.getLogger("TGEngine")

app = Client(
    "tg_robust_leecher_v15", 
    api_id=API_ID, 
    api_hash=API_HASH, 
    proxy=TG_PROXY
)

# 🚦 全局大盘与锁
GLOBAL_ROUTE_CACHE = {"folder_name": "", "category": "", "folder_season": 1, "file_season": 1, "year": "", "version": "", "expire_time": 0, "manual_ep": None, "enable_139": False}
GLOBAL_ACTIVE_LOCKS = set() 
GLOBAL_CANCEL_TASKS = set() 
GLOBAL_TRACKED_GIDS = set()  
ORPHAN_PROMPTED_GIDS = set() 

# 🔥 核心更新：完全独立的双轨锁 (互不阻塞)
GLOBAL_TRANSFER_LOCK = asyncio.Semaphore(1)   # TG 下载专属锁
GLOBAL_UPLOAD_LOCK = asyncio.Semaphore(1)     # 189 高速上传锁
MAGNET_UPLOAD_LOCK = asyncio.Semaphore(1)     # 189 磁力上传锁
GLOBAL_139_UPLOAD_LOCK = asyncio.Semaphore(1) # 139 限速上传独立锁 (TG与磁力共用排队)

PENDING_MAGNETS = {} 
GLOBAL_STOP_SWEEP = False

async def notify_steward_log(msg, level="INFO"):
    logger.info(f"[{level}] {msg}")
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            await client.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": level, "msg": msg})
    except Exception: pass

def load_magnet_tasks():
    if os.path.exists(TG_MAGNET_DB):
        try:
            with open(TG_MAGNET_DB, 'r', encoding='utf-8') as f: return json.load(f)
        except Exception: pass
    return {}

def save_magnet_tasks(data):
    try:
        with open(TG_MAGNET_DB, 'w', encoding='utf-8') as f: json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception: pass

def add_magnet_task(gid, magnet_link, config_text):
    tasks = load_magnet_tasks()
    tasks[gid] = {"magnet_link": magnet_link, "config_text": config_text, "added_at": int(time.time())}
    save_magnet_tasks(tasks)

def remove_magnet_task(gid):
    tasks = load_magnet_tasks()
    if gid in tasks:
        del tasks[gid]
        save_magnet_tasks(tasks)

def clean_orphan_temp_files(max_age_hours=24):
    if os.path.exists(LOCAL_TEMP_DIR):
        now = time.time()
        for f in os.listdir(LOCAL_TEMP_DIR):
            file_path = os.path.join(LOCAL_TEMP_DIR, f)
            try:
                if os.path.isfile(file_path) and (now - os.path.getmtime(file_path)) > (max_age_hours * 3600):
                    os.remove(file_path)
            except Exception: pass

def load_listener_config():
    if not os.path.exists(TG_LISTENER_DB):
        dummy = {"trusted_channels": {}}
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(dummy, f, ensure_ascii=False, indent=4)
        return dummy
    try:
        with open(TG_LISTENER_DB, "r", encoding="utf-8") as f: return json.load(f)
    except Exception: return {"trusted_channels": {}}

def load_history():
    if os.path.exists(TG_HISTORY_DB):
        try:
            with open(TG_HISTORY_DB, 'r', encoding='utf-8') as f: 
                return json.load(f) 
        except Exception: pass
    return {}

def check_history(drama, file_season, ep, version=""):
    history = load_history()
    ver_tag = f".{version}" if version else ""
    base_name = drama.rstrip(".").replace(" ", ".")
    key = f"{base_name}.S{file_season:02d}E{ep:02d}{ver_tag}"
    if key in history: return True
    prefix = f"{base_name}.S{file_season:02d}E{ep:02d}"
    current_tags = set(t.lower() for t in version.split('.') if t)
    for hist_key in history:
        if hist_key.startswith(prefix):
            hist_version = hist_key[len(prefix):].strip(".")
            hist_tags = set(t.lower() for t in hist_version.split('.') if t)
            if current_tags == hist_tags: return True
    return False

def record_history(drama, file_season, ep, version=""):
    history = load_history()
    ver_tag = f".{version}" if version else ""
    base_name = drama.rstrip(".").replace(" ", ".")
    key = f"{base_name}.S{file_season:02d}E{ep:02d}{ver_tag}"
    if key not in history:
        history[key] = int(time.time())
        try:
            with open(TG_HISTORY_DB, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
        except Exception: pass

def load_tg_routes():
    if os.path.exists(TG_ROUTE_DB):
        try:
            with open(TG_ROUTE_DB, 'r', encoding='utf-8') as f: return json.load(f)
        except Exception: pass
    return {}

def save_tg_routes(data):
    try:
        with open(TG_ROUTE_DB, 'w', encoding='utf-8') as f: json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception: pass

async def fetch_tmdb_details(cn_title):
    if not TMDB_API_KEY: return datetime.now().strftime("%Y"), 9999
    clean_q = re.sub(r'S\d+$|\s+\d+$', '', cn_title).strip()
    url = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={quote(clean_q)}&page=1"
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            res = await client.get(url)
            results = res.json().get("results")
            if results:
                item = results[0]
                year = (item.get("first_air_date") or item.get("release_date") or "")[:4]
                if not year: year = datetime.now().strftime("%Y")
                tmdb_id = item.get("id")
                media_type = item.get("media_type", "tv")
                total_eps = 9999
                if media_type == "tv" and tmdb_id:
                    det_url = f"https://api.themoviedb.org/3/tv/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN"
                    det_res = await client.get(det_url)
                    total_eps = det_res.json().get("number_of_episodes", 9999)
                return year, total_eps
    except Exception: pass
    return datetime.now().strftime("%Y"), 9999

def extract_pure_episode(search_text, drama_anchor=None):
    text = search_text
    if drama_anchor:
        try: text = re.compile(re.escape(drama_anchor), re.IGNORECASE).sub(' ', text)
        except: pass
    m = re.search(r'(?i)E(?:P)?0*(\d+)', text)
    if m: return int(m.group(1))
    m = re.search(r'第\d+[季部].*?第\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    m = re.search(r'第\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    m = re.search(r'(?<![第\d])\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    m_cn = re.search(r'第\s*([一二三四五六七八九十零百]+)\s*[集话期更]', text)
    if m_cn:
        cn_str = m_cn.group(1)
        cn_map = {"一":1, "二":2, "三":3, "四":4, "五":5, "六":6, "七":7, "八":8, "九":9, "十":10}
        if cn_str in cn_map: return cn_map[cn_str]
        if cn_str.startswith("十") and len(cn_str) == 2: return 10 + cn_map.get(cn_str[1], 0)
        if len(cn_str) == 3 and cn_str[1] == "十": return cn_map.get(cn_str[0],0)*10 + cn_map.get(cn_str[2],0)
        if cn_str.endswith("十") and len(cn_str) == 2: return cn_map.get(cn_str[0],0)*10
    clean_text = re.sub(r'(?i)(h264|h265|x264|x265|720p|1080p|2160p|4k|8k|web-dl|webrip)', '', text)
    m_trail = re.search(r'(?<!\d)0*(\d{1,3})(?!\d)', clean_text)
    if m_trail and not (1900 < int(m_trail.group(1)) < 2100): return int(m_trail.group(1))
    return None

def extract_movie_part(search_text):
    m = re.search(r'(?i)(?:cd|part|pt)[\s_.-]*(\d{1,2})(?!\d)', search_text)
    if m: return f"cd{m.group(1)}" 
    return ""

def extract_media_tags(search_text):
    text = search_text.upper() 
    tags = []
    if re.search(r'(2160P|4K)', text): tags.append("2160p") 
    elif re.search(r'(1080P)', text): tags.append("1080p")
    elif re.search(r'(720P)', text): tags.append("720p")
    if re.search(r'(DV|DOVI|DOLBY VISION)', text): tags.append("DV")
    if re.search(r'(HDR10\+|HDR10|HDR)', text): tags.append("HDR")
    if re.search(r'(SDR)', text): tags.append("SDR")
    if re.search(r'(HQ|HIGH QUALITY)', text): tags.append("HQ")
    if re.search(r'(IQ)', text): tags.append("IQ")
    if re.search(r'(60FPS)', text): tags.append("60fps")
    return ".".join(tags)

def smart_rename(orig_name, drama_name, ep_num=None, season=None, is_movie=False):
    has_cn = re.search(r'[\u4e00-\u9fa5]', orig_name)
    if has_cn:
        return orig_name.replace(" ", ".")
    
    clean_drama = drama_name.replace(" ", ".")
    if is_movie:
        return f"{clean_drama}.{orig_name}".replace("..", ".")
    else:
        has_std_ep = re.search(r'(?i)(S\d+E\d+|EP?\d+)', orig_name)
        if has_std_ep:
            return f"{clean_drama}.{orig_name}".replace("..", ".")
        else:
            ep_tag = f"S{season:02d}E{ep_num:02d}" if season and ep_num else ""
            if ep_tag:
                return f"{clean_drama}.{ep_tag}.{orig_name}".replace("..", ".")
            return f"{clean_drama}.{orig_name}".replace("..", ".")

async def wipe_magnet_task(gids_to_wipe):
    if isinstance(gids_to_wipe, str): gids_to_wipe = [gids_to_wipe]
    target_gids = set(gids_to_wipe)
    paths_to_delete = set()
    info_hashes = set()  
    
    await notify_steward_log(f"🧹 [清理系统启动] 开始清理 Aria2 任务，锁入 GID 链: {list(target_gids)}")
    try:
        async with httpx.AsyncClient(timeout=10.0) as h:
            for gid in list(target_gids):
                try:
                    res = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellStatus", "params":[f"token:{ARIA2_RPC_SECRET}", gid]})
                    st = res.json().get("result", {})
                    if "followedBy" in st and st["followedBy"]: target_gids.update(st["followedBy"])
                    if "belongsTo" in st and st["belongsTo"]: target_gids.add(st["belongsTo"])
                except: pass
            for g in list(target_gids):
                try:
                    res_g = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellStatus", "params":[f"token:{ARIA2_RPC_SECRET}", g]})
                    st_data = res_g.json().get("result", {})
                    if "infoHash" in st_data and st_data["infoHash"]: info_hashes.add(st_data["infoHash"])
                    files = st_data.get("files", [])
                    for f in files:
                        if f.get("path"): paths_to_delete.add(f.get("path"))
                except: pass
    except: pass
    try:
        async with httpx.AsyncClient(timeout=10.0) as h:
            for g in target_gids:
                try: await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.forceRemove", "params":[f"token:{ARIA2_RPC_SECRET}", g]})
                except: pass
    except: pass
    await asyncio.sleep(3.0)
    for info_hash in info_hashes:
        torrent_path = os.path.join(ARIA2_DOWNLOAD_DIR, f"{info_hash}.torrent")
        if os.path.exists(torrent_path) and os.path.isfile(torrent_path):
            try: os.remove(torrent_path)
            except: pass
    top_dirs_to_delete = set()
    for f_path in paths_to_delete:
        if f_path.startswith(ARIA2_DOWNLOAD_DIR):
            rel = os.path.relpath(f_path, ARIA2_DOWNLOAD_DIR)
            parts = rel.split(os.sep)
            if len(parts) > 0:
                top_dir = os.path.join(ARIA2_DOWNLOAD_DIR, parts[0])
                if os.path.isdir(top_dir) and top_dir != ARIA2_DOWNLOAD_DIR:
                    top_dirs_to_delete.add(top_dir)
        if os.path.exists(f_path) and os.path.isfile(f_path):
            try: os.remove(f_path)
            except: pass
        aria2_path = f"{f_path}.aria2"
        if os.path.exists(aria2_path) and os.path.isfile(aria2_path):
            try: os.remove(aria2_path)
            except: pass
    for top_dir in top_dirs_to_delete:
        try:
            if os.path.exists(top_dir): shutil.rmtree(top_dir, ignore_errors=True)
        except: pass
    try:
        async with httpx.AsyncClient(timeout=10.0) as h:
            for g in target_gids:
                for attempt in range(1, 6): 
                    try:
                        res = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.removeDownloadResult", "params":[f"token:{ARIA2_RPC_SECRET}", g]})
                        if "error" not in res.json(): break
                    except: pass
                    await asyncio.sleep(1.5)
    except: pass
    await notify_steward_log("🧹 [清理系统完毕] 磁力任务关联的实体文件与面板记录已无痕销毁。")

async def bg_fetch_cas_task(cas_target_full, final_cas_path, sub_path, cas_file_name, status_msg=None, olist_url=OLIST_URL, olist_token=OLIST_TOKEN, drive_name="189"):
    try:
        await asyncio.sleep(10)
        get_info_url = f"{olist_url}/api/fs/get"
        headers_get = {"Authorization": olist_token, "Content-Type": "application/json"}
        cas_downloaded = False
        
        for attempt in range(30):
            try:
                async with httpx.AsyncClient(timeout=20.0, follow_redirects=True) as client_get:
                    resp = await client_get.post(get_info_url, json={"path": cas_target_full}, headers=headers_get)
                    resp_data = resp.json()
                    
                    if resp_data.get("code") == 200:
                        raw_url = resp_data["data"]["raw_url"]
                        resp_dl = await client_get.get(raw_url)
                        
                        if resp_dl.status_code == 200:
                            temp_cas_path = final_cas_path + ".tmp"
                            with open(temp_cas_path, "wb") as f_cas:
                                f_cas.write(resp_dl.content)
                            os.rename(temp_cas_path, final_cas_path)
                            await notify_steward_log(f"🔗 [{drive_name} CAS下发成功] `{sub_path}/{cas_file_name}`")
                            
                            if status_msg:
                                try: await status_msg.edit_text(f"🔗 **[{drive_name} CAS就绪]**\n📥 `{cas_file_name}`\n✅ 镜像完毕，等待接管！")
                                except: pass
                            else:
                                try: await app.send_message(COMMAND_CENTER_CHAT, f"🔗 **[{drive_name} CAS就绪]**\n📥 `{cas_file_name}`\n✅ 镜像完毕，等待接管！")
                                except: pass
                            # 🔥 新增：成功拿到 CAS 后，通知 5555 脚本去干活
                            # 这里我把 final_cas_path (也就是下载好的本地 .cas 文件完整路径) 传给了 path 参数,如果 5555 需要的不是本地 .cas 路径，而是云端目录，你只需把代码里的 quote(final_cas_path) 改成 quote(cas_target_full) 即可
                            trigger_url = f"http://127.0.0.1:5555/force_harvest?path={quote(final_cas_path)}"
                            asyncio.create_task(trigger_strm_sync(trigger_url, cas_file_name, "189联动"))

                            cas_downloaded = True
                            break
            except Exception: pass 
            if not cas_downloaded: await asyncio.sleep(10)

        if not cas_downloaded:
            if status_msg:
                try: await status_msg.edit_text(f"⚠️ **[{drive_name} CAS拉取失败]**\n❌ `{cas_file_name}`\n云端未生成或直链被拦截。")
                except: pass
            else:
                try: await app.send_message(COMMAND_CENTER_CHAT, f"⚠️ **[{drive_name} CAS拉取失败]**\n❌ `{cas_file_name}`\n云端未生成或直链被拦截。")
                except: pass
    except Exception as e:
        await notify_steward_log(f"⚠️ [后台CAS派发失败]: {e}", level="WARNING")

# 🔥 核心更新：精简 5000 端口生成 STRM 和入库日志
async def trigger_strm_sync(url, file_name, drive_name):
    try:
        await asyncio.sleep(5)  # 稍微等待 5 秒，确保 OpenList 云端缓存已刷新
        async with httpx.AsyncClient(timeout=15.0) as client:
            resp = await client.get(url)
            if resp.status_code == 200:
                await notify_steward_log(f"🎬 [{drive_name} STRM同步触发] `{file_name}`")
            else:
                await notify_steward_log(f"⚠️ [{drive_name} STRM同步异常] HTTP {resp.status_code}", level="WARNING")
    except Exception as e:
        await notify_steward_log(f"⚠️ [{drive_name} STRM同步失败]: {e}", level="WARNING")

# =================================================================
# 🚀 【磁力专属】后台双轨接力直推
# =================================================================
async def magnet_upload_task(local_path, target_dir_189, cat, folder_name, folder_season, total_bytes, clean_base, file_season, ep_num, is_movie, standard_name, version_suffix="", status_msg=None, original_gid=None, enable_139=False, skip_189=False):
    msg_189 = status_msg
    try:
        # ----- 阶段 1：天翼 189 高速直推 -----
        target_full_189 = f"{target_dir_189}/{standard_name}".replace("//", "/")
        await asyncio.sleep(2)  
        
        if not skip_189:
            retry_count = 0
            while os.path.exists(local_path):
                try:
                    put_url = f"{OLIST_URL}/api/fs/put"
                    headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full_189), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                    
                    # 🔥 统一使用 GLOBAL_UPLOAD_LOCK 确保与 TG 下载任务严格排队
                    async with GLOBAL_UPLOAD_LOCK:
                        start_time = time.time()
                        last_ui_time = start_time
                        
                        with open(local_path, "rb") as f_upload:
                            async def file_iter():
                                nonlocal last_ui_time
                                sent = 0
                                chunk_size = 1024 * 1024
                                while True:
                                    chunk = f_upload.read(chunk_size)
                                    if not chunk: break
                                    sent += len(chunk)
                                    pct = int(sent * 100 / total_bytes)
                                    
                                    now = time.time()
                                    if now - last_ui_time > 8.0 or sent == total_bytes:
                                        last_ui_time = now
                                        duration = now - start_time
                                        speed_bps = sent / duration if duration > 0 else 0
                                        speed_mb = speed_bps / (1024 * 1024)
                                        rem_bytes = total_bytes - sent
                                        eta_sec = rem_bytes / speed_bps if speed_bps > 0 else 0
                                        eta_txt = f"{int(eta_sec // 60)}分{int(eta_sec % 60)}秒" if eta_sec > 60 else f"{int(eta_sec)}秒"
                                        
                                        if msg_189:
                                            try: await msg_189.edit_text(f"🚀 **[磁力推天翼]** `{standard_name}`\n📈 速度: **{speed_mb:.2f} MB/s**\n⏳ 进度: **{pct}%** | 剩余: **{eta_txt}**")
                                            except: pass
                                    yield chunk
                                    
                            async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=None, write=None, pool=None), trust_env=False) as h: 
                                resp = await h.put(put_url, content=file_iter(), headers=headers)
                        
                    if resp.status_code != 200:
                        raise Exception(f"HTTP状态码异常: {resp.status_code}")
                        
                    resp_data = resp.json()
                    if resp_data.get("code") == 200:
                        ver_tag = f".{version_suffix}" if version_suffix else ""
                        if not is_movie and ep_num is not None:
                            record_history(clean_base, file_season, ep_num, version_suffix)
                            await notify_steward_log(f"📝 [云端入账] {clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}")
                        
                        try:
                            current_mount = get_mount_root()
                            STAGING_BASE_DIR = "/storage/emulated/0/Download/189cas"
                            sub_path = target_dir_189.replace(current_mount, "", 1)
                            local_cas_dir = f"{STAGING_BASE_DIR}{sub_path}"
                            os.makedirs(local_cas_dir, exist_ok=True)
                            final_cas_path = os.path.join(local_cas_dir, f"{standard_name}.cas")
                            asyncio.create_task(bg_fetch_cas_task(f"{target_full_189}.cas", final_cas_path, sub_path, f"{standard_name}.cas", status_msg=msg_189, olist_url=OLIST_URL, olist_token=OLIST_TOKEN, drive_name="189"))
                        except Exception: pass
                        break # 跳出 189 重试循环
                    else:
                        raise Exception(f"API报错: {resp_data.get('message', '未知错误')}")
                except Exception as e:
                    retry_count += 1
                    delay = 120 if retry_count == 1 else (300 if retry_count == 2 else (600 if retry_count == 3 else (900 if retry_count == 4 else 1800)))
                    
                    if msg_189 and retry_count < 5:
                        try: await msg_189.edit_text(f"⚠️ **[189上传异常]** `{standard_name}`\n❌ 报错: `{str(e)[:50]}`\n⏳ 等待重试 ({retry_count}/5)...")
                        except: pass
                    elif retry_count == 5 and msg_189:
                        try: await msg_189.edit_text(f"⚠️ **[189转后台重试]** `{standard_name}`\n已失败5次，转入后台静默重试。")
                        except: pass
                        
                    await notify_steward_log(f"⚠️ [189重试] {standard_name}: {str(e)[:100]}, 等待 {delay}s", level="WARNING")
                    await asyncio.sleep(delay)
                    
        else:
            if msg_189:
                try: await msg_189.edit_text(f"⏭️ **[跳过天翼 189]** `{standard_name}`\n已指定仅推 139，直接切入移动云盘队列...")
                except: pass

        # ----- 阶段 2：移动 139 限速接力 (140 占位 + CAS 战术) -----
        if enable_139 and os.path.exists(local_path):
            rel_category_path = f"{cat}/{folder_name}" if is_movie else f"{cat}/{folder_name}/Season {folder_season}"
            
            msg_chat_id = msg_189.chat.id if msg_189 else COMMAND_CENTER_CHAT
            msg_139 = None
            if msg_189:
                try: msg_139 = await app.send_message(msg_chat_id, f"🚀 **[移动139战术等待中]** `{standard_name}`\n🚦 正在排队，等待前面任务执行完毕...")
                except: pass
                
            retry_count = 0
            step_139 = 1  
            
            while os.path.exists(local_path):
                try:
                    temp_cloud_dir = "/140/139cas"
                    temp_cloud_path = f"{temp_cloud_dir}/{standard_name}".replace("//", "/")
                    put_url = f"{OLIST_URL}/api/fs/put" 
                    final_cloud_dir = f"/139/139cas/{rel_category_path}" 
                    
                    async with GLOBAL_139_UPLOAD_LOCK:
                        if step_139 == 1:
                            asyncio.create_task(notify_steward_log(f"🟢 [139战术] 开始140占位及CAS计算: {standard_name}"))
                        
                        # 1. 140 占位
                        if step_139 <= 1:
                            headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(temp_cloud_path), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                            start_time = time.time()
                            last_ui_time = start_time
                            
                            with open(local_path, "rb") as f_upload:
                                async def file_iter_139():
                                    nonlocal last_ui_time
                                    sent = 0
                                    chunk_size = 1024 * 1024 
                                    while True:
                                        chunk = f_upload.read(chunk_size)
                                        if not chunk: break
                                        sent += len(chunk)
                                        pct = int(sent * 100 / total_bytes)
                                        
                                        now = time.time()
                                        if now - last_ui_time > 8.0 or sent == total_bytes:
                                            last_ui_time = now
                                            duration = now - start_time
                                            speed_bps = sent / duration if duration > 0 else 0
                                            speed_mb = speed_bps / (1024 * 1024)
                                            rem_bytes = total_bytes - sent
                                            eta_sec = rem_bytes / speed_bps if speed_bps > 0 else 0
                                            eta_txt = f"{int(eta_sec // 60)}分{int(eta_sec % 60)}秒" if eta_sec > 60 else f"{int(eta_sec)}秒"
                                            
                                            if msg_139:
                                                try: await msg_139.edit_text(f"🚀 **[移动140占位]** `{standard_name}`\n📈 速度: **{speed_mb:.2f} MB/s**\n⏳ 进度: **{pct}%** | 剩余: **{eta_txt}**")
                                                except: pass
                                        yield chunk
                                        
                                async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=None, write=None, pool=None), trust_env=False) as h: 
                                    resp = await h.put(put_url, content=file_iter_139(), headers=headers)
                                
                                if resp.status_code != 200 or resp.json().get("code") != 200:
                                    raise Exception(f"140占位上传失败: HTTP {resp.status_code}")

                            step_139 = 2  

                        # 2. 本地计算 CAS
                        if step_139 <= 2:
                            if msg_139:
                                try: await msg_139.edit_text(f"🚀 **[移动139战术]** `{standard_name}`\n✅ 140 占位成功！\n⏳ 正在提取微型 CAS 指纹...")
                                except: pass
                                
                            cas_script_path = os.path.join(BASE_DIR, "cas_server.py")
                            cmd_cas = [
                                "python3", cas_script_path, 
                                "--cli", "--file", local_path, "--cloud", "139", "--category", rel_category_path
                            ]
                            proc_cas = await asyncio.create_subprocess_exec(*cmd_cas, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
                            stdout_data, stderr_data = await proc_cas.communicate()
                            
                            out_log = stdout_data.decode('utf-8', errors='ignore').strip()
                            err_log = stderr_data.decode('utf-8', errors='ignore').strip()
                            if err_log or "error" in out_log.lower() or "failed" in out_log.lower():
                                await notify_steward_log(f"⚠️ [CAS提取异常] {err_log or out_log}", level="WARNING")
                            step_139 = 3  

                        # 3. 将 CAS 上传到 139
                        if step_139 <= 3:
                            local_cas_path_1 = f"/storage/emulated/0/Download/139cas/{rel_category_path}/{standard_name}.cas".replace("//", "/")
                            local_cas_path_2 = f"{local_path}.cas"
                            local_cas_path_3 = os.path.splitext(local_path)[0] + ".cas"
                            local_cas_path_4 = os.path.join(BASE_DIR, "cas_output", f"{standard_name}.cas") 
                            
                            actual_cas_path = None
                            for cp in [local_cas_path_1, local_cas_path_2, local_cas_path_3, local_cas_path_4]:
                                if os.path.exists(cp):
                                    actual_cas_path = cp
                                    break
                                    
                            if not actual_cas_path:
                                raise Exception("CAS提取失败，找不到 .cas 文件！")

                            local_cas_path_final = actual_cas_path

                            if msg_139:
                                try: await msg_139.edit_text(f"🚀 **[移动139战术]** `{standard_name}`\n📤 CAS 提取成功！正在写入 139 目录...")
                                except: pass
                                
                            final_cloud_path = f"{final_cloud_dir}/{standard_name}.cas".replace("//", "/")
                            cas_bytes = os.path.getsize(local_cas_path_final)
                            headers_cas = {"Authorization": OLIST_TOKEN, "File-Path": quote(final_cloud_path), "Content-Length": str(cas_bytes), "Content-Type": "application/octet-stream"}
                            
                            async with httpx.AsyncClient(timeout=30.0) as h:
                                with open(local_cas_path_final, "rb") as f_cas:
                                    cas_resp = await h.put(put_url, content=f_cas.read(), headers=headers_cas)
                                    if cas_resp.status_code != 200 or cas_resp.json().get("code") != 200:
                                        raise Exception("CAS文件上传到139失败")

                            step_139 = 4  

                        # 4. 删除 140 占位大文件
                        if step_139 <= 4:
                            if msg_139:
                                try: await msg_139.edit_text(f"🚀 **[移动139战术]** `{standard_name}`\n🗑️ 狸猫换太子！抹除 140 占位大文件，释放云盘空间...")
                                except: pass
                                
                            remove_url = f"{OLIST_URL}/api/fs/remove"
                            remove_payload = {"names": [standard_name], "dir": temp_cloud_dir}
                            async with httpx.AsyncClient(timeout=30.0) as h:
                                try: 
                                    rem_resp = await h.post(remove_url, json=remove_payload, headers={"Authorization": OLIST_TOKEN, "Content-Type": "application/json"})
                                    if rem_resp.status_code == 200 and rem_resp.json().get("code") == 200:
                                        await notify_steward_log(f"✅ [139战术] 140占位文件删除成功: {standard_name}")
                                    else:
                                        await notify_steward_log(f"⚠️ [139战术] 抹除占位失败, 状态码:{rem_resp.status_code}, 返回:{rem_resp.text}", level="WARNING")
                                except Exception as rem_err: 
                                    await notify_steward_log(f"⚠️ [139战术] 抹除占位异常: {rem_err}", level="WARNING")
                            step_139 = 5  

                        # 5. 触发 5000 管家入库
                        if step_139 <= 5:
                            sync_url = f"{STEWARD_BASE_URL}/api/sync?drive=139&path={quote(final_cloud_dir)}"
                            asyncio.create_task(trigger_strm_sync(sync_url, f"{standard_name}.cas", "139"))
                            
                            if msg_139:
                                try: await msg_139.edit_text(f"🎉 **[139 战术大圆满]** `{standard_name}`\n✅ 空间已白嫖，CAS入库指令已发射！")
                                except: pass
                            
                            break # 跳出 139 重试循环
                        
                except Exception as e:
                    retry_count += 1
                    delay = 120 if retry_count == 1 else (300 if retry_count == 2 else (600 if retry_count == 3 else (900 if retry_count == 4 else 1800)))
                    if msg_139 and retry_count < 5:
                        try: await msg_139.edit_text(f"⚠️ **[139战术异常]** `{standard_name}`\n❌ 报错: `{str(e)[:50]}`\n⏳ 等待重试 ({retry_count}/5)...")
                        except: pass
                    elif retry_count == 5 and msg_139:
                        try: await msg_139.edit_text(f"⚠️ **[139转后台重试]** `{standard_name}`\n已失败5次，转入后台静默重试。")
                        except: pass
                    await notify_steward_log(f"⚠️ [139战术重试] {standard_name}: {str(e)[:100]}, 等待 {delay}s", level="WARNING")
                    await asyncio.sleep(delay)
            
        success_msg = f"🎉 **[双端圆满落盘]** ➔ `{standard_name}`\n✅ 该文件已成功推送，释放本地空间！"
        if msg_189:
            try: await msg_189.edit_text(success_msg)
            except: pass
        else:
            try: await app.send_message(COMMAND_CENTER_CHAT, success_msg)
            except Exception:
                try: await app.send_message("me", success_msg)
                except: pass

    finally:
        if os.path.exists(local_path):
            try: os.remove(local_path)
            except: pass

# =================================================================
# 🚀 【TG专属】后台双轨接力直推
# =================================================================
async def bg_upload_retry_task(local_path, target_dir_189, cat, folder_name, folder_season, total_bytes, clean_base, file_season, ep_num, is_movie, standard_name, history_tags="", status_msg=None, enable_139=False):
    task_lock_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{history_tags}" if not is_movie else f"{clean_base}{history_tags}"
    try:
        await asyncio.sleep(10)  
        
        # ----- 阶段 1：天翼 189 高速直推 -----
        target_full_189 = f"{target_dir_189}/{standard_name}".replace("//", "/")
        msg_189 = status_msg
        retry_count = 0
        
        while os.path.exists(local_path):
            try:
                put_url = f"{OLIST_URL}/api/fs/put"
                headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full_189), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                
                async with GLOBAL_UPLOAD_LOCK:
                    start_time = time.time()
                    last_ui_time = start_time
                    
                    with open(local_path, "rb") as f_upload:
                        async def file_iter():
                            nonlocal last_ui_time
                            sent = 0
                            chunk_size = 1024 * 1024 
                            while True:
                                chunk = f_upload.read(chunk_size)
                                if not chunk: break
                                sent += len(chunk)
                                pct = int(sent * 100 / total_bytes)
                                
                                now = time.time()
                                if now - last_ui_time > 8.0 or sent == total_bytes:
                                    last_ui_time = now
                                    duration = now - start_time
                                    speed_bps = sent / duration if duration > 0 else 0
                                    speed_mb = speed_bps / (1024 * 1024)
                                    rem_bytes = total_bytes - sent
                                    eta_sec = rem_bytes / speed_bps if speed_bps > 0 else 0
                                    eta_txt = f"{int(eta_sec // 60)}分{int(eta_sec % 60)}秒" if eta_sec > 60 else f"{int(eta_sec)}秒"
                                    
                                    if msg_189:
                                        try: await msg_189.edit_text(f"🚀 **[天翼189云推]** `{standard_name}`\n📈 速度: **{speed_mb:.2f} MB/s**\n⏳ 进度: **{pct}%** | 剩余: **{eta_txt}**")
                                        except: pass
                                yield chunk
                                
                        async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=None, write=None, pool=None), trust_env=False) as h: 
                            resp = await h.put(put_url, content=file_iter(), headers=headers)
                        
                    if resp.status_code != 200:
                        raise Exception(f"HTTP状态码异常: {resp.status_code}")
                        
                    resp_data = resp.json()
                    if resp_data.get("code") == 200:
                        ver_tag = f".{history_tags}" if history_tags else ""
                        if not is_movie and ep_num is not None:
                            record_history(clean_base, file_season, ep_num, history_tags)
                            await notify_steward_log(f"📝 [后台重推-补录] {clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}")
                        
                        try:
                            current_mount = get_mount_root()
                            STAGING_BASE_DIR = "/storage/emulated/0/Download/189cas"
                            sub_path = target_dir_189.replace(current_mount, "", 1)
                            local_cas_dir = f"{STAGING_BASE_DIR}{sub_path}"
                            os.makedirs(local_cas_dir, exist_ok=True)
                            final_cas_path = os.path.join(local_cas_dir, f"{standard_name}.cas")
                            asyncio.create_task(bg_fetch_cas_task(f"{target_full_189}.cas", final_cas_path, sub_path, f"{standard_name}.cas", status_msg=msg_189, olist_url=OLIST_URL, olist_token=OLIST_TOKEN, drive_name="189"))
                        except Exception: pass
                        break # 跳出 189 重试循环
                    else:
                        raise Exception(f"API报错: {resp_data.get('message', '未知错误')}")
            except Exception as e:
                retry_count += 1
                delay = 120 if retry_count == 1 else (300 if retry_count == 2 else (600 if retry_count == 3 else (900 if retry_count == 4 else 1800)))
                
                if msg_189 and retry_count < 5:
                    try: await msg_189.edit_text(f"⚠️ **[189上传异常]** `{standard_name}`\n❌ 报错: `{str(e)[:50]}`\n⏳ 等待重试 ({retry_count}/5)...")
                    except: pass
                elif retry_count == 5 and msg_189:
                    try: await msg_189.edit_text(f"⚠️ **[189转后台重试]** `{standard_name}`\n已失败5次，转入后台静默重试。")
                    except: pass
                    
                await notify_steward_log(f"⚠️ [189重试] {standard_name}: {str(e)[:100]}, 等待 {delay}s", level="WARNING")
                await asyncio.sleep(delay)

        # ----- 阶段 2：移动 139 限速接力 (140 占位 + CAS 战术) -----
        if enable_139 and os.path.exists(local_path):
            rel_category_path = f"{cat}/{folder_name}" if is_movie else f"{cat}/{folder_name}/Season {folder_season}"
            
            msg_chat_id = msg_189.chat.id if msg_189 else COMMAND_CENTER_CHAT
            msg_139 = None
            if msg_189:
                try: msg_139 = await app.send_message(msg_chat_id, f"🚀 **[移动139战术等待中]** `{standard_name}`\n🚦 正在排队，等待前面任务执行完毕...")
                except: pass
                
            retry_count = 0
            step_139 = 1  
            
            while os.path.exists(local_path):
                try:
                    temp_cloud_dir = "/140/139cas"
                    temp_cloud_path = f"{temp_cloud_dir}/{standard_name}".replace("//", "/")
                    put_url = f"{OLIST_URL}/api/fs/put" 
                    final_cloud_dir = f"/139/139cas/{rel_category_path}" 
                    
                    async with GLOBAL_139_UPLOAD_LOCK:
                        if step_139 == 1:
                            asyncio.create_task(notify_steward_log(f"🟢 [139战术] 开始140占位及CAS计算: {standard_name}"))
                        
                        # 1. 140 占位
                        if step_139 <= 1:
                            headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(temp_cloud_path), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                            start_time = time.time()
                            last_ui_time = start_time
                            
                            with open(local_path, "rb") as f_upload:
                                async def file_iter_139():
                                    nonlocal last_ui_time
                                    sent = 0
                                    chunk_size = 1024 * 1024 
                                    while True:
                                        chunk = f_upload.read(chunk_size)
                                        if not chunk: break
                                        sent += len(chunk)
                                        pct = int(sent * 100 / total_bytes)
                                        
                                        now = time.time()
                                        if now - last_ui_time > 8.0 or sent == total_bytes:
                                            last_ui_time = now
                                            duration = now - start_time
                                            speed_bps = sent / duration if duration > 0 else 0
                                            speed_mb = speed_bps / (1024 * 1024)
                                            rem_bytes = total_bytes - sent
                                            eta_sec = rem_bytes / speed_bps if speed_bps > 0 else 0
                                            eta_txt = f"{int(eta_sec // 60)}分{int(eta_sec % 60)}秒" if eta_sec > 60 else f"{int(eta_sec)}秒"
                                            
                                            if msg_139:
                                                try: await msg_139.edit_text(f"🚀 **[移动140占位]** `{standard_name}`\n📈 速度: **{speed_mb:.2f} MB/s**\n⏳ 进度: **{pct}%** | 剩余: **{eta_txt}**")
                                                except: pass
                                        yield chunk
                                        
                                async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=None, write=None, pool=None), trust_env=False) as h: 
                                    resp = await h.put(put_url, content=file_iter_139(), headers=headers)
                                
                                if resp.status_code != 200 or resp.json().get("code") != 200:
                                    raise Exception(f"140占位上传失败: HTTP {resp.status_code}")

                            step_139 = 2  

                        # 2. 本地计算 CAS
                        if step_139 <= 2:
                            if msg_139:
                                try: await msg_139.edit_text(f"🚀 **[移动139战术]** `{standard_name}`\n✅ 140 占位成功！\n⏳ 正在提取微型 CAS 指纹...")
                                except: pass
                                
                            cas_script_path = os.path.join(BASE_DIR, "cas_server.py")
                            cmd_cas = [
                                "python3", cas_script_path, 
                                "--cli", "--file", local_path, "--cloud", "139", "--category", rel_category_path
                            ]
                            proc_cas = await asyncio.create_subprocess_exec(*cmd_cas, stdout=asyncio.subprocess.PIPE, stderr=asyncio.subprocess.PIPE)
                            stdout_data, stderr_data = await proc_cas.communicate()
                            
                            out_log = stdout_data.decode('utf-8', errors='ignore').strip()
                            err_log = stderr_data.decode('utf-8', errors='ignore').strip()
                            if err_log or "error" in out_log.lower() or "failed" in out_log.lower():
                                await notify_steward_log(f"⚠️ [CAS提取异常] {err_log or out_log}", level="WARNING")
                            step_139 = 3  

                        # 3. 将 CAS 上传到 139
                        if step_139 <= 3:
                            local_cas_path_1 = f"/storage/emulated/0/Download/139cas/{rel_category_path}/{standard_name}.cas".replace("//", "/")
                            local_cas_path_2 = f"{local_path}.cas"
                            local_cas_path_3 = os.path.splitext(local_path)[0] + ".cas"
                            local_cas_path_4 = os.path.join(BASE_DIR, "cas_output", f"{standard_name}.cas") 
                            
                            actual_cas_path = None
                            for cp in [local_cas_path_1, local_cas_path_2, local_cas_path_3, local_cas_path_4]:
                                if os.path.exists(cp):
                                    actual_cas_path = cp
                                    break
                                    
                            if not actual_cas_path:
                                raise Exception("CAS提取失败，找不到 .cas 文件！")

                            local_cas_path_final = actual_cas_path

                            if msg_139:
                                try: await msg_139.edit_text(f"🚀 **[移动139战术]** `{standard_name}`\n📤 CAS 提取成功！正在写入 139 目录...")
                                except: pass
                                
                            final_cloud_path = f"{final_cloud_dir}/{standard_name}.cas".replace("//", "/")
                            cas_bytes = os.path.getsize(local_cas_path_final)
                            headers_cas = {"Authorization": OLIST_TOKEN, "File-Path": quote(final_cloud_path), "Content-Length": str(cas_bytes), "Content-Type": "application/octet-stream"}
                            
                            async with httpx.AsyncClient(timeout=30.0) as h:
                                with open(local_cas_path_final, "rb") as f_cas:
                                    cas_resp = await h.put(put_url, content=f_cas.read(), headers=headers_cas)
                                    if cas_resp.status_code != 200 or cas_resp.json().get("code") != 200:
                                        raise Exception("CAS文件上传到139失败")

                            step_139 = 4  

                        # 4. 删除 140 占位大文件
                        if step_139 <= 4:
                            if msg_139:
                                try: await msg_139.edit_text(f"🚀 **[移动139战术]** `{standard_name}`\n🗑️ 狸猫换太子！抹除 140 占位大文件，释放云盘空间...")
                                except: pass
                                
                            remove_url = f"{OLIST_URL}/api/fs/remove"
                            remove_payload = {"names": [standard_name], "dir": temp_cloud_dir}
                            async with httpx.AsyncClient(timeout=30.0) as h:
                                try: 
                                    rem_resp = await h.post(remove_url, json=remove_payload, headers={"Authorization": OLIST_TOKEN, "Content-Type": "application/json"})
                                    if rem_resp.status_code == 200 and rem_resp.json().get("code") == 200:
                                        await notify_steward_log(f"✅ [139战术] 140占位文件删除成功: {standard_name}")
                                    else:
                                        await notify_steward_log(f"⚠️ [139战术] 抹除占位失败, 状态码:{rem_resp.status_code}, 返回:{rem_resp.text}", level="WARNING")
                                except Exception as rem_err: 
                                    await notify_steward_log(f"⚠️ [139战术] 抹除占位异常: {rem_err}", level="WARNING")
                            step_139 = 5  

                        # 5. 触发 5000 管家入库
                        if step_139 <= 5:
                            sync_url = f"{STEWARD_BASE_URL}/api/sync?drive=139&path={quote(final_cloud_dir)}"
                            asyncio.create_task(trigger_strm_sync(sync_url, f"{standard_name}.cas", "139"))
                            
                            if msg_139:
                                try: await msg_139.edit_text(f"🎉 **[139 战术大圆满]** `{standard_name}`\n✅ 空间已白嫖，CAS入库指令已发射！")
                                except: pass
                            
                            break # 跳出 139 重试循环
                        
                except Exception as e:
                    retry_count += 1
                    delay = 120 if retry_count == 1 else (300 if retry_count == 2 else (600 if retry_count == 3 else (900 if retry_count == 4 else 1800)))
                    if msg_139 and retry_count < 5:
                        try: await msg_139.edit_text(f"⚠️ **[139战术异常]** `{standard_name}`\n❌ 报错: `{str(e)[:50]}`\n⏳ 等待重试 ({retry_count}/5)...")
                        except: pass
                    elif retry_count == 5 and msg_139:
                        try: await msg_139.edit_text(f"⚠️ **[139转后台重试]** `{standard_name}`\n已失败5次，转入后台静默重试。")
                        except: pass
                    await notify_steward_log(f"⚠️ [139战术重试] {standard_name}: {str(e)[:100]}, 等待 {delay}s", level="WARNING")
                    await asyncio.sleep(delay)
            
        success_msg = f"🎉 **[双端圆满落盘]** ➔ `{standard_name}`\n✅ 该文件已成功推送，释放本地空间！"
        if msg_189:
            try: await msg_189.edit_text(success_msg)
            except: pass
        else:
            try: await app.send_message(COMMAND_CENTER_CHAT, success_msg)
            except Exception:
                try: await app.send_message("me", success_msg)
                except: pass

    finally:
        if os.path.exists(local_path):
            try: os.remove(local_path)
            except: pass
        GLOBAL_ACTIVE_LOCKS.discard(task_lock_key)

# =================================================================
# 🧲 Aria2 异步执行任务 (磁力引擎)
# =================================================================
async def handle_magnet_execution(client, message, magnet_link, config_text, resume_gid=None):
    args = config_text.split()
    if len(args) < 2: 
        if message: 
            try: await message.reply_text("⚠️ 格式错误。请至少提供 `剧名` 和 `分类`。")
            except: pass
        return

    # 🔥 拦截 c139 和 proxy
    enable_139 = False
    use_proxy = False
    for i in range(len(args)-1, -1, -1):
        if args[i].lower() == "c139":
            enable_139 = True
            args.pop(i)
        elif args[i].lower() == "proxy":
            use_proxy = True
            args.pop(i)

    STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]
    cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
    if cat_idx == -1: 
        if message: 
            try: await message.reply_text("⚠️ 必须提供有效分类(如 国漫, 华语剧, 电影 等)。")
            except: pass
        return
    category = args.pop(cat_idx)

    custom_year = next((args.pop(i) for i, arg in enumerate(args) if arg.isdigit() and len(arg) == 4 and (1900 < int(arg) < 2100)), None)
    
    file_season = None
    folder_season = None
    version_suffix = ""
    for i in range(len(args)-1, -1, -1):
        if re.match(r'^s\d+$', args[i], re.IGNORECASE): 
            file_season = int(args.pop(i)[1:])
        elif args[i].lower().startswith("v="):
            version_suffix = args.pop(i)[2:]

    if args and args[-1].isdigit(): folder_season = int(args.pop(-1))
    if folder_season is None: folder_season = file_season if file_season is not None else 1

    # 🔥 修正：针对磁力任务，防止参数被抽空变成“未知磁力”
    raw_input = " ".join(args).strip()
    if not raw_input:
        if message:
            try: await message.reply_text("⚠️ **磁力解析失败**：未能提取到剧名！\n参数可能被系统误判全部抽走（如纯数字剧名）。请检查格式。")
            except: pass
        return
        
    if "|" in raw_input: search_kw, drama_name = [x.strip() for x in raw_input.split("|", 1)]
    else: search_kw = drama_name = raw_input

    year, _ = await fetch_tmdb_details(drama_name)
    if custom_year: year = custom_year

    folder_name = f"{drama_name} ({year})" if year else drama_name
    if version_suffix: folder_name = f"{folder_name} {version_suffix}"
    is_movie = "电影" in category or category in ["演唱会", "纪录片"]

    magnet_task_key = f"磁力_{drama_name}"
    GLOBAL_ACTIVE_LOCKS.add(magnet_task_key)
    
    original_gid = None

    try:
        if not resume_gid:
            # 🔥 新增 Aria2 代理注入逻辑 (此处 7890 替换为你的本地 HTTP 代理端口)
            aria2_options = {"dir": ARIA2_DOWNLOAD_DIR}
            if use_proxy:
                aria2_options["all-proxy"] = "http://127.0.0.1:7890"

            payload = {
                "jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.addUri",
                "params": [f"token:{ARIA2_RPC_SECRET}", [magnet_link], aria2_options]
            }

            try:
                if message: status_msg = await message.reply_text(f"🚀 **[Aria2 涡轮点火]**\n{'🔗 **已开启 Tracker 代理加速**' if use_proxy else ''}\n正在将磁力喂给后端 P2P 引擎...")
                else: status_msg = await client.send_message(COMMAND_CENTER_CHAT, f"🚀 **[Aria2 涡轮点火]**\n{'🔗 **已开启 Tracker 代理加速**' if use_proxy else ''}\n正在将磁力喂给后端 P2P 引擎...")
                
                # 🔥 补回丢失的核心代码，不再报缩进错误！
                async with httpx.AsyncClient(timeout=10.0) as h_client:
                    resp = await h_client.post(ARIA2_RPC_URL, json=payload)
                    res_json = resp.json()
                    if "error" in res_json:
                        error_msg = res_json["error"].get("message", "")
                        if "already registered" in error_msg.lower():
                            try: await status_msg.edit_text(f"❌ **添加失败**: 任务已在 Aria2 队列中，请勿重复添加！")
                            except: pass
                            return 
                        else:
                            try: await status_msg.edit_text(f"❌ Aria2 拒绝接单: `{error_msg}`")
                            except: pass
                            return 
                    gid = res_json.get("result")
                    if not gid: 
                        try: await status_msg.edit_text(f"❌ Aria2 未返回 GID！")
                        except: pass
                        return 
                    
                    original_gid = gid
                    add_magnet_task(original_gid, magnet_link, config_text)
                    tracked_gids = {gid} 
                    GLOBAL_TRACKED_GIDS.add(gid)
                    
            except Exception as e:
                try: await status_msg.edit_text(f"❌ Aria2 连接失败: {e}")
                except: pass
                return
        else:
            gid = resume_gid
            original_gid = resume_gid
            tracked_gids = {gid}
            GLOBAL_TRACKED_GIDS.add(gid)
            add_magnet_task(original_gid, magnet_link, config_text)
            
            if message: status_msg = await message.reply_text(f"🔄 **[任务恢复]** 重新接管磁力任务 `{drama_name}`...")
            else: status_msg = await client.send_message(COMMAND_CENTER_CHAT, f"🔄 **[任务恢复]** 重新接管磁力任务 `{drama_name}`...")

        last_ui_time = 0
        meta_wait_time = 0
        st_data = {}
        
        notified_50 = False
        notified_100 = False
        
        while True:
            await asyncio.sleep(3.0)
            
            if GLOBAL_STOP_SWEEP or "ALL" in GLOBAL_CANCEL_TASKS or magnet_task_key in GLOBAL_CANCEL_TASKS:
                await wipe_magnet_task(tracked_gids)
                if original_gid: remove_magnet_task(original_gid)
                try: await status_msg.edit_text(f"🛑 **[任务拉闸]** 磁力任务 `{drama_name}` 已被强行单独销毁！")
                except: pass
                return 
                
            try:
                p2 = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellStatus", "params": [f"token:{ARIA2_RPC_SECRET}", gid]}
                async with httpx.AsyncClient(timeout=5.0) as h_client:
                    resp2 = await h_client.post(ARIA2_RPC_URL, json=p2)
                    res_json = resp2.json()
                    
                    if "error" in res_json:
                        try: await status_msg.edit_text("🛑 任务已从Aria2面板中消失或被外部手动删除！")
                        except: pass
                        return
                        
                    st_data = res_json.get("result", {})

                followed_by = st_data.get("followedBy")
                if followed_by and len(followed_by) > 0:
                    gid = followed_by[0]  
                    tracked_gids.add(gid)
                    GLOBAL_TRACKED_GIDS.add(gid)
                    meta_wait_time = 0
                    notified_50 = False
                    notified_100 = False
                    
                    try: await status_msg.edit_text(f"🧲 **[解析成功]** 获取到种子文件，开始正式拉取实体视频...")
                    except: pass
                    continue

                status = st_data.get("status")
                if not status: continue
                
                info_hash = st_data.get("infoHash")
                bt_info = st_data.get("bittorrent", {})
                is_metadata = False
                
                if info_hash and "info" not in bt_info:
                    is_metadata = True

                if is_metadata:
                    if status == "error":
                        try: await status_msg.edit_text(f"❌ **种子解析报错**: {st_data.get('errorMessage')}")
                        except: pass
                        return
                    
                    if status == "complete":
                        try:
                            p_active = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellActive", "params": [f"token:{ARIA2_RPC_SECRET}"]}
                            p_waiting = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellWaiting", "params": [f"token:{ARIA2_RPC_SECRET}", 0, 100]}
                            p_stopped = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellStopped", "params": [f"token:{ARIA2_RPC_SECRET}", 0, 100]}
                            async with httpx.AsyncClient() as h:
                                res_a = await h.post(ARIA2_RPC_URL, json=p_active)
                                res_w = await h.post(ARIA2_RPC_URL, json=p_waiting)
                                res_s = await h.post(ARIA2_RPC_URL, json=p_stopped)
                                tasks = res_a.json().get("result", []) + res_w.json().get("result", []) + res_s.json().get("result", [])
                                for t in tasks:
                                    if t.get("infoHash") == info_hash and "info" in t.get("bittorrent", {}):
                                        gid = t.get("gid")
                                        tracked_gids.add(gid)
                                        GLOBAL_TRACKED_GIDS.add(gid)
                                        try: await status_msg.edit_text(f"🧲 **[解析成功]** 跨队列捕获实体任务，无缝切入...")
                                        except: pass
                                        break
                        except: pass
                    
                    meta_wait_time += 3
                    if meta_wait_time > 300:
                        await wipe_magnet_task(tracked_gids)
                        try: await status_msg.edit_text("❌ **超时中止**：连续 5 分钟未获取到种子(死链)，已销毁！")
                        except: pass
                        return
                    
                    now = time.time()
                    if now - last_ui_time > 6.0:
                        last_ui_time = now
                        try: await status_msg.edit_text(f"🧲 **[磁力涡轮]** `{drama_name}`\n⏳ 正在获取元数据(种子)中...")
                        except: pass
                    continue 

                if status == "error":
                    try: await status_msg.edit_text(f"❌ **Aria2 下载报错**: {st_data.get('errorMessage')}")
                    except: pass
                    return

                completed = int(st_data.get("completedLength", 0))
                total = int(st_data.get("totalLength", 0))

                if status == "active" and total > 0 and completed >= total:
                    asyncio.create_task(notify_steward_log(f"✅ [磁力涡轮] 进度达100%，主动结束状态监听，向清理流程移交控制权..."))
                    break

                if status in ["complete", "removed"]: 
                    break

                now = time.time()
                if now - last_ui_time > 6.0:
                    last_ui_time = now
                    dl_speed = int(st_data.get("downloadSpeed", 0))
                    up_speed = int(st_data.get("uploadSpeed", 0))
                    sp_mb = dl_speed / (1024*1024)
                    up_mb = up_speed / (1024*1024)
                    connections = st_data.get("connections", "0")
                    
                    if total > 0:
                        pct = int(completed * 100 / total)
                        eta_sec = (total - completed) / dl_speed if dl_speed > 0 else 0
                        eta_txt = f"{int(eta_sec//60)}分{int(eta_sec%60)}秒" if eta_sec > 0 else "计算中"
                        
                        if pct >= 50 and not notified_50:
                            asyncio.create_task(notify_steward_log(f"📥 [磁力下载] {drama_name} 已完成 50%"))
                            notified_50 = True
                        if pct == 100 and not notified_100:
                            asyncio.create_task(notify_steward_log(f"📥 [磁力下载] {drama_name} 已完成 100%"))
                            notified_100 = True
                    else:
                        pct = 0
                        eta_txt = "连接节点中..."

                    try: await status_msg.edit_text(f"🧲 **[磁力涡轮]** `{drama_name}`\n📈 下载: **{sp_mb:.2f} MB/s** | 📤 上传: **{up_mb:.2f} MB/s**\n🤝 连接: **{connections}** 节点\n⏳ 进度: **{pct}%** | 剩余: **{eta_txt}**")
                    except: pass
            except Exception: pass

        try: await status_msg.edit_text(f"✅ **[下载完毕]** 启动清道夫协议！\n正在全量检索下载物，提取视频并执行智能洗名...")
        except: pass

        files = st_data.get("files", [])
        VIDEO_EXTS = ['.mp4', '.mkv', '.ts', '.avi', '.rmvb', '.flv', '.webm', '.iso']
        extracted_videos = []
        
        for f in files:
            f_path = f.get("path")
            if not f_path or not os.path.exists(f_path): continue
            if os.path.getsize(f_path) <= 0: continue 

            if os.path.splitext(f_path)[1].lower() in VIDEO_EXTS:
                try:
                    orig_name = os.path.basename(f_path)
                    file_ep_num = extract_pure_episode(orig_name, drama_anchor=drama_name)
                    new_name = smart_rename(orig_name, drama_name, ep_num=file_ep_num, season=folder_season, is_movie=is_movie)
                    
                    safe_dest = os.path.join(LOCAL_TEMP_DIR, new_name)
                    if os.path.exists(safe_dest):
                        os.remove(safe_dest)
                    shutil.move(f_path, safe_dest)
                    
                    extracted_videos.append({
                        "path": safe_dest,
                        "name": new_name,
                        "size": os.path.getsize(safe_dest),
                        "ep_num": file_ep_num
                    })
                except Exception as e:
                    await notify_steward_log(f"⚠️ 提取视频 {f_path} 时出错: {e}", level="WARNING")

        await wipe_magnet_task(tracked_gids)

        if not extracted_videos:
            if st_data.get("status") == "removed": 
                try: await status_msg.edit_text("🛑 任务已被手动从后台终止并清理。")
                except: pass
                return
            try: await status_msg.edit_text("❌ 提取失败：该链接仅包含垃圾文件或死链残片。所有残留已干干净净地销毁。")
            except: pass
            return

        try: await status_msg.edit_text(f"🎉 **[清洗成功]** 共提取 {len(extracted_videos)} 个纯净视频！\n原始垃圾已焚毁，正由磁力专属双轨接力直推...")
        except: pass

        current_mount = get_mount_root()
        for idx, v in enumerate(extracted_videos):
            if is_movie: target_dir = f"{current_mount}/{category}/{folder_name}"
            else: target_dir = f"{current_mount}/{category}/{folder_name}/Season {folder_season}"
            
            pass_msg = status_msg if idx == 0 else None
            
            asyncio.create_task(magnet_upload_task(
                local_path=v['path'], 
                target_dir_189=target_dir, 
                cat=category,
                folder_name=folder_name,
                folder_season=folder_season,
                total_bytes=v['size'], 
                clean_base=drama_name.replace(" ", "."), 
                file_season=folder_season, 
                ep_num=v['ep_num'], 
                is_movie=is_movie, 
                standard_name=v['name'], 
                version_suffix=version_suffix,
                status_msg=pass_msg,
                original_gid=original_gid,
                enable_139=enable_139,
                skip_189=False
            ))
            
    finally:
        GLOBAL_ACTIVE_LOCKS.discard(magnet_task_key)
        GLOBAL_CANCEL_TASKS.discard(magnet_task_key)
        if original_gid:
            remove_magnet_task(original_gid)  # <--- 加上这两行，确保无论如何都会清理 JSON

# =================================================================
# 🎯 [指令响应大网关] 
# =================================================================
STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]

@app.on_message(filters.chat(COMMAND_CENTER_CHAT) & filters.command(["sub", "unsub", "list", "add", "del", "go", "history", "ping", "rm", "clean", "scan", "rmh", "setdir", "cancel", "mag", "reup", "clh"]) & filters.user("me"))
async def manage_system_commands(client, message):
    global GLOBAL_STOP_SWEEP
    command = message.command[0].lower()
    config = load_listener_config()
    
    # 🔥 核心更新：新增的本地遗留文件强制重推引擎
    if command == "reup":
        args = message.command[1:]
        if len(args) < 2: 
            return await message.reply_text("⚠️ 语法：`/reup [剧名关键字] [分类] [可选:年份] [c139|only139]`\n例如：`/reup 灿如繁星 欧美剧 2026 only139`")
        
        enable_139 = False
        skip_189 = False
        for i in range(len(args)-1, -1, -1):
            arg_lower = args[i].lower()
            if arg_lower == "c139":
                enable_139 = True
                args.pop(i)
            elif arg_lower == "only139":
                enable_139 = True
                skip_189 = True
                args.pop(i)
                
        # 提取可选的手动年份
        custom_year = next((args.pop(i) for i, arg in enumerate(args) if arg.isdigit() and len(arg) == 4 and (1900 < int(arg) < 2100)), None)
                
        # 提取分类
        cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
        if cat_idx != -1: category = args.pop(cat_idx)
        else: category = args.pop(-1)

        kw = " ".join(args).lower()
        drama_name_input = args[0]  # 🔥 提取纯中文剧名
        
        found_files = []
        if os.path.exists(LOCAL_TEMP_DIR):
            for f in os.listdir(LOCAL_TEMP_DIR):
                if kw in f.lower() and os.path.isfile(os.path.join(LOCAL_TEMP_DIR, f)):
                    found_files.append(f)
                    
        if not found_files:
            return await message.reply_text(f"❌ 在本地缓存文件夹(tg_temp)中未找到包含 `{kw}` 的视频文件，可能已被清理。")
            
        await message.reply_text(f"🔍 成功截获 {len(found_files)} 个未上传的本地遗留文件，启动本地补推引擎！")
        
        for fname in found_files:
            local_path = os.path.join(LOCAL_TEMP_DIR, fname)
            total_bytes = os.path.getsize(local_path)
            
            # 智能反向推导剧名、季数、集数
            m_tv = re.search(r'^(.*?)\.S(\d+)E(\d+)\.(.*?)$', fname, re.IGNORECASE)
            m_movie = re.search(r'^(.*?)\.(19\d{2}|20\d{2})\.(.*?)$', fname)
            
            is_movie = "电影" in category or category in ["演唱会", "纪录片"]
            year = ""
            clean_base = fname.split('.')[0]
            file_season = folder_season = ep_num = 1
            
            if m_tv and not is_movie:
                clean_base = m_tv.group(1)
                file_season = folder_season = int(m_tv.group(2))
                ep_num = int(m_tv.group(3))
                m_year = re.search(r'\b(20\d{2})\b', fname)
                if m_year: year = m_year.group(1)
            elif m_movie and is_movie:
                clean_base = m_movie.group(1)
                year = m_movie.group(2)
            
            # 🔥 年份优先级：手动指定 > 文件名提取 > 留空
            final_year = custom_year if custom_year else year
            folder_name = f"{drama_name_input} ({final_year})" if final_year else drama_name_input
            
            current_mount = get_mount_root()
            if is_movie: target_dir = f"{current_mount}/{category}/{folder_name}"
            else: target_dir = f"{current_mount}/{category}/{folder_name}/Season {folder_season}"
            
            status_msg = await message.reply_text(f"🚀 **[本地补录推流]** 准备上传 `{fname}` ...")
            
            # 直接调用磁力的双轨上传队列，跳过下载环节
            asyncio.create_task(magnet_upload_task(
                local_path=local_path, 
                target_dir_189=target_dir, 
                cat=category,
                folder_name=folder_name,
                folder_season=folder_season,
                total_bytes=total_bytes, 
                clean_base=clean_base, 
                file_season=file_season, 
                ep_num=ep_num, 
                is_movie=is_movie, 
                standard_name=fname, 
                version_suffix="",
                status_msg=status_msg,
                original_gid=None,
                enable_139=enable_139,
                skip_189=skip_189
            ))
        return

    if command == "ping":
        uptime_minutes = int((time.time() - START_TIME) / 60)
        return await message.reply_text(f"🟢 **系统健康度 [优秀]**\n⏱️ 存活: `{uptime_minutes}` 分钟")

    if command == "setdir":
        if len(message.command) < 2: return await message.reply_text(f"📁 当前上传目录: `{get_mount_root()}`\n⚠️ 语法：`/setdir [新目录路径]`")
        new_dir = message.command[1]
        set_mount_root(new_dir)
        return await message.reply_text(f"✅ 上传主目录已成功切换至: `{new_dir}`")

    if command == "clean":
        count = 0
        if os.path.exists(LOCAL_TEMP_DIR):
            for f in os.listdir(LOCAL_TEMP_DIR):
                try: os.remove(os.path.join(LOCAL_TEMP_DIR, f)); count += 1
                except: pass
        return await message.reply_text(f"🧹 抹除 `{count}` 个遗留碎片！")

    if command == "cancel":
        kw = " ".join(message.command[1:]).lower() if len(message.command) > 1 else "all"
        if kw == "all":
            GLOBAL_STOP_SWEEP = True
            GLOBAL_CANCEL_TASKS.add("ALL")
            GLOBAL_ACTIVE_LOCKS.clear()
            
            # 🔥 修复补丁：5秒后自动解除全局封印，防止永久阻塞后续新任务
            async def clear_all_lock():
                await asyncio.sleep(5)
                GLOBAL_CANCEL_TASKS.discard("ALL")
            asyncio.create_task(clear_all_lock())
            
            return await message.reply_text("🛑 已下达最高追杀令：全局清空 TG 队列、拉闸后台扫荡、销毁底层 Aria2 任务！")
        else:
            kws = kw.split()
            matched = [name for name in GLOBAL_ACTIVE_LOCKS if all(k in name.lower() for k in kws)]
            if not matched: return await message.reply_text(f"❌ 没找到包含 `{kw}` 的任务。")
            for name in matched: GLOBAL_CANCEL_TASKS.add(name)
            return await message.reply_text(f"🛑 已取消：\n" + "\n".join([f"`{name}`" for name in matched]))

    if command == "mag":
        args = message.command[1:]
        if not args or not args[0].startswith("magnet:?"): return await message.reply_text("⚠️ 语法：`/mag [磁力链接]`\n(建议直接把链接发给机器人)")
        magnet_link = args.pop(0)
        prompt = await message.reply_text("🧲 磁力链接已直接捕获！请回复: `剧名|别名 分类 [S季数] [年份] [v=版本] [c139]`")
        chat_id = message.chat.id
        if chat_id not in PENDING_MAGNETS:
            PENDING_MAGNETS[chat_id] = {}
        PENDING_MAGNETS[chat_id][prompt.id] = {"link": magnet_link, "mag_msg_id": message.id, "prompt_msg_id": prompt.id}
        return

    if command == "rm":
        if len(message.command) < 2: return await message.reply_text("⚠️ 语法：`/rm [航线剧名关键字]`")
        kw = " ".join(message.command[1:]).lower()
        routes = load_tg_routes()
        matched_keys = [k for k in routes.keys() if kw in k.lower()]
        if not matched_keys: return await message.reply_text(f"❌ 没找到包含 `{kw}` 的航线")
        for k in matched_keys: del routes[k]
        save_tg_routes(routes)
        return await message.reply_text(f"🗑️ 已彻底删除 {len(matched_keys)} 条航线记录！")

    if command == "rmh":
        if len(message.command) < 2: return await message.reply_text("⚠️ `/rmh [剧名关键字]`")
        kw = " ".join(message.command[1:]).lower()
        history = load_history()
        matched_keys = [k for k in history.keys() if kw in k.lower()]
        if not matched_keys: return await message.reply_text(f"❌ 历史库里没找到包含 `{kw}` 的记录")
        for k in matched_keys: del history[k]
        with open(TG_HISTORY_DB, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
        return await message.reply_text(f"🗑️ 已抹除 {len(matched_keys)} 条历史记录！")

    if command == "history":
        history = load_history()
        if not history: return await message.reply_text("📭 历史账本目前是空的。")
        sorted_hist = sorted(history.items(), key=lambda x: x[1], reverse=True)[:15]
        msg_text = "📜 **[最近入库记录]**\n\n"
        for k, v in sorted_hist:
            time_str = datetime.fromtimestamp(v).strftime('%m-%d %H:%M')
            msg_text += f"🔹 `{k}`  *(于 {time_str})*\n"
        return await message.reply_text(msg_text)
    
    if command == "clh":
        config = load_listener_config()
        history = load_history()
        
        if not history: 
            return await message.reply_text("📭 历史账本目前是空的，无需清理。")
        
        # 1. 提取当前所有在追订阅的剧名基准 (剔除版本号后缀，并转换为历史记录的 '.' 格式)
        active_bases = set()
        for chat_id, info in config.get("trusted_channels", {}).items():
            for drama_key, d_info in info.get("monitored_dramas", {}).items():
                db_version = d_info.get("version", "")
                pure_drama = drama_key[:-len(f"_{db_version}")] if db_version and drama_key.endswith(f"_{db_version}") else drama_key
                active_bases.add(pure_drama.replace(" ", "."))
                
        # 2. 遍历历史记录，找出已退订或不存在的孤儿记录
        keys_to_delete = []
        for hist_key in history.keys():
            is_active = False
            for base in active_bases:
                # 兼容历史键值格式：剧集通常带 .SxxExx，电影可能是纯名或带版本号
                if hist_key.startswith(base + ".") or hist_key == base:
                    is_active = True
                    break
            if not is_active:
                keys_to_delete.append(hist_key)
                
        # 3. 执行物理抹除并落盘保存
        if not keys_to_delete:
            return await message.reply_text("✅ 历史账本很干净，所有记录均匹配当前订阅，无陈旧孤儿记录。")
            
        for k in keys_to_delete:
            del history[k]
            
        with open(TG_HISTORY_DB, 'w', encoding='utf-8') as f:
            json.dump(history, f, ensure_ascii=False, indent=2)
            
        return await message.reply_text(f"🧹 **[无情清道夫]** 交叉比对完毕！\n🗑️ 共抹除了 `{len(keys_to_delete)}` 条已退订或陈旧的历史记录。")

    if command == "add":
        if len(message.command) < 3: return await message.reply_text("⚠️ `/add [频道ID] [别名]`")
        c_id, c_name = message.command[1], " ".join(message.command[2:])
        if c_id not in config["trusted_channels"]: config["trusted_channels"][c_id] = {"monitored_dramas": {}}
        config["trusted_channels"][c_id]["channel_name"] = c_name
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        return await message.reply_text(f"🏢 **频道挂载成功**: {c_name}")

    if command == "del":
        if len(message.command) < 2: return
        if message.command[1] in config["trusted_channels"]:
            del config["trusted_channels"][message.command[1]]
            with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
            return await message.reply_text("🗑️ 频道已拔除。")

    if command == "list":
        msg_text = "📡 **[雷达大盘状态]**\n\n"
        for chat_id, info in config.get("trusted_channels", {}).items():
            msg_text += f"🏢 **频道**: {info.get('channel_name', chat_id)}\n"
            dramas = info.get("monitored_dramas", {})
            if not dramas: msg_text += "  └ 📭 (空)\n"
            else:
                for d_name, d_info in dramas.items(): 
                    f_season = int(d_info.get('file_season', 1))
                    min_mb = d_info.get('min_mb', 0)
                    max_mb = d_info.get('max_mb', 999999)
                    freq = d_info.get('frequency', '日更')
                    end_ep = d_info.get('end_ep', 9999)
                    search_kw = d_info.get('search_kw', d_name)
                    ver = d_info.get('version', '')
                    e139 = " [139双传]" if d_info.get('enable_139') else ""
                    s_text = f"S{f_season:02d}"
                    size_txt = f" / {min_mb}-{max_mb if max_mb != 999999 else '上限'}MB" if (min_mb or max_mb != 999999) else ""
                    end_txt = f" / 🏁{end_ep}集" if end_ep != 9999 else " / ♾️连载"
                    ver_txt = f" / 🏷️{ver}" if ver else ""
                    line = f"  └ 🎬 `{search_kw}` ({s_text}{size_txt} / ⏱️{freq}{end_txt}{ver_txt}){e139}\n"
                    if len(msg_text) + len(line) > 4000:
                        await message.reply_text(msg_text)
                        msg_text = ""
                    msg_text += line
        if msg_text: await message.reply_text(msg_text)
        return

    if command == "scan":
        GLOBAL_STOP_SWEEP = False
        GLOBAL_CANCEL_TASKS.discard("ALL")  # 🔥 修复补丁：启动扫荡前，强制清空全局追杀标记
        args = message.command[1:]
        target_kw = " ".join(args).lower() if args else None
        await message.reply_text("🔍 **[手动扫荡触发]** ➔ 正在全自动翻找目标漏网之鱼...")
        for chat_id, info in config.get("trusted_channels", {}).items():
            for drama_name, d_info in info.get("monitored_dramas", {}).items():
                if target_kw and target_kw not in drama_name.lower(): continue
                folder_s = d_info.get("folder_season")
                file_s = d_info.get("file_season")
                if folder_s is None: folder_s = file_s if file_s is not None else 1
                if file_s is None: file_s = folder_s
                min_ep = d_info.get("min_ep", 1)
                cat = d_info.get("category", "未分类")
                min_mb = d_info.get("min_mb", 0)
                max_mb = d_info.get("max_mb", 999999)
                d_year = d_info.get("year")
                asyncio.create_task(sweep_existing_history(client, chat_id, drama_name, cat, folder_s, file_s, min_ep, min_mb, max_mb, fetch_limit=200, year=d_year))
        return

    if command == "sub":
        args = message.command[1:]
        if len(args) < 2: return await message.reply_text("⚠️ 语法：`/sub [剧名] [分类] [频率] [v=版本] [f=参数] [end=完结集数] [可选:年份/季数] [c139]`")
        
        # 🔥 拦截 c139 标识
        enable_139 = False
        for i in range(len(args)-1, -1, -1):
            if args[i].lower() == "c139":
                enable_139 = True
                args.pop(i)

        cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
        if cat_idx == -1: return await message.reply_text("⚠️ 请提供有效分类。")
        category = args.pop(cat_idx)
        
        freq_list = ["日更", "周一", "周二", "周三", "周四", "周五", "周六", "周日"]
        frequency = "日更"
        for i, arg in enumerate(args):
            if arg in freq_list: frequency = args.pop(i); break
        
        min_mb, max_mb = 0, 999999
        size_idx = next((i for i, arg in enumerate(args) if re.match(r'^\d+-\d+$', arg)), -1)
        if size_idx != -1:
            size_str = args.pop(size_idx)
            min_mb, max_mb = map(int, size_str.split('-'))
        
        replied = message.reply_to_message
        if replied and replied.forward_from_chat:
            specific_channel = str(replied.forward_from_chat.id)
            channel_title = replied.forward_from_chat.title or specific_channel
            if specific_channel not in config.setdefault("trusted_channels", {}):
                config["trusted_channels"][specific_channel] = {"channel_name": channel_title, "monitored_dramas": {}}
        else:
            specific_channel = next((args.pop(i) for i, arg in enumerate(args) if arg.startswith("-100")), None)
            
        if not specific_channel:
            return await message.reply_text("⚠️ **安全拦截**：未指定目标频道！\n为防止多频道标签(f=)相互覆盖导致串台，请**回复该频道的一条消息**发送 `/sub`，或在参数中指定 `-100xxx` 频道ID。")
            
        target_pools = [specific_channel]

        custom_year = next((args.pop(i) for i, arg in enumerate(args) if arg.isdigit() and len(arg) == 4 and (1900 < int(arg) < 2100)), None)
        version_suffix, file_suffix, end_ep = "", "", None
        for i in range(len(args)-1, -1, -1):
            if args[i].lower().startswith("v="): version_suffix = args.pop(i)[2:]
            elif args[i].lower().startswith("f="): file_suffix = args.pop(i)[2:]
            elif args[i].lower().startswith("end="): end_ep = int(args.pop(i)[4:])

        file_season, folder_season, min_ep = None, None, 1
        for i in range(len(args)-1, -1, -1):
            if re.match(r'^s\d+$', args[i], re.IGNORECASE): file_season = int(args.pop(i)[1:])
            elif re.match(r'^e\d+$', args[i], re.IGNORECASE): min_ep = int(args.pop(i)[1:])
                
        if args and args[-1].isdigit():
            val = int(args.pop(-1))
            if args and args[-1].isdigit():
                folder_season = int(args.pop(-1))
                min_ep = val
            else: min_ep = val
                
        if folder_season is None: folder_season = file_season if file_season is not None else 1
        if file_season is None: file_season = folder_season
        
        raw_input = " ".join(args) if args else "未知目标"
        if "|" in raw_input: search_kw, drama_name = [x.strip() for x in raw_input.split("|", 1)]
        else: search_kw = drama_name = raw_input

        drama_key = f"{drama_name}_{version_suffix}" if version_suffix else drama_name
        fetched_year, fetched_end = await fetch_tmdb_details(drama_name)
        if not custom_year: custom_year = fetched_year
        if end_ep is None: end_ep = 9999 if category in ["国漫", "日漫", "综艺", "短剧"] else fetched_end
                
        for chat_id in target_pools:
            if "monitored_dramas" not in config["trusted_channels"][chat_id]: config["trusted_channels"][chat_id]["monitored_dramas"] = {}
            config["trusted_channels"][chat_id]["monitored_dramas"][drama_key] = {
                "search_kw": search_kw, "version": version_suffix, "file_version": file_suffix, 
                "category": category, "folder_season": folder_season, "file_season": file_season, 
                "min_ep": min_ep, "min_mb": min_mb, "max_mb": max_mb, "frequency": frequency,
                "year": custom_year, "end_ep": end_ep, "enable_139": enable_139
            }
            
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        end_display = "♾️无限连载" if end_ep == 9999 else f"{end_ep} 集杀青"
        e139_display = "\n🚀 **附加**: 已开启139云盘同步接力" if enable_139 else ""
        
        try:
            await asyncio.sleep(2.5)
            await message.delete()
        except: pass
        await message.reply_text(f"✅ **订阅成功**: `{drama_name}`\n🏷️ **追踪**: `{version_suffix or '默认'}`\n📁 **路径**: `Season {folder_season}` (命名: `S{file_season:02d}`)\n⚖️ **画质锁定**: `{min_mb}-{max_mb} MB`\n🏁 **完结**: `{end_display}`{e139_display}\n🚀 **启动扫荡！**")
        
        for chat_id in target_pools:
            asyncio.create_task(sweep_existing_history(client, chat_id, drama_key, category, folder_season, file_season, min_ep, min_mb, max_mb, fetch_limit=200, year=custom_year))
        return
    
    if command in ["unsub", "del"]:
        args = message.command[1:]
        if not args: return await message.reply_text("⚠️ 语法：`/unsub [剧名] [可选: v=版本号] [可选:-100...]`")
        specific_channel = next((arg for arg in args if arg.startswith("-100")), None)
        if specific_channel: args.remove(specific_channel)
        version_suffix = ""
        for i, arg in enumerate(args):
            if arg.lower().startswith("v="): version_suffix = args.pop(i)[2:]; break
        drama_name = " ".join(args)
        if not drama_name: return await message.reply_text("⚠️ 请输入剧名！")
        drama_key = f"{drama_name}_{version_suffix}" if version_suffix else drama_name
        deleted_count = 0
        target_pools = [specific_channel] if specific_channel else list(config.get("trusted_channels", {}).keys())
        for chat_id in target_pools:
            if chat_id in config.get("trusted_channels", {}) and "monitored_dramas" in config["trusted_channels"][chat_id]:
                if drama_key in config["trusted_channels"][chat_id]["monitored_dramas"]:
                    del config["trusted_channels"][chat_id]["monitored_dramas"][drama_key]
                    deleted_count += 1
                elif drama_name in config["trusted_channels"][chat_id]["monitored_dramas"]:
                    del config["trusted_channels"][chat_id]["monitored_dramas"][drama_name]
                    deleted_count += 1
        if deleted_count > 0:
            with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
            return await message.reply_text(f"✅ **精确打击成功**\n💣 目标: `{drama_key}`\n📉 共从 **{deleted_count}** 个频道中移除！")
        else: return await message.reply_text(f"⚠️ **未找到目标**: `{drama_key}`")

    if command == "go":
        args = message.command[1:]
        if not args: return await message.reply_text("⚠️ 语法：`/go [剧名] [分类] [年份] [v=版本] [季数] [c139]`")
        
        # 🔥 拦截 c139
        enable_139 = False
        for i in range(len(args)-1, -1, -1):
            if args[i].lower() == "c139":
                enable_139 = True
                args.pop(i)

        routes = load_tg_routes()
        version_suffix, file_suffix = "", ""
        for i in range(len(args)-1, -1, -1):
            if args[i].lower().startswith("v="): version_suffix = args.pop(i)[2:]
            elif args[i].lower().startswith("f="): file_suffix = args.pop(i)[2:]
        custom_year = next((args.pop(i) for i, arg in enumerate(args) if arg.isdigit() and len(arg) == 4 and (1900 < int(arg) < 2100)), None)
        cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
        if cat_idx == -1:
            search_kw = " ".join(args).lower()
            matched_item = None
            for k, v in routes.items():
                if search_kw in k.lower() or search_kw in v.get("folder_name", "").lower():
                    matched_item = v; break
            if matched_item:
                folder_s = matched_item.get("folder_season", 1)
                file_s = matched_item.get("file_season", folder_s)
                GLOBAL_ROUTE_CACHE.update({"folder_name": matched_item["folder_name"], "category": matched_item["category"], "folder_season": folder_s, "file_season": file_s, "year": matched_item.get("year", ""), "version": matched_item.get("version", ""), "expire_time": time.time() + 300, "manual_ep": None, "enable_139": matched_item.get("enable_139", False)})
                
                try:
                    await asyncio.sleep(2.5)
                    await message.delete()
                except: pass
                return await message.reply_text(f"✅ 命中记忆航线！已锁定目标: `{matched_item['folder_name']}`")
            return await message.reply_text("⚠️ 未匹配到历史航线。")
        cat = args.pop(cat_idx)
        file_season, folder_season = None, None
        for i in range(len(args)-1, -1, -1):
            if re.match(r'^s\d+$', args[i], re.IGNORECASE): file_season = int(args.pop(i)[1:])
        if args and args[-1].isdigit(): folder_season = int(args.pop(-1))
        if folder_season is None: folder_season = file_season if file_season is not None else 1
        if file_season is None: file_season = folder_season 
        pure_title = " ".join(args)
        year, _ = await fetch_tmdb_details(pure_title)
        if custom_year: year = custom_year
        folder_name = f"{pure_title} ({year})" if year else pure_title
        if version_suffix: folder_name = f"{folder_name} {version_suffix}"
        routes[folder_name] = {"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "version": version_suffix, "file_version": file_suffix, "created_at": int(time.time()), "enable_139": enable_139}
        save_tg_routes(routes)
        GLOBAL_ROUTE_CACHE.update({"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "version": version_suffix, "file_version": file_suffix, "expire_time": time.time() + 360, "manual_ep": None, "enable_139": enable_139})
        
        try:
            await asyncio.sleep(2.5)
            await message.delete()
        except: pass
        e139_display = "\n🚀 **[已开启139双端接力]**" if enable_139 else ""
        return await message.reply_text(f"✅ 新航线已打通：\n📁 目标锁定: `{folder_name}`{e139_display}\n请直接转发视频！")

# 🔥 补丁：给磁力直贴功能也加上控制台专属锁，防止旧手机“跨界接单”
@app.on_message(filters.chat(COMMAND_CENTER_CHAT) & filters.text & filters.regex(r"(?i)^magnet:\?xt=") & filters.user("me"))
async def catch_magnet_link(client, message):
    magnet_link = message.text.strip()
    prompt = await message.reply_text("🧲 **[雷达警报] 已直接捕获磁力！**\n请直接回复本条消息提供归属信息：\n👉 **格式**: `剧名|别名 分类 [S季数] [年份] [v=版本] [c139] [proxy]`")
    chat_id = message.chat.id
    if chat_id not in PENDING_MAGNETS:
        PENDING_MAGNETS[chat_id] = {}
    PENDING_MAGNETS[chat_id][prompt.id] = {"link": magnet_link, "mag_msg_id": message.id, "prompt_msg_id": prompt.id}

# 🔥 补丁2：让所有的普通文字回复（比如回复配置信息）也只在对应的控制台里生效
@app.on_message(filters.chat(COMMAND_CENTER_CHAT) & filters.text & ~filters.media & filters.user("me"))
async def process_text_commands(client, message):
    text = message.text.strip().upper()
    if text.startswith("#") and GLOBAL_ROUTE_CACHE.get("expire_time", 0) > time.time():
        m_se = re.search(r'^#S(\d+)E(\d+)', text)
        m_s = re.search(r'^#S(\d+)$', text)
        m_e = re.search(r'^#E(\d+)$', text)
        reply_msg = ""
        if m_se:
            GLOBAL_ROUTE_CACHE["file_season"] = int(m_se.group(1)); GLOBAL_ROUTE_CACHE["manual_ep"] = int(m_se.group(2))
            reply_msg = f"🎛️ **重塑**: S{int(m_se.group(1)):02d}E{int(m_se.group(2)):02d}"
        elif m_s:
            GLOBAL_ROUTE_CACHE["file_season"] = int(m_s.group(1)); reply_msg = f"🎛️ **重塑**: S{int(m_s.group(1)):02d}"
        elif m_e:
            GLOBAL_ROUTE_CACHE["manual_ep"] = int(m_e.group(1)); reply_msg = f"🎛️ **重塑**: E{int(m_e.group(1)):02d}"
        if reply_msg: return await message.reply_text(reply_msg)

    chat_id = message.chat.id
    if chat_id in PENDING_MAGNETS and PENDING_MAGNETS[chat_id]:
        mag_data = None
        if message.reply_to_message and message.reply_to_message.id in PENDING_MAGNETS[chat_id]:
            mag_data = PENDING_MAGNETS[chat_id].pop(message.reply_to_message.id)
        elif not message.text.startswith("/"):
            first_key = list(PENDING_MAGNETS[chat_id].keys())[0]
            mag_data = PENDING_MAGNETS[chat_id].pop(first_key)
            
        if mag_data:
            if not PENDING_MAGNETS[chat_id]:
                del PENDING_MAGNETS[chat_id]
                
            config_text = message.text.strip()
            
            msgs_to_delete = [message.id, mag_data.get("prompt_msg_id")]
            if "mag_msg_id" in mag_data and mag_data["mag_msg_id"]:
                msgs_to_delete.append(mag_data["mag_msg_id"])
                
            async def delay_delete():
                await asyncio.sleep(10)
                try: await client.delete_messages(chat_id, [m for m in msgs_to_delete if m])
                except: pass
            asyncio.create_task(delay_delete())
            
            if "resume_gid" in mag_data:
                await handle_magnet_execution(client, message, magnet_link="", config_text=config_text, resume_gid=mag_data["resume_gid"])
            else:
                await handle_magnet_execution(client, message, mag_data["link"], config_text)
            return

@app.on_message(filters.video | filters.document)
@app.on_edited_message(filters.video | filters.document)
async def media_routing_gateway(client, message):
    try:
        config = load_listener_config()
        chat_id_to_check = str(message.chat.id) if message.chat else ""
        original_channel_id = ""
        parent_text = ""
        
        if message.forward_from_chat:
            original_channel_id = str(message.forward_from_chat.id)
        elif message.sender_chat:
            original_channel_id = str(message.sender_chat.id)

        if message.reply_to_message:
            parent = message.reply_to_message
            parent_text = parent.caption or parent.text or ""
            if not original_channel_id:
                if parent.forward_from_chat: original_channel_id = str(parent.forward_from_chat.id)
                elif parent.sender_chat: original_channel_id = str(parent.sender_chat.id) 
            if parent.reply_to_message: parent_text += f" {parent.reply_to_message.caption or parent.reply_to_message.text or ''}"

        matched_channel = None
        c_test = chat_id_to_check.replace("-100", "").strip("-") if chat_id_to_check else ""
        c_orig = original_channel_id.replace("-100", "").strip("-") if original_channel_id else ""
        
        for k in config.get("trusted_channels", {}):
            k_norm = str(k).replace("-100", "").strip("-")
            if c_test and k_norm == c_test: 
                matched_channel = k
                break
            if c_orig and k_norm == c_orig: 
                matched_channel = k
                break

        if matched_channel:
            channel_info = config["trusted_channels"][matched_channel]
            media = message.video or message.document
            text_to_scan = f"{message.caption or ''} {getattr(media, 'file_name', '')} {parent_text}"
            matched_routes = []
            for drama_key, route_info in channel_info.get("monitored_dramas", {}).items():
                if route_info.get("search_kw", drama_key).lower() in text_to_scan.lower(): matched_routes.append((drama_key, route_info))
            if not matched_routes: return
            
            best_match = None
            for key, route in matched_routes:
                if route.get("version", "") and route.get("version", "").lower() in text_to_scan.lower(): best_match = (key, route); break
            if not best_match:
                for key, route in matched_routes:
                    if not route.get("version", ""): best_match = (key, route); break
            if not best_match: return
            drama_key, route_info = best_match

            db_version, db_file_version = route_info.get("version", ""), route_info.get("file_version", "")
            search_kw = route_info.get("search_kw", drama_key)
            ep_num = extract_pure_episode(text_to_scan, drama_anchor=search_kw)
            if ep_num is not None and ep_num < route_info.get("min_ep", 1): return 
            
            min_mb, max_mb = route_info.get("min_mb", 0), route_info.get("max_mb", 999999)
            file_size_mb = media.file_size / (1024 * 1024) if getattr(media, "file_size", 0) else 0
            if not (min_mb <= file_size_mb <= max_mb): return 
            
            if db_file_version:
                combined_physical_tags = db_file_version.replace('.', '.')
                combined_history_tags = (db_file_version + "." + db_version).strip('.')
            else:
                auto_list = extract_media_tags(text_to_scan).split('.')
                res_list = [t for t in auto_list if t.lower() in ["2160p", "1080p", "720p", "4k", "8k"]]
                temp_tags = res_list + (db_version.split('.') if db_version else [])
                combined_physical_tags = ".".join(dict.fromkeys([t for t in temp_tags if t]))
                combined_history_tags = combined_physical_tags
            
            ver_tag_physical = f".{combined_physical_tags}" if combined_physical_tags else ""
            ver_tag_history = f".{combined_history_tags}" if combined_history_tags else ""
            
            folder_season = route_info.get("folder_season", route_info.get("file_season", 1))
            file_season = route_info.get("file_season", folder_season)
            year = route_info.get("year")
            if not year: year, _ = await fetch_tmdb_details(search_kw)
            
            pure_drama_name = drama_key[:-len(f"_{db_version}")] if db_version and drama_key.endswith(f"_{db_version}") else drama_key
            folder_name = f"{pure_drama_name} ({year})" if year else pure_drama_name
            if db_version: folder_name = f"{folder_name} {db_version}"  
            
            clean_base_for_check = pure_drama_name.replace(" ", ".")
            if check_history(clean_base_for_check, file_season, ep_num, combined_history_tags): return
            
            # 🔥 加入 enable_139
            enable_139 = route_info.get("enable_139", False)
            override_info = (folder_name, route_info["category"], folder_season, file_season, year, ep_num, db_version, db_file_version, matched_channel, drama_key, route_info.get("end_ep", 9999), enable_139)
            try: status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[实时发车]**\n📺 `{search_kw}` ➔ S{file_season:02d}E{ep_num:02d}")
            except: status = await client.send_message("me", f"🎯 **[备用发车]**\n📺 `{search_kw}` ➔ S{file_season:02d}E{ep_num:02d}")
            asyncio.create_task(process_media_transfer(client, message, status, override_info))
            return

        if message.chat and message.chat.type in [enums.ChatType.PRIVATE, enums.ChatType.BOT]:
            if time.time() > GLOBAL_ROUTE_CACHE["expire_time"] or not GLOBAL_ROUTE_CACHE["folder_name"]: return
            try: status = await message.reply_text("⚡ 转发航线认证通过，正向引流拉取...")
            except: return
            asyncio.create_task(process_media_transfer(client, message, status))
    except: pass

# =================================================================
# 🚀 TG 自动下载后台任务处理
# =================================================================
async def process_media_transfer(client, message, status, override_info=None):
    media = message.video or message.document
    if not media: return
    raw_file = getattr(media, "file_name", f"TG_{message.id}.mp4")
    _, ext = os.path.splitext(raw_file)
    if not ext: ext = ".mp4"
    
    src_chat_id = src_drama_key = src_end_ep = None
    if override_info:
        if len(override_info) >= 12: 
            folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix, src_chat_id, src_drama_key, src_end_ep, enable_139 = override_info[:12]
        elif len(override_info) >= 11:
            folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix, src_chat_id, src_drama_key, src_end_ep = override_info[:11]
            enable_139 = False
        elif len(override_info) >= 8: 
            folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix = override_info[:8]
            enable_139 = False
        else:
            folder, cat, folder_season, file_season, year, ep_num = override_info[:6]
            version_suffix = file_suffix = ""
            enable_139 = False
    else:
        folder, cat = GLOBAL_ROUTE_CACHE["folder_name"], GLOBAL_ROUTE_CACHE["category"]
        folder_season, file_season = GLOBAL_ROUTE_CACHE["folder_season"], GLOBAL_ROUTE_CACHE["file_season"]
        year = GLOBAL_ROUTE_CACHE.get("year", "")
        version_suffix = GLOBAL_ROUTE_CACHE.get("version", "")
        file_suffix = GLOBAL_ROUTE_CACHE.get("file_version", "") 
        enable_139 = GLOBAL_ROUTE_CACHE.get("enable_139", False)
        if GLOBAL_ROUTE_CACHE["manual_ep"] is not None:
            ep_num = GLOBAL_ROUTE_CACHE["manual_ep"]
            GLOBAL_ROUTE_CACHE["manual_ep"] += 1
        else: ep_num = extract_pure_episode(f"{message.caption or ''} {raw_file}")
            
    text_to_scan = f"{message.caption or ''} {raw_file}"
    auto_tags = extract_media_tags(text_to_scan)
    is_movie = "电影" in cat or cat in ["演唱会", "纪录片"]
    movie_part_tag = extract_movie_part(text_to_scan) if is_movie else ""
    
    clean_base = re.sub(r'\s*\(\d{4}\)', '', folder).strip()
    if version_suffix and clean_base.endswith(version_suffix): clean_base = clean_base[:-len(version_suffix)].strip()
    clean_base = clean_base.replace(" ", ".")
    
    auto_list = auto_tags.split('.') if auto_tags else []
    f_list = file_suffix.split('.') if file_suffix else []
    res_list = [t for t in auto_list if t.lower() in ["2160p", "1080p", "720p", "4k", "8k"]]
    other_auto = [t for t in auto_list if t not in res_list]

    raw_physical_tags = []
    if movie_part_tag: raw_physical_tags.append(movie_part_tag) 
    raw_physical_tags.extend(res_list); raw_physical_tags.extend(f_list); raw_physical_tags.extend(other_auto) 
    final_physical_tags = []
    seen_physical = set()
    for t in raw_physical_tags:
        if t and t.lower() not in seen_physical: final_physical_tags.append(t); seen_physical.add(t.lower())
    combined_physical_tags = ".".join(final_physical_tags)
    ver_tag_physical = f".{combined_physical_tags}" if combined_physical_tags else ""
    
    raw_history_tags = []
    if movie_part_tag: raw_history_tags.append(movie_part_tag)
    raw_history_tags.extend(res_list); raw_history_tags.extend(f_list)   
    if version_suffix: raw_history_tags.extend(version_suffix.split('.'))
    final_history_tags = []
    seen_history = set()
    for t in raw_history_tags:
        if t and t.lower() not in seen_history: final_history_tags.append(t); seen_history.add(t.lower())
    combined_history_tags = ".".join(final_history_tags)
    ver_tag_history = f".{combined_history_tags}" if combined_history_tags else ""
    
    current_mount = get_mount_root()
    if is_movie:
        standard_name = f"{clean_base}.{year}{ver_tag_physical}{ext}" if year else f"{clean_base}{ver_tag_physical}{ext}"
        target_dir_189 = f"{current_mount}/{cat}/{folder}"
    else:
        if ep_num is None:
            await notify_steward_log(f"⚠️ [过滤跳过] `{clean_base}` 未提取到集数(疑似预告)，已过滤。")
            try: await status.edit_text(f"⚠️ `{clean_base}` 未能提取到集数，已被防垃圾机制拦截！")
            except: pass
            return
        if check_history(clean_base, file_season, ep_num, combined_history_tags):
            hist_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag_history}"
            await notify_steward_log(f"🛡️ [账本拦截] `{hist_key}` 账本中已存在，自动销毁重复发车！")
            try: await status.delete() 
            except: pass
            return
        ep_str = f"{ep_num:02d}"
        target_dir_189 = f"{current_mount}/{cat}/{folder}/Season {folder_season}"
        standard_name = f"{clean_base}.S{file_season:02d}E{ep_str}.{year}{ver_tag_physical}{ext}" if year else f"{clean_base}.S{file_season:02d}E{ep_str}{ver_tag_physical}{ext}"
        
    task_lock_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag_history}" if not is_movie else f"{clean_base}{ver_tag_history}"
    if task_lock_key in GLOBAL_ACTIVE_LOCKS:
        try: await status.delete()
        except: pass
        return
    
    GLOBAL_ACTIVE_LOCKS.add(task_lock_key)
    bg_task_spawned = False
    downloaded_bytes = 0

    try:
        local_temp_name = f"[{version_suffix}]_{standard_name}" if version_suffix else standard_name
        local_path = os.path.join(LOCAL_TEMP_DIR, local_temp_name)
        total_bytes = media.file_size
        file_size_mb = total_bytes / (1024 * 1024) if total_bytes else 0
        source_name = message.forward_from_chat.title if message.forward_from_chat and message.forward_from_chat.title else (message.chat.title if message.chat and message.chat.title else "未知来源")
            
        try: await status.edit_text(f"⏳ **[TG下载排队中]** `{standard_name}`\n🚦 正在等待下载通道空闲...")
        except: pass

        async with GLOBAL_TRANSFER_LOCK:
            await notify_steward_log(f"📥 [TG拉取启动] 来源: {source_name} | 大小: {file_size_mb:.2f} MB")
            chunk_size = 1024 * 1024
            retry_count = 0
            max_retries = 999999  
            last_stuck_chunks = -1
            stuck_loop_count = 0
            last_ui_time = time.time()
            last_ui_bytes = 0
            speed_text, eta_text = "0.00 MB/s", "计算中..."
            
            notified_50_dl = False
            notified_98_dl = False
            
            while True:
                if retry_count >= max_retries: break
                current_bytes = os.path.getsize(local_path) if os.path.exists(local_path) else 0
                if current_bytes > total_bytes:
                    try: os.remove(local_path)
                    except: pass
                    current_bytes = 0
                if current_bytes == total_bytes:
                    downloaded_bytes = current_bytes; break
                    
                completed_chunks = current_bytes // chunk_size
                secure_bytes = completed_chunks * chunk_size
                if secure_bytes > 0:
                    try:
                        with open(local_path, "r+b") as f: f.truncate(secure_bytes)
                    except: pass
                    downloaded_bytes = secure_bytes
                    open_mode = "ab"
                else: 
                    downloaded_bytes, completed_chunks = 0, 0
                    open_mode = "wb"
                    
                try:
                    buffer_queue = asyncio.Queue(maxsize=50)
                    writer_error = None
                    async def disk_writer():
                        nonlocal writer_error
                        try:
                            with open(local_path, open_mode) as f:
                                while True:
                                    chunk = await buffer_queue.get()
                                    if chunk is None: break 
                                    f.write(chunk); buffer_queue.task_done()
                        except Exception as e: writer_error = e

                    writer_task = asyncio.create_task(disk_writer())
                    yielded_any = False
                    
                    async for chunk in client.stream_media(message, offset=completed_chunks):
                        yielded_any = True
                        if task_lock_key in GLOBAL_CANCEL_TASKS or "ALL" in GLOBAL_CANCEL_TASKS: break 
                        if writer_error: raise writer_error 
                        await buffer_queue.put(chunk)
                        downloaded_bytes += len(chunk)
                        
                        if total_bytes > 0:
                            pct = int(downloaded_bytes * 100 / total_bytes)
                            
                            if pct >= 50 and not notified_50_dl:
                                asyncio.create_task(notify_steward_log(f"📥 [TG拉取] {standard_name} 进度达 50%"))
                                notified_50_dl = True
                            if pct >= 98 and not notified_98_dl:
                                asyncio.create_task(notify_steward_log(f"📥 [TG拉取] {standard_name} 进度达 98%"))
                                notified_98_dl = True

                        now = time.time()
                        duration = now - last_ui_time
                        if duration >= 8.0:
                            bytes_diff = downloaded_bytes - last_ui_bytes
                            speed_mb = (bytes_diff / duration) / (1024 * 1024)
                            speed_text = f"{speed_mb:.2f} MB/s"
                            rem_bytes = total_bytes - downloaded_bytes
                            speed_bps = bytes_diff / duration
                            if speed_bps > 0:
                                eta_sec = rem_bytes / speed_bps
                                eta_text = f"{int(eta_sec // 60)}分{int(eta_sec % 60)}秒" if eta_sec > 60 else f"{int(eta_sec)}秒"
                            else: eta_text = "卡顿"
                            pct = int(downloaded_bytes * 100 / total_bytes)
                            last_ui_time, last_ui_bytes = now, downloaded_bytes
                            try:
                                asyncio.create_task(status.edit_text(f"🚀 **[极速拉取]** `{standard_name}`\n📡 来源: **{source_name}** | ⚖️ 大小: **{file_size_mb:.2f} MB**\n📈 实时网速: **{speed_text}**\n⏳ 预计剩余: **{eta_text}**\n⚡ 涡轮进度: **{pct}%**"))
                            except: pass
                    
                    await buffer_queue.put(None)
                    await writer_task

                    if task_lock_key in GLOBAL_CANCEL_TASKS or "ALL" in GLOBAL_CANCEL_TASKS: break 
                    if downloaded_bytes >= total_bytes: 
                        try: await message.delete()
                        except: pass
                        break
                    else:
                        retry_count += 1
                        if not yielded_any:
                            try: message = await client.get_messages(message.chat.id, message.id)
                            except: pass
                        if completed_chunks == last_stuck_chunks:
                            stuck_loop_count += 1
                            try: await status.edit_text(f"⚠️ **[节点堵塞]** 断点死磕中 ({retry_count})...")
                            except: pass
                        else: last_stuck_chunks, stuck_loop_count = completed_chunks, 0
                        await asyncio.sleep(3.0)
                        
                except Exception as e:
                    if 'writer_task' in locals() and not writer_task.done(): writer_task.cancel()
                    retry_count += 1
                    if retry_count % 5 == 0:
                        try: message = await client.get_messages(message.chat.id, message.id)
                        except: pass
                    await asyncio.sleep(3.0)
                    
            if downloaded_bytes < total_bytes: 
                try:
                    if os.path.exists(local_path): os.remove(local_path)
                    await status.edit_text("❌ 任务已被彻底终止或残片销毁。")
                except: pass
                return

        try: await status.edit_text("✅ 本地落盘完成，进入云端上传排队队列...")
        except: pass

        if ext.lower() in [".cas", ".zip"]:
            import zipfile
            STAGING_DIR = "/data/data/com.termux/files/home/sharecas"
            os.makedirs(STAGING_DIR, exist_ok=True)
            final_cas_path = os.path.join(STAGING_DIR, standard_name.replace(ext, ".cas"))
            try:
                if ext.lower() == ".zip":
                    with zipfile.ZipFile(local_path, "r") as z:
                        cas_files = [f for f in z.namelist() if f.lower().endswith(".cas")]
                        if cas_files:
                            with open(final_cas_path, "wb") as f_out: f_out.write(z.read(cas_files[0]))
                else: shutil.move(local_path, final_cas_path)
                try: os.remove(local_path) 
                except: pass
                try: await status.edit_text(f"🎉 **CAS截留成功**\n已洗名并存入本地 `{STAGING_DIR}`，等待接管。")
                except: pass
                return
            except Exception as e: return
        
        asyncio.create_task(bg_upload_retry_task(
            local_path=local_path, 
            target_dir_189=target_dir_189, 
            cat=cat,
            folder_name=folder,
            folder_season=folder_season,
            total_bytes=total_bytes, 
            clean_base=clean_base, 
            file_season=file_season, 
            ep_num=ep_num, 
            is_movie=is_movie, 
            standard_name=standard_name, 
            history_tags=combined_history_tags,
            status_msg=status,
            enable_139=enable_139
        ))
        bg_task_spawned = True
        
        if src_chat_id and src_drama_key and src_end_ep and ep_num >= src_end_ep:
            config_end = load_listener_config()
            if src_chat_id in config_end.get("trusted_channels", {}) and src_drama_key in config_end["trusted_channels"][src_chat_id].get("monitored_dramas", {}):
                del config_end["trusted_channels"][src_chat_id]["monitored_dramas"][src_drama_key]
                try:
                    with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: 
                        json.dump(config_end, f, ensure_ascii=False, indent=4)
                except: pass
                try: await client.send_message(COMMAND_CENTER_CHAT, f"🎉 **[圆满杀青]** `{src_drama_key}` 已达 {src_end_ep} 集完结线，自动解除监控！")
                except: pass
        
    finally:
        if not bg_task_spawned: GLOBAL_ACTIVE_LOCKS.discard(task_lock_key)
        GLOBAL_CANCEL_TASKS.discard(task_lock_key)

async def sweep_existing_history(client, chat_id, drama_key, category, folder_season, file_season, min_ep, min_mb=0, max_mb=999999, fetch_limit=200, year=None):
    try:
        chat_id_int = int(chat_id) if str(chat_id).lstrip('-').isdigit() else chat_id
        config = load_listener_config()
        d_info = config.get("trusted_channels", {}).get(str(chat_id), {}).get("monitored_dramas", {}).get(drama_key, {})
        search_kw = d_info.get("search_kw", drama_key)
        db_version, db_file_version, end_ep = d_info.get("version", ""), d_info.get("file_version", ""), d_info.get("end_ep", 9999)
        enable_139 = d_info.get("enable_139", False)
        
        if not year: year, _ = await fetch_tmdb_details(search_kw)
        
        global GLOBAL_STOP_SWEEP
        async for old_msg in client.get_chat_history(chat_id_int, limit=fetch_limit):
            if GLOBAL_STOP_SWEEP: break
            media = old_msg.video or old_msg.document
            if not media: continue
            text_to_scan = f"{old_msg.caption or ''} {getattr(media, 'file_name', '')}"
            if old_msg.reply_to_message: text_to_scan += f" {old_msg.reply_to_message.caption or old_msg.reply_to_message.text or ''}"
            
            if search_kw.lower() in text_to_scan.lower():
                skip_due_to_exclusive = False
                if not db_version:
                    for other_key, other_info in config.get("trusted_channels", {}).get(str(chat_id), {}).get("monitored_dramas", {}).items():
                        if other_info.get("version") and other_info["version"].lower() in text_to_scan.lower() and other_info.get("search_kw", "").lower() in text_to_scan.lower():
                            skip_due_to_exclusive = True; break
                if skip_due_to_exclusive: continue
                
                ep_num = extract_pure_episode(text_to_scan, drama_anchor=search_kw)
                if ep_num is None or ep_num < min_ep: continue
                file_size_mb = media.file_size / (1024 * 1024) if getattr(media, "file_size", 0) else 0
                if not (min_mb <= file_size_mb <= max_mb): continue
                
                auto_list = extract_media_tags(text_to_scan).split('.')
                res_list = [t for t in auto_list if t.lower() in ["2160p", "1080p", "720p", "4k", "8k"]]
                temp_tags = res_list + (db_file_version.split('.') if db_file_version else []) + (db_version.split('.') if db_version else [])
                preview_tags = ".".join(dict.fromkeys(temp_tags))
                
                pure_drama_name = drama_key[:-len(f"_{db_version}")] if db_version and drama_key.endswith(f"_{db_version}") else drama_key
                folder_name = f"{pure_drama_name} ({year})" if year else pure_drama_name
                if db_version: folder_name = f"{folder_name} {db_version}" 
                
                clean_base_for_check = pure_drama_name.replace(" ", ".")
                if check_history(clean_base_for_check, file_season, ep_num, preview_tags): continue
                
                override_info = (folder_name, category, folder_season, file_season, year, ep_num, db_version, db_file_version, str(chat_id), drama_key, end_ep, enable_139)
                try: status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[哨发发车]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
                except: status = await client.send_message("me", f"🎯 **[备用嗅探]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
                asyncio.create_task(process_media_transfer(client, old_msg, status, override_info))
                await asyncio.sleep(5) 
    except Exception as e: print(f"⚠️ [扫荡崩溃]: {e}")

async def smart_patrol_daemon(client):
    await asyncio.sleep(60) 
    while True:
        try:
            current_hour = datetime.now().hour
            if 0 <= current_hour < 9: await asyncio.sleep(3600); continue
            await asyncio.sleep(900)
            today_cn = {0: "周一", 1: "周二", 2: "周三", 3: "周四", 4: "周五", 5: "周六", 6: "周日"}[datetime.now().weekday()]
            config = load_listener_config()
            for chat_id, info in config.get("trusted_channels", {}).items():
                for drama_name, d_info in info.get("monitored_dramas", {}).items():
                    if d_info.get("frequency", "日更") not in ["日更", today_cn]: continue
                    f_s = d_info.get("file_season", d_info.get("folder_season", 1))
                    fo_s = d_info.get("folder_season", f_s)
                    asyncio.create_task(sweep_existing_history(client, chat_id, drama_name, d_info.get("category", "未分类"), fo_s, f_s, d_info.get("min_ep", 1), d_info.get("min_mb", 0), d_info.get("max_mb", 999999), 10, d_info.get("year")))
        except: await asyncio.sleep(60)

async def startup_catchup_sweep(client):
    await asyncio.sleep(5)
    try:
        config = load_listener_config()
        channels = config.get("trusted_channels", {})
        if not channels: return
        
        await notify_steward_log("🔍 [开机扫荡] 启动开机自检，正在扫描漏网之鱼...")
        try: await client.send_message(COMMAND_CENTER_CHAT, "🔍 **[系统自检]** 机器人重启，正在自动扫荡频道漏网之鱼...")
        except: pass

        for chat_id, info in channels.items():
            for drama_name, d_info in info.get("monitored_dramas", {}).items():
                f_s = d_info.get("file_season", d_info.get("folder_season", 1))
                fo_s = d_info.get("folder_season", f_s)
                await sweep_existing_history(client, chat_id, drama_name, d_info.get("category", "未分类"), fo_s, f_s, d_info.get("min_ep", 1), d_info.get("min_mb", 0), d_info.get("max_mb", 999999), 200, d_info.get("year"))
                await asyncio.sleep(5)
                
        await notify_steward_log("✅ [开机扫荡] 所有频道历史自检完成！")
    except: pass

# =================================================================
# 🚀 磁力任务断点接管协议 (开机激活)
# =================================================================
async def resume_magnet_tasks(client):
    tasks = load_magnet_tasks()
    if not tasks: return
    
    await notify_steward_log(f"🔄 [记忆恢复] 发现 {len(tasks)} 个未完成的磁力历史任务，正在尝试重新接管...")
    for gid, info in list(tasks.items()):
        try:
            async with httpx.AsyncClient(timeout=5.0) as h:
                res = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellStatus", "params":[f"token:{ARIA2_RPC_SECRET}", gid]})
                res_json = res.json()
                if "error" in res_json:
                    continue
                
                GLOBAL_TRACKED_GIDS.add(gid)
                asyncio.create_task(handle_magnet_execution(client, None, info["magnet_link"], info["config_text"], resume_gid=gid))
                await asyncio.sleep(2)
        except Exception: pass

# =================================================================
# 🛸 野生任务捕获雷达 (自动逆向监控 Aria2)
# =================================================================
async def orphan_magnet_scanner(client):
    await asyncio.sleep(20) 
    while True:
        try:
            async with httpx.AsyncClient(timeout=10.0) as h:
                res_a = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellActive", "params":[f"token:{ARIA2_RPC_SECRET}"]})
                res_w = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellWaiting", "params":[f"token:{ARIA2_RPC_SECRET}", 0, 100]})
                
                tasks_a = res_a.json().get("result", []) if res_a.status_code == 200 else []
                tasks_w = res_w.json().get("result", []) if res_w.status_code == 200 else []
                all_tasks = tasks_a + tasks_w
                
                for t in all_tasks:
                    gid = t.get("gid")
                    belongs_to = t.get("belongsTo")
                    
                    is_bt = "bittorrent" in t or "infoHash" in t
                    if not is_bt: continue
                    
                    if gid in GLOBAL_TRACKED_GIDS or (belongs_to and belongs_to in GLOBAL_TRACKED_GIDS):
                        if gid not in GLOBAL_TRACKED_GIDS: 
                            GLOBAL_TRACKED_GIDS.add(gid)
                        continue
                        
                    if gid in ORPHAN_PROMPTED_GIDS: 
                        continue
                        
                    ORPHAN_PROMPTED_GIDS.add(gid)
                    
                    name = "未知磁力任务"
                    if "bittorrent" in t and "info" in t["bittorrent"]:
                        name = t["bittorrent"]["info"].get("name", name)
                    else:
                        name = f"种子元数据 (GID: {gid[:8]}...)"
                        
                    try:
                        prompt_msg = await client.send_message(
                            COMMAND_CENTER_CHAT, 
                            f"👽 **[野生任务捕获]** 发现外部添加的未监控下载任务：\n📄 `{name}`\n\n请**回复本条消息**（或直接在下方输入）提供归属信息以接管：\n👉 **格式**: `剧名|别名 分类 [S季数] [年份] [v=版本] [c139]`"
                        )
                        chat_id = prompt_msg.chat.id
                        if chat_id not in PENDING_MAGNETS:
                            PENDING_MAGNETS[chat_id] = {}
                        PENDING_MAGNETS[chat_id][prompt_msg.id] = {"resume_gid": gid, "prompt_msg_id": prompt_msg.id}
                    except: pass
        except Exception: pass
        await asyncio.sleep(60)

if __name__ == "__main__":
    clean_orphan_temp_files(max_age_hours=24) 
    load_listener_config()
    async def start_system():
        await app.start()
        
        # 🔥 新增：开机强制同步通讯录，彻底解决 Peer id invalid 报错
        try:
            await notify_steward_log("🔄 正在同步云端通讯录至本地数据库")
            async for _ in app.get_dialogs(limit=200):
                pass
        except Exception:
            pass
            
        await notify_steward_log("✅ 机器人重启完毕，双下载与上传引擎已全线上线！")
        asyncio.create_task(startup_catchup_sweep(app))
        asyncio.create_task(smart_patrol_daemon(app))
        asyncio.create_task(resume_magnet_tasks(app)) 
        asyncio.create_task(orphan_magnet_scanner(app)) 
        try: await app.send_message(COMMAND_CENTER_CHAT, "🤖 下载与上传推引擎已启动！")
        except: pass
        await idle()
        await app.stop()
    app.run(start_system())
```