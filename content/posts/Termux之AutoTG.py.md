---

title: "Termux之AutoTG.py"

author: "xxsky"

type: "posts"

date: 2026-05-12T19:31:24+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

咱们利用 Python 的 Pyrogram 库作为接头人伪装成 TG 客户端（或 Bot），直接从 TG 服务器实时拉取视频流数据块（Chunk），拿到一块就立刻通过异步管道通过 PUT 请求直接灌入 Openlist 的上传接口 (/api/fs/put)。

<!--more-->
### 一、安装环境
在 Termux 里安装所需的现代异步网络库：
```
pip install pyrogram tgcrypto httpx
```

### 二、Telegram 的 API_ID 和 API_HASH

手把手拿下 Telegram 的 API_ID 和 API_HASH

Telegram 官方要求任何第三方自动化脚本（无论是普通客户端还是 Bot）必须绑定一组开发者身份凭证。申请过程完全免费，但必须走官方开发者后台。

📝 极简申请步骤

1.准备高稳定全局网络：打开你的代理软件（务必开启全局路由模式，避开频繁抖动的免费 CDN 节点，确保访问顺畅）。

2.进入官方后台：浏览器直接打开开发者门户网址 👉 [my.telegram.org](https://my.telegram.org/auth)

3.安全登录：

* 在输入框填入你绑定的 Telegram 手机号（务必带上国际区号，例如中国大陆号码填 +86138xxxx）。

* 点击 Next 后，千万别等短信，直接打开你的 Telegram 客户端，官方服务号（Service Notifications）会实时发给你一串长文本验证码。复制粘贴回网页完成登录。

4.进入开发配置：登录成功后，点击页面核心菜单里的 API development tools。

5.填写应用表单（首次进入需创建应用）：

* App title：随便起个名字（比如 AutoLeecherPro）。

* Short name：必须全网唯一，且只能用纯英文字母和数字（比如 mytermuxleecher2026）。

* URL：直接留空不管。

* Platform：下拉选择 Desktop 或 Android 均可。

* Description：留空，直接点击底部的 Create application 提交。

6.提取终极密钥：

* 提交成功后，页面当场刷新，显示出你的 App api_id（一串纯数字）和 App api_hash（一串极长的混合字符）。

* ⚠️ 铁律：直接把这两个值复制贴进咱们脚本的配置项里。绝对绝不泄露给任何人，这是你账号操控权的底层凭证。

### 三、autotg.py

1.脚本：
```
import os
import re
import time
import json
import asyncio
from datetime import datetime
from urllib.parse import quote
import httpx
from pyrogram import Client, filters, enums, idle
import logging
import shutil

# =================================================================
# ⚙️ 核心网关、路径与凭证配置区域
# =================================================================
API_ID = 33349348              
API_HASH = "44bde7f01d2b6001589c28cea93716af"        

COMMAND_CENTER_CHAT = "@xxskyemby_bot"

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

# 🚦 全局大盘与锁 (TG与磁力彻底隔离)
GLOBAL_ROUTE_CACHE = {"folder_name": "", "category": "", "folder_season": 1, "file_season": 1, "year": "", "version": "", "expire_time": 0, "manual_ep": None}
GLOBAL_ACTIVE_LOCKS = set() 
GLOBAL_CANCEL_TASKS = set() 

# TG专属锁
GLOBAL_TRANSFER_LOCK = asyncio.Semaphore(1) 
GLOBAL_UPLOAD_LOCK = asyncio.Semaphore(1)   

# 磁力专属锁，不干涉TG
MAGNET_UPLOAD_LOCK = asyncio.Semaphore(1)

PENDING_MAGNETS = {} 
GLOBAL_STOP_SWEEP = False

async def notify_steward_log(msg, level="INFO"):
    logger.info(f"[{level}] {msg}")
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            await client.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": level, "msg": msg})
    except Exception: pass

# =================================================================
# 📂 磁力任务本地持久化记忆函数
# =================================================================
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

# 🔥 核心：智能洗名函数 (防冗余叠加优化)
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

# =================================================================
# 🔥 全新独立重构的清道夫函数（无干扰静默拔除版，强化种子/碎片狙杀）
# =================================================================
async def wipe_magnet_task(gids_to_wipe):
    if isinstance(gids_to_wipe, str):
        gids_to_wipe = [gids_to_wipe]
        
    target_gids = set(gids_to_wipe)
    paths_to_delete = set()
    info_hashes = set()  # 🔥 抓取任务特有的 infoHash 标识
    
    try:
        async with httpx.AsyncClient(timeout=10.0) as h:
            for gid in list(target_gids):
                try:
                    res = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellStatus", "params":[f"token:{ARIA2_RPC_SECRET}", gid]})
                    st = res.json().get("result", {})
                    if "followedBy" in st and st["followedBy"]:
                        target_gids.update(st["followedBy"])
                    if "belongsTo" in st and st["belongsTo"]:
                        target_gids.add(st["belongsTo"])
                except: pass
                
            for g in list(target_gids):
                try:
                    res_g = await h.post(ARIA2_RPC_URL, json={"jsonrpc":"2.0", "id":"tg_bot", "method":"aria2.tellStatus", "params":[f"token:{ARIA2_RPC_SECRET}", g]})
                    st_data = res_g.json().get("result", {})
                    
                    # 🔥 收集所有任务的 infoHash
                    if "infoHash" in st_data and st_data["infoHash"]:
                        info_hashes.add(st_data["infoHash"])
                        
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

    # 🔥 强制狙杀底层可能遗留的 .torrent 种子文件（针对 cancel 时独有情况）
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
            
        # 🔥 强制狙杀中断时可能遗留的 .aria2 碎片缓存文件
        aria2_path = f"{f_path}.aria2"
        if os.path.exists(aria2_path) and os.path.isfile(aria2_path):
            try: os.remove(aria2_path)
            except: pass

    for top_dir in top_dirs_to_delete:
        try:
            if os.path.exists(top_dir):
                shutil.rmtree(top_dir, ignore_errors=True)
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
# =================================================================

async def bg_fetch_cas_task(cas_target_full, final_cas_path, sub_path, cas_file_name, status_msg=None, original_gid=None):
    try:
        await asyncio.sleep(10)
        get_info_url = f"{OLIST_URL}/api/fs/get"
        headers_get = {"Authorization": OLIST_TOKEN, "Content-Type": "application/json"}
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
                            await notify_steward_log(f"🔗 [CAS下发成功] `{sub_path}/{cas_file_name}`")
                            
                            # 🔥 沿用同一个消息更新 CAS 状态，不刷屏
                            if status_msg:
                                try: await status_msg.edit_text(f"🔗 **[CAS就绪]**\n📥 `{cas_file_name}`\n✅ 镜像完毕，等待接管！")
                                except: pass
                            else:
                                try: await app.send_message(COMMAND_CENTER_CHAT, f"🔗 **[CAS就绪]**\n📥 `{cas_file_name}`\n✅ 镜像完毕，等待接管！")
                                except: pass
                            cas_downloaded = True
                            break
            except Exception: pass 
            if not cas_downloaded: await asyncio.sleep(10)

        if not cas_downloaded:
            if status_msg:
                try: await status_msg.edit_text(f"⚠️ **[CAS拉取失败]**\n❌ `{cas_file_name}`\n云端未生成或直链被拦截。")
                except: pass
            else:
                try: await app.send_message(COMMAND_CENTER_CHAT, f"⚠️ **[CAS拉取失败]**\n❌ `{cas_file_name}`\n云端未生成或直链被拦截。")
                except: pass
    finally:
        # 🔥 CAS 落盘彻底完成后，安全擦除该任务的本地记忆档案
        if original_gid:
            remove_magnet_task(original_gid)

# =================================================================
# 🚀 【磁力专属】后台直推任务 (独立通道)
# =================================================================
# 🔥 修改：支持传入已有的 status_msg 进行一镜到底复用更新
async def magnet_upload_task(local_path, target_full, total_bytes, clean_base, file_season, ep_num, is_movie, standard_name, version_suffix="", status_msg=None, original_gid=None):
    up_msg = status_msg
    cas_launched = False
    try:
        if not up_msg:
            try: up_msg = await app.send_message(COMMAND_CENTER_CHAT, f"☁️ **[磁力-天翼云直推]**\n📄 `{standard_name}`\n正在建立上传通道...")
            except: pass
        else:
            try: await up_msg.edit_text(f"☁️ **[磁力-天翼云直推]**\n📄 `{standard_name}`\n正在建立上传通道...")
            except: pass

        await asyncio.sleep(2)  
        asyncio.create_task(notify_steward_log(f"🧲 [磁力直推开始] {standard_name}"))
        
        while os.path.exists(local_path):
            try:
                put_url = f"{OLIST_URL}/api/fs/put"
                headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                
                # 磁力专属并发锁，绝对不卡 TG
                async with MAGNET_UPLOAD_LOCK:
                    if up_msg:
                        try: await up_msg.edit_text(f"🚀 **[磁力直推]** `{standard_name}`\n上传进度: **0%**")
                        except: pass
                        
                    last_up = 0
                    with open(local_path, "rb") as f_upload:
                        async def file_iter():
                            nonlocal last_up
                            sent = 0
                            while True:
                                chunk = f_upload.read(1024 * 1024)
                                if not chunk: break
                                sent += len(chunk)
                                pct = int(sent * 100 / total_bytes)
                                
                                if (pct // 10) > (last_up // 10):
                                    last_up = pct - (pct % 10)
                                    if up_msg:
                                        try: await up_msg.edit_text(f"🚀 **[磁力直推]** `{standard_name}`\n上传进度: **{last_up}%**")
                                        except: pass
                                    if last_up in [50, 100]:
                                        asyncio.create_task(notify_steward_log(f"🧲 [磁力直推] {standard_name} ➔ {last_up}%"))
                                yield chunk
                                
                        async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=1800.0, write=60.0, pool=None), trust_env=False) as h: 
                            resp = await h.put(put_url, content=file_iter(), headers=headers)
                    
                    if resp.json().get("code") == 200:
                        try: os.remove(local_path)
                        except: pass
                        
                        ver_tag = f".{version_suffix}" if version_suffix else ""
                        if not is_movie and ep_num is not None:
                            record_history(clean_base, file_season, ep_num, version_suffix)
                            await notify_steward_log(f"📝 [云端入账] {clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}")
                        
                        success_msg = f"🎉 **[磁力落盘成功]** ➔ `{standard_name}`\n✅ 空间已释放！"
                        if up_msg:
                            try: await up_msg.edit_text(success_msg)
                            except: pass
                        else:
                            try: await app.send_message(COMMAND_CENTER_CHAT, success_msg)
                            except: pass
                        
                        try:
                            target_dir = os.path.dirname(target_full)
                            current_mount = get_mount_root()
                            STAGING_BASE_DIR = "/storage/emulated/0/Download/189cas"
                            cas_file_name = f"{standard_name}.cas"
                            cas_target_full = f"{target_full}.cas"
                            sub_path = target_dir.replace(current_mount, "", 1)
                            local_cas_dir = f"{STAGING_BASE_DIR}{sub_path}"
                            os.makedirs(local_cas_dir, exist_ok=True)
                            final_cas_path = os.path.join(local_cas_dir, cas_file_name)
                            
                            cas_launched = True
                            asyncio.create_task(bg_fetch_cas_task(cas_target_full, final_cas_path, sub_path, cas_file_name, status_msg=up_msg, original_gid=original_gid))
                        except Exception as e:
                            await notify_steward_log(f"⚠️ [CAS派发失败]: {e}", level="WARNING")
                        break
            except Exception as e:
                await notify_steward_log(f"⚠️ [磁力上传重试] {standard_name} 遇阻: {str(e)}", level="WARNING")
            
            await asyncio.sleep(60)
    finally:
        # 如果因为某些异常导致未能顺利接力给 CAS，就保底删库防止内存垃圾
        if original_gid and not cas_launched:
            remove_magnet_task(original_gid)

# =================================================================
# 🚀 【TG专属】后台直推任务
# =================================================================
async def bg_upload_retry_task(local_path, target_full, total_bytes, clean_base, file_season, ep_num, is_movie, standard_name, history_tags="", status_msg=None):
    try:
        await asyncio.sleep(10)  
        
        notified_50_up = False
        notified_98_up = False
        
        while os.path.exists(local_path):
            try:
                put_url = f"{OLIST_URL}/api/fs/put"
                headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                
                async with GLOBAL_UPLOAD_LOCK:
                    last_up = 0
                    with open(local_path, "rb") as f_upload:
                        async def file_iter():
                            nonlocal last_up, notified_50_up, notified_98_up
                            sent = 0
                            while True:
                                chunk = f_upload.read(1024 * 1024)
                                if not chunk: break
                                sent += len(chunk)
                                pct = int(sent * 100 / total_bytes)
                                
                                # 🔥 TG上传 10% 动态进度与卡片更新
                                if (pct // 10) > (last_up // 10):
                                    last_up = pct - (pct % 10)
                                    if status_msg:
                                        try: await status_msg.edit_text(f"🚀 **[直推天翼云]** `{standard_name}`\n上传进度: **{last_up}%**")
                                        except: pass
                                
                                # 🔥 TG上传 50% 与 98% 端口日志推送
                                if pct >= 50 and not notified_50_up:
                                    asyncio.create_task(notify_steward_log(f"☁️ [TG云推] {standard_name} 进度达 50%"))
                                    notified_50_up = True
                                if pct >= 98 and not notified_98_up:
                                    asyncio.create_task(notify_steward_log(f"☁️ [TG云推] {standard_name} 进度达 98%"))
                                    notified_98_up = True
                                        
                                yield chunk
                        async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=1800.0, write=60.0, pool=None), trust_env=False) as h: 
                            resp = await h.put(put_url, content=file_iter(), headers=headers)
                    
                    if resp.json().get("code") == 200:
                        try: os.remove(local_path)
                        except: pass
                        
                        ver_tag = f".{history_tags}" if history_tags else ""
                        
                        if not is_movie and ep_num is not None:
                            record_history(clean_base, file_season, ep_num, history_tags)
                            await notify_steward_log(f"📝 [后台重推-补录] {clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}")
                        
                        success_msg = f"🎉 **[云端落盘成功]** ➔ `{standard_name}`\n✅ 该文件已成功推送至天翼云，释放本地空间！"
                        
                        # 🔥 沿用同一个消息卡片更新
                        if status_msg:
                            try: await status_msg.edit_text(success_msg)
                            except: pass
                        else:
                            try: await app.send_message(COMMAND_CENTER_CHAT, success_msg)
                            except Exception:
                                try: await app.send_message("me", success_msg)
                                except: pass
                        
                        try:
                            target_dir = os.path.dirname(target_full)
                            current_mount = get_mount_root()
                            STAGING_BASE_DIR = "/storage/emulated/0/Download/189cas"
                            cas_file_name = f"{standard_name}.cas"
                            cas_target_full = f"{target_full}.cas"
                            sub_path = target_dir.replace(current_mount, "", 1)
                            local_cas_dir = f"{STAGING_BASE_DIR}{sub_path}"
                            os.makedirs(local_cas_dir, exist_ok=True)
                            final_cas_path = os.path.join(local_cas_dir, cas_file_name)
                            
                            # 传给 CAS 继续复用同一条消息卡片
                            asyncio.create_task(bg_fetch_cas_task(cas_target_full, final_cas_path, sub_path, cas_file_name, status_msg=status_msg))
                        except Exception as e:
                            await notify_steward_log(f"⚠️ [后台CAS派发失败]: {e}", level="WARNING")
                        break
            except Exception: pass
            await asyncio.sleep(60)
    finally:
        ver_tag = f".{history_tags}" if history_tags else ""
        task_lock_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}" if not is_movie else f"{clean_base}{ver_tag}"
        GLOBAL_ACTIVE_LOCKS.discard(task_lock_key)

# =================================================================
# 🧲 Aria2 异步执行任务 (磁力引擎) - 加入开机唤醒记忆接口
# =================================================================
async def handle_magnet_execution(client, message, magnet_link, config_text, resume_gid=None):
    args = config_text.split()
    if len(args) < 2: 
        if message: await message.reply_text("⚠️ 格式错误。请至少提供 `剧名` 和 `分类`。")
        return

    STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]
    cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
    if cat_idx == -1: 
        if message: await message.reply_text("⚠️ 必须提供有效分类(如 国漫, 华语剧, 电影 等)。")
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

    raw_input = " ".join(args) if args else "未知磁力"
    if "|" in raw_input: search_kw, drama_name = [x.strip() for x in raw_input.split("|", 1)]
    else: search_kw = drama_name = raw_input

    year, _ = await fetch_tmdb_details(drama_name)
    if custom_year: year = custom_year

    folder_name = f"{drama_name} ({year})" if year else drama_name
    if version_suffix: folder_name = f"{folder_name} {version_suffix}"
    is_movie = "电影" in category or category in ["演唱会", "纪录片"]

    # 注册独立的磁力任务锁，用于 /cancel 剧名
    magnet_task_key = f"磁力_{drama_name}"
    GLOBAL_ACTIVE_LOCKS.add(magnet_task_key)
    
    task_aborted = True
    original_gid = None

    try:
        if not resume_gid:
            payload = {
                "jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.addUri",
                "params": [f"token:{ARIA2_RPC_SECRET}", [magnet_link], {"dir": ARIA2_DOWNLOAD_DIR}]
            }

            try:
                if message: status_msg = await message.reply_text(f"🚀 **[Aria2 涡轮点火]**\n正在将磁力喂给后端 P2P 引擎...")
                else: status_msg = await client.send_message(COMMAND_CENTER_CHAT, f"🚀 **[Aria2 涡轮点火]**\n正在将磁力喂给后端 P2P 引擎...")
                
                async with httpx.AsyncClient(timeout=10.0) as h_client:
                    resp = await h_client.post(ARIA2_RPC_URL, json=payload)
                    res_json = resp.json()
                    if "error" in res_json:
                        error_msg = res_json["error"].get("message", "")
                        if "already registered" in error_msg.lower():
                            return await status_msg.edit_text(f"❌ **添加失败**: 任务已在 Aria2 队列中，请勿重复添加！")
                        else:
                            return await status_msg.edit_text(f"❌ Aria2 拒绝接单: `{error_msg}`")
                    gid = res_json.get("result")
                    if not gid: return await status_msg.edit_text(f"❌ Aria2 未返回 GID！")
                    
                    original_gid = gid
                    add_magnet_task(original_gid, magnet_link, config_text)
                    tracked_gids = {gid} 
                    
            except Exception as e:
                return await status_msg.edit_text(f"❌ Aria2 连接失败: {e}")
        else:
            gid = resume_gid
            original_gid = resume_gid
            tracked_gids = {gid}
            if message: status_msg = await message.reply_text(f"🔄 **[任务恢复]** 重新接管磁力任务 `{drama_name}`...")
            else: status_msg = await client.send_message(COMMAND_CENTER_CHAT, f"🔄 **[任务恢复]** 重新接管磁力任务 `{drama_name}`...")

        last_ui_time = 0
        meta_wait_time = 0
        st_data = {}
        
        notified_50 = False
        notified_100 = False
        
        while True:
            await asyncio.sleep(3.0)
            
            # 🔥 触发拉闸：传入整个链条的 GID
            if GLOBAL_STOP_SWEEP or "ALL" in GLOBAL_CANCEL_TASKS or magnet_task_key in GLOBAL_CANCEL_TASKS:
                await wipe_magnet_task(tracked_gids)
                return await status_msg.edit_text(f"🛑 **[任务拉闸]** 磁力任务 `{drama_name}` 已被强行单独销毁！")
                
            try:
                p2 = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellStatus", "params": [f"token:{ARIA2_RPC_SECRET}", gid]}
                async with httpx.AsyncClient(timeout=5.0) as h_client:
                    resp2 = await h_client.post(ARIA2_RPC_URL, json=p2)
                    res_json = resp2.json()
                    
                    if "error" in res_json:
                        return await status_msg.edit_text("🛑 任务已从Aria2面板中消失或被外部手动删除！")
                        
                    st_data = res_json.get("result", {})

                followed_by = st_data.get("followedBy")
                if followed_by and len(followed_by) > 0:
                    gid = followed_by[0]  
                    tracked_gids.add(gid) # 🔥 捕获衍生出的实体视频 GID
                    meta_wait_time = 0
                    notified_50 = False
                    notified_100 = False
                    
                    await status_msg.edit_text(f"🧲 **[解析成功]** 获取到种子文件，开始正式拉取实体视频...")
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
                        return await status_msg.edit_text(f"❌ **种子解析报错**: {st_data.get('errorMessage')}")
                    
                    if status == "complete":
                        try:
                            # 修复丢失的 id 字段，保证跨队列捕获生效
                            p_active = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellActive", "params": [f"token:{ARIA2_RPC_SECRET}"]}
                            p_waiting = {"jsonrpc": "2.0", "id": "tg_bot", "method": "aria2.tellWaiting", "params": [f"token:{ARIA2_RPC_SECRET}", 0, 100]}
                            async with httpx.AsyncClient() as h:
                                res_a = await h.post(ARIA2_RPC_URL, json=p_active)
                                res_w = await h.post(ARIA2_RPC_URL, json=p_waiting)
                                tasks = res_a.json().get("result", []) + res_w.json().get("result", [])
                                for t in tasks:
                                    if t.get("infoHash") == info_hash and "info" in t.get("bittorrent", {}):
                                        gid = t.get("gid")
                                        tracked_gids.add(gid) # 🔥 捕获跨队列实体 GID
                                        await status_msg.edit_text(f"🧲 **[解析成功]** 跨队列捕获实体任务，无缝切入...")
                                        break
                        except: pass
                    
                    meta_wait_time += 3
                    if meta_wait_time > 300:
                        await wipe_magnet_task(tracked_gids)
                        return await status_msg.edit_text("❌ **超时中止**：连续 5 分钟未获取到种子(死链)，已销毁！")
                    
                    now = time.time()
                    if now - last_ui_time > 6.0:
                        last_ui_time = now
                        await status_msg.edit_text(f"🧲 **[磁力涡轮]** `{drama_name}`\n⏳ 正在获取元数据(种子)中...")
                    continue 

                if status == "error":
                    return await status_msg.edit_text(f"❌ **Aria2 下载报错**: {st_data.get('errorMessage')}")

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

                    await status_msg.edit_text(f"🧲 **[磁力涡轮]** `{drama_name}`\n📈 下载: **{sp_mb:.2f} MB/s** | 📤 上传: **{up_mb:.2f} MB/s**\n🤝 连接: **{connections}** 节点\n⏳ 进度: **{pct}%** | 剩余: **{eta_txt}**")
            except Exception: pass

        await status_msg.edit_text(f"✅ **[下载完毕]** 启动清道夫协议！\n正在全量检索下载物，提取视频并执行智能洗名...")

        # ==========================
        # 🧹 提取与毁灭级清道夫
        # ==========================
        files = st_data.get("files", [])
        VIDEO_EXTS = ['.mp4', '.mkv', '.ts', '.avi', '.rmvb', '.flv', '.webm', '.iso']
        extracted_videos = []
        
        for f in files:
            f_path = f.get("path")
            if not f_path or not os.path.exists(f_path): continue
            if os.path.getsize(f_path) <= 0: continue 

            if os.path.splitext(f_path)[1].lower() in VIDEO_EXTS:
                orig_name = os.path.basename(f_path)
                file_ep_num = extract_pure_episode(orig_name, drama_anchor=drama_name)
                
                # 🔥 调用磁力专属智能洗名
                new_name = smart_rename(orig_name, drama_name, ep_num=file_ep_num, season=folder_season, is_movie=is_movie)
                
                safe_dest = os.path.join(LOCAL_TEMP_DIR, new_name)
                shutil.move(f_path, safe_dest)
                
                extracted_videos.append({
                    "path": safe_dest,
                    "name": new_name,
                    "size": os.path.getsize(safe_dest),
                    "ep_num": file_ep_num
                })

        # 🔥 传入追踪到的完整 GID 链条（原磁力、跨列、衍生实体），确保满门抄斩！
        await wipe_magnet_task(tracked_gids)

        if not extracted_videos:
            if st_data.get("status") == "removed": return await status_msg.edit_text("🛑 任务已被手动从后台终止并清理。")
            return await status_msg.edit_text("❌ 提取失败：该链接仅包含垃圾文件或死链残片。所有残留已干干净净地销毁。")

        await status_msg.edit_text(f"🎉 **[清洗成功]** 共提取 {len(extracted_videos)} 个纯净视频！\n原始垃圾已焚毁，正由磁力专属通道直推天翼云...")

        # 🚀 移交【磁力专属通道】
        current_mount = get_mount_root()
        for idx, v in enumerate(extracted_videos):
            if is_movie: target_dir = f"{current_mount}/{category}/{folder_name}"
            else: target_dir = f"{current_mount}/{category}/{folder_name}/Season {folder_season}"
                
            target_full = f"{target_dir}/{v['name']}".replace("//", "/")
            
            # 🔥 仅首个视频复用面板消息一镜到底，其余新建消息条，避免并发冲突！
            pass_msg = status_msg if idx == 0 else None
            
            asyncio.create_task(magnet_upload_task(
                local_path=v['path'], 
                target_full=target_full, 
                total_bytes=v['size'], 
                clean_base=drama_name.replace(" ", "."), 
                file_season=folder_season, 
                ep_num=v['ep_num'], 
                is_movie=is_movie, 
                standard_name=v['name'], 
                version_suffix=version_suffix,
                status_msg=pass_msg,
                original_gid=original_gid
            ))
            
        task_aborted = False # 只有一切执行成功切交接完毕后，才标记为非流产
            
    finally:
        # 如果任务在中途意外中断或人为终止（未成功进入上传流程），保底清空记忆档案
        if task_aborted and original_gid:
            remove_magnet_task(original_gid)
            
        GLOBAL_ACTIVE_LOCKS.discard(magnet_task_key)
        GLOBAL_CANCEL_TASKS.discard(magnet_task_key)

# =================================================================
# 🎯 [指令响应大网关] (根据Pyrogram严格排序)
# =================================================================
STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]

@app.on_message(filters.command(["sub", "unsub", "list", "add", "del", "go", "history", "ping", "rm", "clean", "scan", "rmh", "setdir", "cancel", "mag"]) & filters.user("me"))
async def manage_system_commands(client, message):
    global GLOBAL_STOP_SWEEP
    command = message.command[0].lower()
    config = load_listener_config()
    
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
        prompt = await message.reply_text("🧲 磁力链接已直接捕获！请回复: `剧名|别名 分类 [S季数] [年份] [v=版本]`")
        PENDING_MAGNETS[message.chat.id] = {"link": magnet_link, "mag_msg_id": message.id, "prompt_msg_id": prompt.id}
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
                    s_text = f"S{f_season:02d}"
                    size_txt = f" / {min_mb}-{max_mb if max_mb != 999999 else '上限'}MB" if (min_mb or max_mb != 999999) else ""
                    end_txt = f" / 🏁{end_ep}集" if end_ep != 9999 else " / ♾️连载"
                    ver_txt = f" / 🏷️{ver}" if ver else ""
                    line = f"  └ 🎬 `{search_kw}` ({s_text}{size_txt} / ⏱️{freq}{end_txt}{ver_txt})\n"
                    if len(msg_text) + len(line) > 4000:
                        await message.reply_text(msg_text)
                        msg_text = ""
                    msg_text += line
        if msg_text: await message.reply_text(msg_text)
        return

    if command == "scan":
        GLOBAL_STOP_SWEEP = False
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
        if len(args) < 2: return await message.reply_text("⚠️ 语法：`/sub [剧名] [分类] [频率] [v=版本] [f=参数] [end=完结集数] [可选:年份/季数]`")
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
            
        target_pools = [specific_channel] if specific_channel else list(config.get("trusted_channels", {}).keys())
        if not target_pools: return await message.reply_text("⚠️ 你的大盘里没有任何频道！")

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
        if end_ep is None: end_ep = 9999 if category in ["国漫", "日漫", "综艺"] else fetched_end
                
        for chat_id in target_pools:
            if "monitored_dramas" not in config["trusted_channels"][chat_id]: config["trusted_channels"][chat_id]["monitored_dramas"] = {}
            config["trusted_channels"][chat_id]["monitored_dramas"][drama_key] = {
                "search_kw": search_kw, "version": version_suffix, "file_version": file_suffix, 
                "category": category, "folder_season": folder_season, "file_season": file_season, 
                "min_ep": min_ep, "min_mb": min_mb, "max_mb": max_mb, "frequency": frequency,
                "year": custom_year, "end_ep": end_ep
            }
            
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        end_display = "♾️无限连载" if end_ep == 9999 else f"{end_ep} 集杀青"
        
        # 🔥 修改：延迟删除，给界面反馈留出缓冲时间
        try:
            await asyncio.sleep(2.5)
            await message.delete()
        except: pass
        await message.reply_text(f"✅ **订阅成功**: `{drama_name}`\n🏷️ **追踪**: `{version_suffix or '默认'}`\n📁 **路径**: `Season {folder_season}` (命名: `S{file_season:02d}`)\n⚖️ **画质锁定**: `{min_mb}-{max_mb} MB`\n🏁 **完结**: `{end_display}`\n🚀 **启动扫荡！**")
        
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
        if not args: return await message.reply_text("⚠️ 语法：`/go [剧名] [分类] [年份] [v=版本] [季数]`")
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
                GLOBAL_ROUTE_CACHE.update({"folder_name": matched_item["folder_name"], "category": matched_item["category"], "folder_season": folder_s, "file_season": file_s, "year": matched_item.get("year", ""), "version": matched_item.get("version", ""), "expire_time": time.time() + 300, "manual_ep": None})
                
                # 🔥 修改：阅后即焚，延迟删除
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
        routes[folder_name] = {"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "version": version_suffix, "file_version": file_suffix, "created_at": int(time.time())}
        save_tg_routes(routes)
        GLOBAL_ROUTE_CACHE.update({"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "version": version_suffix, "file_version": file_suffix, "expire_time": time.time() + 360, "manual_ep": None})
        
        # 🔥 修改：阅后即焚，延迟删除
        try:
            await asyncio.sleep(2.5)
            await message.delete()
        except: pass
        return await message.reply_text(f"✅ 新航线已打通：\n📁 目标锁定: `{folder_name}`\n请直接转发视频！")

@app.on_message(filters.text & filters.regex(r"(?i)^magnet:\?xt=") & filters.user("me"))
async def catch_magnet_link(client, message):
    magnet_link = message.text.strip()
    prompt = await message.reply_text("🧲 **[雷达警报] 已直接捕获磁力！**\n请直接回复本条消息提供归属信息：\n👉 **格式**: `剧名|别名 分类 [S季数] [年份] [v=版本]`")
    PENDING_MAGNETS[message.chat.id] = {"link": magnet_link, "mag_msg_id": message.id, "prompt_msg_id": prompt.id}

@app.on_message(filters.text & ~filters.media & filters.user("me"))
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

    if message.chat.id in PENDING_MAGNETS and not message.text.startswith("/"):
        mag_data = PENDING_MAGNETS.pop(message.chat.id)
        
        # 🔥 修改：后台异步延迟 10 秒删除，立即触发磁力下载避免卡顿感
        msgs_to_delete = [message.id]
        if isinstance(mag_data, dict):
            magnet_link = mag_data["link"]
            msgs_to_delete.extend([mag_data["mag_msg_id"], mag_data["prompt_msg_id"]])
        else:
            magnet_link = mag_data
            
        async def delay_delete():
            await asyncio.sleep(10)
            try: await client.delete_messages(message.chat.id, msgs_to_delete)
            except: pass
        asyncio.create_task(delay_delete())
        
        await handle_magnet_execution(client, message, magnet_link, message.text.strip())

@app.on_message(filters.video | filters.document)
@app.on_edited_message(filters.video | filters.document)
async def media_routing_gateway(client, message):
    try:
        config = load_listener_config()
        chat_id_to_check = str(message.chat.id) if message.chat else ""
        original_channel_id = ""
        parent_text = ""
        if message.reply_to_message:
            parent = message.reply_to_message
            parent_text = parent.caption or parent.text or ""
            if parent.forward_from_chat: original_channel_id = str(parent.forward_from_chat.id)
            elif parent.sender_chat: original_channel_id = str(parent.sender_chat.id) 
            if parent.reply_to_message: parent_text += f" {parent.reply_to_message.caption or parent.reply_to_message.text or ''}"

        matched_channel = None
        for k in config.get("trusted_channels", {}):
            if chat_id_to_check and (k in chat_id_to_check or chat_id_to_check in k): matched_channel = k; break
            if original_channel_id and (k in original_channel_id or original_channel_id in k): matched_channel = k; break

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
            
            auto_list = extract_media_tags(text_to_scan).split('.')
            res_list = [t for t in auto_list if t.lower() in ["2160p", "1080p", "720p", "4k", "8k"]]
            temp_tags = res_list + (db_file_version.split('.') if db_file_version else []) + (db_version.split('.') if db_version else [])
            preview_tags = ".".join(dict.fromkeys(temp_tags))
            
            folder_season = route_info.get("folder_season", route_info.get("file_season", 1))
            file_season = route_info.get("file_season", folder_season)
            year = route_info.get("year")
            if not year: year, _ = await fetch_tmdb_details(search_kw)
            
            pure_drama_name = drama_key[:-len(f"_{db_version}")] if db_version and drama_key.endswith(f"_{db_version}") else drama_key
            folder_name = f"{pure_drama_name} ({year})" if year else pure_drama_name
            if db_version: folder_name = f"{folder_name} {db_version}"  
            
            # 🔥 修复别名频弹消息：使用纯净剧名替代别名进行预查，查到就静默跳过
            clean_base_for_check = pure_drama_name.replace(" ", ".")
            if check_history(clean_base_for_check, file_season, ep_num, preview_tags): return
            
            override_info = (folder_name, route_info["category"], folder_season, file_season, year, ep_num, db_version, db_file_version, matched_channel, drama_key, route_info.get("end_ep", 9999))
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
        if len(override_info) >= 11: folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix, src_chat_id, src_drama_key, src_end_ep = override_info[:11]
        elif len(override_info) >= 8: folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix = override_info[:8]
        else:
            folder, cat, folder_season, file_season, year, ep_num = override_info[:6]
            version_suffix = file_suffix = ""
    else:
        folder, cat = GLOBAL_ROUTE_CACHE["folder_name"], GLOBAL_ROUTE_CACHE["category"]
        folder_season, file_season = GLOBAL_ROUTE_CACHE["folder_season"], GLOBAL_ROUTE_CACHE["file_season"]
        year = GLOBAL_ROUTE_CACHE.get("year", "")
        version_suffix = GLOBAL_ROUTE_CACHE.get("version", "")
        file_suffix = GLOBAL_ROUTE_CACHE.get("file_version", "") 
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
        target_dir = f"{current_mount}/{cat}/{folder}"
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
        target_dir = f"{current_mount}/{cat}/{folder}/Season {folder_season}"
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
        target_full = f"{target_dir}/{standard_name}".replace("//", "/")
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
                        # 🔥 修改：无差别摧毁指令源消息，只要机器人有删除权限
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
            local_path=local_path, target_full=target_full, total_bytes=total_bytes, 
            clean_base=clean_base, file_season=file_season, ep_num=ep_num, 
            is_movie=is_movie, standard_name=standard_name, history_tags=combined_history_tags,
            status_msg=status
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
                
                override_info = (folder_name, category, folder_season, file_season, year, ep_num, db_version, db_file_version, str(chat_id), drama_key, end_ep)
                try: status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[哨兵发车]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
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
                    # 任务在Aria2中已被删除或丢失，默默销毁账本记录
                    remove_magnet_task(gid)
                    continue
                
                # 启动后台恢复
                asyncio.create_task(handle_magnet_execution(client, None, info["magnet_link"], info["config_text"], resume_gid=gid))
                await asyncio.sleep(2)
        except Exception: pass

if __name__ == "__main__":
    clean_orphan_temp_files(max_age_hours=24) 
    load_listener_config()
    async def start_system():
        await app.start()
        await notify_steward_log("✅ 机器人重启完毕，TG下载与Aria2磁力引擎已全线上线！")
        asyncio.create_task(startup_catchup_sweep(app))
        asyncio.create_task(smart_patrol_daemon(app))
        asyncio.create_task(resume_magnet_tasks(app)) # 唤醒记忆
        try: await app.send_message(COMMAND_CENTER_CHAT, "🤖 下载与上传推引擎已启动！")
        except: pass
        await idle()
        await app.stop()
    app.run(start_system())
```
2.登陆帐号
在前台手动点火，直接用 Python 原生命令把代码跑起来，让它把输出打在你的屏幕上：
```
python3 ~/189py/autotg.py
```
回车之后，盯着屏幕，你会看到 Pyrogram 极度硬核的认证提示依次弹出：

Enter phone number or bot token:
👉 既然注释了 token，直接输入你的 Telegram 注册手机号，必须带国际区号！比如中国大陆就输入 +8613800138000，然后回车。

Is "+86 138 0013 8000" correct? (y/N):
👉 输入 y 回车确认。

Enter confirmation code:
👉 此时，你的 Telegram 官方客户端（手机或电脑版）会收到一条系统发来的 5 位数验证码。在 Termux 里敲进去并回车。

Enter password (hint: ***):
👉 (注意：只有你开了两步验证密码才会有这一步)。如果有，输入你的两步验证密码（注意：Linux 终端输入密码时屏幕上不会显示任何字符，这是正常的防偷窥保护，凭感觉盲打完直接回车即可）。

一旦认证成功，屏幕上会刷出一堆日志，最后显示：
🤖 工业级全栖搬运中枢 (V15 三维雷达全自动版) 满血上线！
并且你的目录里会自动生成一个极其珍贵的授权凭证文件：tg_robust_leecher_v15.session。

有了这个文件，脚本以后就彻底记住了你是谁，再也不用输密码了。

现在，按下键盘上的 Ctrl + C，强行把前台运行的脚本关掉。

#### 3.指令说明：
📖 V15.10 核心遥控指令清单
📡 一、 雷达全自动阵列（适用于未来更新的剧集）
/ping ➔ 💓 查看系统是否活着、存活时间、雷达正在盯防多少部剧。

/list ➔ 📋 查看当前所有雷达监控大名单。

/add [频道ID] [别名] ➔ 🏢 把一个发资源的群加入信任大名单。

(例：/add_channel -1001234567 顶级原盘群)

/del [频道ID] ➔ 🗑️ 把某个群从大名单里踢出去。

/sub [分类] [剧名] [季数] [起步集] ➔ 🎯 部署雷达！（分类和剧名谁在前都可以，闭眼发）。

完整语法：/sub [剧名关键字] [分类] [文件夹季数] [起步集数] [可选: 文件大小范围] [可选: 文件名季数] [可选: 剧名年份] [可选: v=剧名文件夹版本号] [可选: f=文件名参数] [可选: 频道ID] [可选: end=剧总集数]

别名：/sub 仙剑奇侠传三|仙剑奇侠传叁 国漫

(例：/sub 国漫 凡人修仙传 1 15 ➔ 从第15集开始死盯凡人修仙传)

/unsub [剧名] ➔ ⛔ 剧追完了？取消雷达监控。

(例：/unsub 凡人修仙传)

🎫 二、 手工霸王发车（适用于补以前的旧视频）
/history ➔ 📂 查看曾经发过车的历史航线记录（方便你想复用）。

/go [关键字] ➔ 🚀 秒开旧车！ 模糊搜索历史记录，一秒复用。

(例：发 /go 卧底，自动匹配《宗门里除了我都是卧底》，直接发车)

/go [分类] [新剧名] ➔ 🆕 秒开新车！ 历史里没有的剧，直接强行建档。

(例：/go 日漫 咒术回战)

#E[数字] ➔ 🔢 强行纠正/覆写下一集的集数。

完整语法：/go [剧名关键字] [分类] [文件夹季数] [可选: 文件名季数]

(例：视频名字太乱提取不出集数，你直接发 #E12，下一集强制按 12 集命名)

🧹 三、 清理与维护（本次为你紧急加装！）
/rm [剧名关键字] ➔ 💥 删除历史航线！ 把不再需要的死档从 /history 里永久抹除。

(例：/rm 卧底 ➔ 瞬间清理账本，保持后台极度干净)
/rmh [剧名关键字] ➔ 删除下载历史记录
/clean 清除下载碎片

/scan 拉取订阅下载（+剧名可直接拉取单剧）

```
终极备忘录：全指令参数详解图鉴
你可以把这段保存在你的记事本里，随时查阅：

/sub - ➕ 自动订阅 (可直接回复视频抓取频道)

完整语法：/sub [剧名] [分类] [最小MB-最大MB] [频率:日更/周一~周日] [可选:年份] [可选:v=版本号] [可选:文件夹季数] [可选:S文件名季数]

实战举例：/sub 将夜 国漫 1500-3000 周四 2026 v=HDR 1 S2

快捷玩法：直接转发目标频道的视频，对它点击【回复】，然后输入上述参数（此时无需手填频道ID，机器自动抓取建档）。

/unsub - 🗑️ 取消订阅 (全网通杀或精准单杀)

完整语法：/unsub [剧名] [可选:v=版本号] [可选:-100开头的频道ID]

实战举例：/unsub 将夜 v=HDR (只取消所有频道的HDR版本)

实战举例：/unsub 将夜 -1001234567 (只踢掉某个发垃圾画质的频道)

/go - 🚀 手动发车 (开辟航线并锁死物理文件夹)

完整语法：/go [剧名关键字] [可选:分类] [可选:年份] [可选:v=版本号] [可选:文件夹季数] [可选:S文件名季数]

实战举例：/go 绝命毒师 欧美剧 2018 v=4K 1 (将后续文件死锁进 Season 1 文件夹)

配合遥控：发车后，发送 #S2E8 (只改变文件的刮削名字为第二季第八集，绝不改变刚才锁死的 Season 1 文件夹)。

/list - 📡 查看雷达大盘排班与状态

语法：直接发送 /list 即可，无参数。

/scan - 🔍 强行触发一次全网补漏扫荡

语法：直接发送 /scan 即可，无参数。

/help - 📖 查看随身说明书与指令语法

语法：直接发送 /help 或 /h 即可。

```
### 五、机器人指令菜单

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
list - 📡 查看雷达大盘排班与状态
scan - 🔍 强行触发一次全网补漏扫荡
history - 📋 查看曾经发过车的历史航线记录
rm - ❌ 删除历史航线！ 把不再需要的死档从history 里永久抹除
rmh - ❎ 删除下载历史记录
clean - ⭕️ 清除下载碎片
cancel - ⭕️ 终止下载 (all/剧名(+e01/s01)/.mp4)
setdir - ❇️ 设置上传目录
ping - 💓 查看系统是否活着、存活时间、雷达正在盯防多少部剧
help - 📖 查看随身说明书与指令语法
```

#### 六、新填磁力下载

1.cancel+剧名  终止下载与清理文件

2.使用 Python 获取所有 GID

请在你的服务器上创建一个名为 get_aria2_tasks.py 的文件，并将下方代码填入（确保已安装 aria2p 库，若未安装可运行 pip install aria2p）：

```
import aria2p

# 这里的配置请根据你的实际 Aria2 设置修改
# 如果是本地运行，通常 host='localhost', port=6800, secret=''
aria2 = aria2p.API(
    aria2p.Client(
        host="http://localhost",
        port=6800,
        secret="xxsky1127"  # 如果你有设置 RPC 密钥，请填在这里
    )
)

def list_all_tasks():
    # 获取所有正在下载的任务
    downloads = aria2.get_downloads()
    
    print(f"{'任务名称 (文件名)':<40} | {'GID':<20}")
    print("-" * 65)
    
    for download in downloads:
        # download.name 是文件名，download.gid 是我们要找的那个 ID
        print(f"{download.name:<40} | {download.gid:<20}")

if __name__ == "__main__":
    list_all_tasks()
```
启动
```
python3 ~/189py/get_aria2_tasks.py
```
查看结果： 你会看到类似这样的输出：
```
任务名称 (文件名)                          | GID                 
-----------------------------------------------------------------
灿如繁星 1080p...                         | 7d4a...5a91         
逝爱迷局 S1...                            | ed83...a2           
暗金 S1...                                | 0bbf...47           
3.  **获取数据：** 找到“灿如繁星”那一栏，后面那一串长字符就是你缺失的 `GID`。
```
更新配置文件
拿到这个 7d4a...5a91（假设值）之后，你就可以把它手动填回tg_magnet_tasks.json里的JSON 文件了