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
os.makedirs(LOCAL_TEMP_DIR, exist_ok=True)

TMDB_API_KEY = "9c88e18e43543c8ff195c631aaa0d2fa" 
START_TIME = time.time()

TG_SETTINGS_DB = os.path.join(BASE_DIR, "tg_settings.json")

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
# 🤫 日志与上报中枢配置
# =================================================================
logging.basicConfig(level=logging.INFO, format='[%(asctime)s] %(message)s', datefmt='%H:%M:%S')
logging.getLogger("pyrogram").setLevel(logging.WARNING)
logging.getLogger("httpx").setLevel(logging.WARNING)
logger = logging.getLogger("TGEngine")

app = Client("tg_robust_leecher_v15", api_id=API_ID, api_hash=API_HASH)

async def notify_steward_log(msg, level="INFO"):
    logger.info(f"[{level}] {msg}")
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            await client.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": level, "msg": msg})
    except Exception: pass

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

# 🔥 终极版历史查账模块：无序集合比对，防重下且支持多版本共存
def check_history(drama, file_season, ep, version=""):
    history = load_history()
    ver_tag = f".{version}" if version else ""
    base_name = drama.rstrip(".").replace(" ", ".")
    key = f"{base_name}.S{file_season:02d}E{ep:02d}{ver_tag}"
    
    # 1. 完美精确命中，直接放行
    if key in history:
        return True
        
    # 2. 模糊查账逻辑：剥离 SDR 并无视标签顺序
    prefix = f"{base_name}.S{file_season:02d}E{ep:02d}"
    
    current_tags = set(t for t in version.lower().split('.') if t)
    current_tags.discard("sdr")
    
    for hist_key in history:
        if hist_key.startswith(prefix):
            hist_version = hist_key[len(prefix):].strip(".")
            hist_tags = set(t for t in hist_version.lower().split('.') if t)
            hist_tags.discard("sdr")
            
            if current_tags == hist_tags:
                return True
                
    return False

# 🔥 严谨入账模块（严格记录实际排队的特征）
def record_history(drama, file_season, ep, version=""):
    history = load_history()
    ver_tag = f".{version}" if version else ""
    base_name = drama.rstrip(".").replace(" ", ".")
    key = f"{base_name}.S{file_season:02d}E{ep:02d}{ver_tag}"
    
    if key not in history:
        history[key] = int(time.time())
        try:
            with open(TG_HISTORY_DB, 'w', encoding='utf-8') as f: 
                json.dump(history, f, ensure_ascii=False, indent=2)
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

# =================================================================
# 🚦 全局大盘与双轨排队锁 (双车道解耦)
# =================================================================
GLOBAL_ROUTE_CACHE = {"folder_name": "", "category": "", "folder_season": 1, "file_season": 1, "year": "", "version": "", "expire_time": 0, "manual_ep": None}
GLOBAL_ACTIVE_LOCKS = set() 
GLOBAL_CANCEL_TASKS = set() 
GLOBAL_TRANSFER_LOCK = asyncio.Semaphore(1) 
GLOBAL_UPLOAD_LOCK = asyncio.Semaphore(1)   

# 🔥 终极 TMDB 抓取器：一次性获取年份和总集数
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
                    # 获取该剧的详细信息以拿到总集数
                    det_url = f"https://api.themoviedb.org/3/tv/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN"
                    det_res = await client.get(det_url)
                    total_eps = det_res.json().get("number_of_episodes", 9999)
                    
                return year, total_eps
    except Exception: pass
    return datetime.now().strftime("%Y"), 9999

def extract_pure_episode(search_text, drama_anchor=None):
    text = search_text
    if drama_anchor and drama_anchor.lower() in text.lower():
        idx = text.lower().find(drama_anchor.lower())
        text = text[idx:]
        
    m = re.search(r'(?i)E0*(\d+)', text)
    if m: return int(m.group(1))
    
    m = re.search(r'第\d+[季部].*?第\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    m = re.search(r'第\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    m = re.search(r'(?<![第\d])\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    
    clean_text = re.sub(r'(?i)(h264|h265|x264|x265|720p|1080p|2160p|4k|8k|web-dl|webrip)', '', text)
    m_trail = re.search(r'(?<!\d)0*(\d{1,3})(?!\d)', clean_text)
    if m_trail and not (1900 < int(m_trail.group(1)) < 2100):
        return int(m_trail.group(1))
        
    m_cn = re.search(r'第\s*([一二三四五六七八九十零百]+)\s*[集话期更]', text)
    if m_cn:
        cn_str = m_cn.group(1)
        cn_map = {"一":1, "二":2, "三":3, "四":4, "五":5, "六":6, "七":7, "八":8, "九":9, "十":10}
        if cn_str in cn_map: return cn_map[cn_str]
        if cn_str.startswith("十") and len(cn_str) == 2: return 10 + cn_map.get(cn_str[1], 0)
        if len(cn_str) == 3 and cn_str[1] == "十": return cn_map.get(cn_str[0],0)*10 + cn_map.get(cn_str[2],0)
        if cn_str.endswith("十") and len(cn_str) == 2: return cn_map.get(cn_str[0],0)*10
        
    return None

# =================================================================
# 🏷️ TG生态定制版：媒体标签嗅探引擎 
# =================================================================
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

async def bg_upload_retry_task(local_path, target_full, total_bytes, clean_base, file_season, ep_num, is_movie, standard_name, version_suffix=""):
    try:
        await asyncio.sleep(600)  
        while os.path.exists(local_path):
            try:
                put_url = f"{OLIST_URL}/api/fs/put"
                headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
                
                async with GLOBAL_UPLOAD_LOCK:
                    with open(local_path, "rb") as f_upload:
                        async def file_iter():
                            while True:
                                chunk = f_upload.read(1024 * 1024)
                                if not chunk: break
                                yield chunk
                        async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=900.0, write=60.0, pool=None), trust_env=False) as h: 
                            resp = await h.put(put_url, content=file_iter(), headers=headers)
                    
                    if resp.json().get("code") == 200:
                        try: os.remove(local_path)
                        except: pass
                        if not is_movie and ep_num is not None:
                            record_history(clean_base, file_season, ep_num, version_suffix)
                            ver_tag = f".{version_suffix}" if version_suffix else ""
                            await notify_steward_log(f"📝 [后台重推-补录] {clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}")
                        break
            except Exception:
                pass
            await asyncio.sleep(600)
    finally:
        ver_tag = f".{version_suffix}" if version_suffix else ""
        task_lock_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag}" if not is_movie else f"{clean_base}{ver_tag}"
        GLOBAL_ACTIVE_LOCKS.discard(task_lock_key)

# =================================================================
# 🚀 核心媒体传输引擎
# =================================================================
async def process_media_transfer(client, message, status, override_info=None):
    media = message.video or message.document
    if not media: return
    raw_file = getattr(media, "file_name", f"TG_{message.id}.mp4")
    _, ext = os.path.splitext(raw_file)
    if not ext: ext = ".mp4"
    
    src_chat_id = src_drama_key = src_end_ep = None
    
    if override_info:
        # 支持传参解包扩展，引入频道ID、剧名Key和最终集数用于自动杀青
        if len(override_info) >= 11:
            folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix, src_chat_id, src_drama_key, src_end_ep = override_info[:11]
        elif len(override_info) >= 8:
            folder, cat, folder_season, file_season, year, ep_num, version_suffix, file_suffix = override_info[:8]
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
        else: 
            ep_num = extract_pure_episode(f"{message.caption or ''} {raw_file}")
            
    text_to_scan = f"{message.caption or ''} {raw_file}"
    auto_tags = extract_media_tags(text_to_scan)
            
    is_movie = "电影" in cat or cat in ["演唱会", "纪录片"]
    
    clean_base = re.sub(r'\s*\(\d{4}\)', '', folder).strip()
    if version_suffix and clean_base.endswith(version_suffix):
        clean_base = clean_base[:-len(version_suffix)].strip()
    clean_base = clean_base.replace(" ", ".")
    
    # ==========================================
    # 🔥 双轨分离制：物理文件名只认 f=，底层账本认全套
    # ==========================================
    
    # A轨道：生成物理文件的标签 (过滤掉了 v= 版本组)
    raw_physical_tags = []
    if auto_tags: raw_physical_tags.extend(auto_tags.split('.'))
    if file_suffix: raw_physical_tags.extend(file_suffix.split('.'))
    
    final_physical_tags = []
    for t in raw_physical_tags:
        if t and t not in final_physical_tags: final_physical_tags.append(t)
        
    combined_physical_tags = ".".join(final_physical_tags)
    ver_tag_physical = f".{combined_physical_tags}" if combined_physical_tags else ""
    
    # B轨道：生成历史账本和防撞锁的标签 (包含 v=)
    raw_history_tags = raw_physical_tags.copy()
    if version_suffix: raw_history_tags.extend(version_suffix.split('.'))
    
    final_history_tags = []
    for t in raw_history_tags:
        if t and t not in final_history_tags: final_history_tags.append(t)
        
    combined_history_tags = ".".join(final_history_tags)
    ver_tag_history = f".{combined_history_tags}" if combined_history_tags else ""
    
    current_mount = get_mount_root()
        
    if is_movie:
        standard_name = f"{clean_base}.{year}{ver_tag_physical}{ext}" if year else f"{clean_base}{ver_tag_physical}{ext}"
        target_dir = f"{current_mount}/{cat}/{folder}"
    else:
        if ep_num is None:
            await notify_steward_log(f"⚠️ [过滤跳过] `{clean_base}` 未提取到集数(疑似预告)，已过滤。")
            try: await status.edit_text(f"⚠️ `{clean_base}` 未能提取到集数，已被防垃圾机制拦截！(可用 #E集数 遥控器强行指派)")
            except: pass
            return
            
        # 🔥 更新历史拦截查询：使用带有v=的完整B轨道特征码
        if check_history(clean_base, file_season, ep_num, combined_history_tags):
            hist_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag_history}"
            await notify_steward_log(f"🛡️ [账本拦截] `{hist_key}` 账本中已存在，自动销毁重复发车！")
            try: await status.delete()  # 静默拦截，不留残骸
            except: pass
            return
            
        ep_str = f"{ep_num:02d}"
        target_dir = f"{current_mount}/{cat}/{folder}/Season {folder_season}"
        standard_name = f"{clean_base}.S{file_season:02d}E{ep_str}.{year}{ver_tag_physical}{ext}" if year else f"{clean_base}.S{file_season:02d}E{ep_str}{ver_tag_physical}{ext}"
        
    task_lock_key = f"{clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag_history}" if not is_movie else f"{clean_base}{ver_tag_history}"
    
    if task_lock_key in GLOBAL_ACTIVE_LOCKS:
        await notify_steward_log(f"🛡️ [防撞锁拦截] `{task_lock_key}` 正在被施工处理，巡逻哨兵已静默撤退。")
        try:
            await status.delete()
        except: pass
        return
    
    GLOBAL_ACTIVE_LOCKS.add(task_lock_key)
    bg_task_spawned = False
    downloaded_bytes = 0

    try:
        # 🔥 本地防撞补丁：为了防止大通铺撞车，本地暂存文件名强制加上老大的名号
        local_temp_name = f"[{version_suffix}]_{standard_name}" if version_suffix else standard_name
        local_path = os.path.join(LOCAL_TEMP_DIR, local_temp_name)
        
        target_full = f"{target_dir}/{standard_name}".replace("//", "/")
        total_bytes = media.file_size
        file_size_mb = total_bytes / (1024 * 1024) if total_bytes else 0
        
        source_name = message.forward_from_chat.title if message.forward_from_chat and message.forward_from_chat.title else (message.chat.title if message.chat and message.chat.title else "未知来源")
            
        try: await status.edit_text(f"⏳ **[TG下载排队中]** `{standard_name}`\n🚦 正在等待下载通道空闲...")
        except: pass

        async with GLOBAL_TRANSFER_LOCK:
            await notify_steward_log(f"📥 [涡轮启动] 拉取: {standard_name} | 来源: {source_name} | 大小: {file_size_mb:.2f} MB")
            
            try:
                await status.edit_text(f"🚀 **[极速拉取]** `{standard_name}`\n📡 来源频道: **{source_name}**\n⚖️ 机器测重: **{file_size_mb:.2f} MB**\n涡轮进度: **0%**")
            except: pass
            
            chunk_size = 1024 * 1024
            retry_count = 0
            max_retries = 999999  
            last_stuck_chunks = -1
            stuck_loop_count = 0
            
            last_ui_time = time.time()
            last_ui_bytes = 0
            speed_text = "0.00 MB/s"
            eta_text = "计算中..."
            
            while True:
                if retry_count >= max_retries: break
                current_bytes = os.path.getsize(local_path) if os.path.exists(local_path) else 0
                
                if current_bytes > total_bytes:
                    try: os.remove(local_path)
                    except: pass
                    current_bytes = 0
                    
                if current_bytes == total_bytes:
                    downloaded_bytes = current_bytes
                    break
                    
                completed_chunks = current_bytes // chunk_size
                secure_bytes = completed_chunks * chunk_size
                if secure_bytes > 0:
                    try:
                        with open(local_path, "r+b") as f: f.truncate(secure_bytes)
                    except: pass
                    downloaded_bytes = secure_bytes
                    open_mode = "ab"
                else: 
                    downloaded_bytes = 0
                    completed_chunks = 0
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
                        if task_lock_key in GLOBAL_CANCEL_TASKS: break 
                        if writer_error: raise writer_error 
                        await buffer_queue.put(chunk)
                        downloaded_bytes += len(chunk)
                        
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
                                if eta_sec > 60: eta_text = f"{int(eta_sec // 60)}分{int(eta_sec % 60)}秒"
                                else: eta_text = f"{int(eta_sec)}秒"
                            else: eta_text = "卡顿"
                                
                            pct = int(downloaded_bytes * 100 / total_bytes)
                            last_ui_time = now
                            last_ui_bytes = downloaded_bytes
                            
                            try:
                                asyncio.create_task(status.edit_text(
                                    f"🚀 **[极速拉取]** `{standard_name}`\n"
                                    f"📡 来源: **{source_name}** | ⚖️ 大小: **{file_size_mb:.2f} MB**\n"
                                    f"📈 实时网速: **{speed_text}**\n"
                                    f"⏳ 预计剩余: **{eta_text}**\n"
                                    f"⚡ 涡轮进度: **{pct}%**"
                                ))
                            except: pass
                            asyncio.create_task(notify_steward_log(f"🚀 [下载进度] {standard_name} ➔ {pct}% | {speed_text} | 剩余: {eta_text}"))
                    
                    await buffer_queue.put(None)
                    await writer_task

                    if task_lock_key in GLOBAL_CANCEL_TASKS: break 
                    if downloaded_bytes >= total_bytes: break
                    else:
                        retry_count += 1
                        if not yielded_any:
                            try: message = await client.get_messages(message.chat.id, message.id)
                            except: pass

                        if completed_chunks == last_stuck_chunks:
                            stuck_loop_count += 1
                            try: await status.edit_text(f"⚠️ **[节点堵塞]** 发现底层断流，正执行信道唤醒...\n🔄 第 **{retry_count}** 次无损断点死磕...")
                            except: pass
                        else: 
                            last_stuck_chunks = completed_chunks; stuck_loop_count = 0
                        await asyncio.sleep(3.0)
                        
                except Exception as e:
                    if 'writer_task' in locals() and not writer_task.done(): writer_task.cancel()
                    retry_count += 1
                    try: 
                        err_msg = str(e) if str(e) else "底层连接被掐断"
                        await status.edit_text(f"⚠️ **[网络防御生效]** `{standard_name}` 传输受阻！\n🛑 死因: `{err_msg}`\n🛠️ 第 **{retry_count}** 次无缝重联死磕...")
                    except: pass
                    
                    if retry_count % 5 == 0:
                        try: message = await client.get_messages(message.chat.id, message.id)
                        except: pass
                    await asyncio.sleep(3.0)
                    
            if downloaded_bytes < total_bytes: 
                if retry_count >= max_retries or task_lock_key in GLOBAL_CANCEL_TASKS:
                    try:
                        if os.path.exists(local_path): os.remove(local_path)
                        await status.edit_text("❌ 任务已被手动彻底终止或超时，已连根拔起销毁残片。")
                    except: pass
                else:
                    try: await status.edit_text("⚠️ 节点暂存断点，保留残片。")
                    except: pass
                return

        try: await status.edit_text("✅ 本地落盘完成，进入云端上传排队队列 (不阻塞下载)...")
        except: pass

        if ext.lower() in [".cas", ".zip"]:
            import shutil
            import zipfile
            
            STAGING_DIR = "/data/data/com.termux/files/home/sharecas"
            os.makedirs(STAGING_DIR, exist_ok=True)
            final_cas_path = os.path.join(STAGING_DIR, standard_name.replace(ext, ".cas"))
            
            try:
                if ext.lower() == ".zip":
                    with zipfile.ZipFile(local_path, "r") as z:
                        cas_files = [f for f in z.namelist() if f.lower().endswith(".cas")]
                        if cas_files:
                            with open(final_cas_path, "wb") as f_out: 
                                f_out.write(z.read(cas_files[0]))
                else:
                    shutil.move(local_path, final_cas_path)
                
                try: os.remove(local_path) 
                except: pass
                
                try: await status.edit_text(f"🎉 **CAS截留成功**\n已洗名并存入本地 `{STAGING_DIR}`，等待 auto189 接管。")
                except: pass
                return
            except Exception as e:
                try: await status.edit_text(f"❌ CAS处理失败: {e}")
                except: pass
                return
        
        put_url = f"{OLIST_URL}/api/fs/put"
        headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
        push_success, up_retries = False, 0
        f_upload = None
        
        async with GLOBAL_UPLOAD_LOCK:
            while not push_success and up_retries < 5:
                if task_lock_key in GLOBAL_CANCEL_TASKS:
                    try: await status.edit_text("❌ 云端直推已被手动彻底终止。")
                    except: pass
                    break 

                try:
                    last_up = 0
                    f_upload = open(local_path, "rb")
                    async def file_iter():
                        nonlocal last_up
                        sent = 0
                        while True:
                            if task_lock_key in GLOBAL_CANCEL_TASKS: break 
                            chunk = f_upload.read(1024 * 1024)
                            if not chunk: break
                            sent += len(chunk)
                            pct = int(sent * 100 / total_bytes)
                            if (pct // 10) > (last_up // 10):
                                last_up = pct - (pct % 10)
                                if last_up >= 100:
                                    asyncio.create_task(status.edit_text(f"🚀 **[直推天翼云]** `{standard_name}`\n📡 来源: **{source_name}** | ⚖️ 大小: **{file_size_mb:.2f} MB**\n云端进度: **100% (数据推送完毕，正等待天翼云校验落盘...)**"))
                                else:
                                    asyncio.create_task(status.edit_text(f"🚀 **[直推天翼云]** `{standard_name}`\n📡 来源: **{source_name}** | ⚖️ 大小: **{file_size_mb:.2f} MB**\n云端进度: **{last_up}%**"))
                                asyncio.create_task(notify_steward_log(f"☁️ [上传进度] {standard_name} ➔ {last_up}%"))
                            yield chunk
                    
                    async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=900.0, write=60.0, pool=None), trust_env=False) as h: 
                        resp = await h.put(put_url, content=file_iter(), headers=headers)
                    
                    resp_json = resp.json()
                    if resp_json.get("code") == 200: 
                        push_success = True
                    else: 
                        err_msg = resp_json.get("message", "未知拦截")
                        raise Exception(f"云端拒绝落盘: {err_msg}")
                
                except Exception as e: 
                    up_retries += 1
                    err_detail = str(e) if str(e) else "网络响应死锁或彻底超时"
                    try: await status.edit_text(f"⚠️ 云端传输受阻: {err_detail}\n🔄 正在进行第 {up_retries} 次重连...")
                    except: pass
                    await asyncio.sleep(up_retries*4)
                finally:
                    if f_upload:
                        try: f_upload.close()
                        except: pass
                        f_upload = None

            if push_success:
                if not is_movie and ep_num is not None:
                    # 历史记录必须带上完整的B轨道特征码，防止防撞锁失效
                    record_history(clean_base, file_season, ep_num, combined_history_tags)
                    await notify_steward_log(f"📝 [历史已写入] {clean_base}.S{file_season:02d}E{ep_num:02d}{ver_tag_history}")
                    
                await notify_steward_log(f"🎉 [终极入库成功] ➔ {target_full}")
                try: await status.edit_text(f"🎉 **终极入库成功** ➔ `{target_full}`")
                except Exception: 
                    try: await client.send_message("me", f"🎉 **[备用通道通知] 入库成功** ➔ `{standard_name}`")
                    except: pass
                
                try:
                    STAGING_BASE_DIR = "/storage/emulated/0/Download/189cas"
                    cas_file_name = f"{standard_name}.cas"
                    cas_target_full = f"{target_full}.cas"
                    sub_path = target_dir.replace(current_mount, "", 1)
                    local_cas_dir = f"{STAGING_BASE_DIR}{sub_path}"
                    os.makedirs(local_cas_dir, exist_ok=True)
                    final_cas_path = os.path.join(local_cas_dir, cas_file_name)
                    
                    get_info_url = f"{OLIST_URL}/api/fs/get"
                    headers_get = {"Authorization": OLIST_TOKEN, "Content-Type": "application/json"}
                    
                    cas_downloaded = False
                    async with httpx.AsyncClient(timeout=30.0, follow_redirects=True) as client_get:
                        for attempt in range(6):
                            await asyncio.sleep(5) 
                            resp = await client_get.post(get_info_url, json={"path": cas_target_full}, headers=headers_get)
                            resp_data = resp.json()
                            if resp_data.get("code") == 200:
                                raw_url = resp_data["data"]["raw_url"]
                                resp_dl = await client_get.get(raw_url)
                                if resp_dl.status_code == 200:
                                    with open(final_cas_path, "wb") as f_cas:
                                        f_cas.write(resp_dl.content)
                                    await notify_steward_log(f"🔗 [CAS下发] `{sub_path}/{cas_file_name}` 已成功镜像至本地。")
                                    cas_downloaded = True
                                    break
                        if not cas_downloaded:
                            await notify_steward_log(f"⚠️ [CAS拉取超时] 轮询30秒，云端未吐出 `{cas_file_name}`，或直链被拒绝。")
                except Exception as e:
                    await notify_steward_log(f"⚠️ [CAS拉取异常] {str(e)}")
                
                try:
                    if os.path.exists(local_path): os.remove(local_path)
                except Exception: pass

                # =======================================================
                # 🔥 杀青清理逻辑 (Auto-Cleanup)
                # =======================================================
                if src_drama_key and src_end_ep and ep_num is not None:
                    if ep_num >= src_end_ep and src_end_ep != 9999:
                        try:
                            config = load_listener_config()
                            if src_chat_id in config.get("trusted_channels", {}):
                                if src_drama_key in config["trusted_channels"][src_chat_id].get("monitored_dramas", {}):
                                    del config["trusted_channels"][src_chat_id]["monitored_dramas"][src_drama_key]
                                    with open(TG_LISTENER_DB, "w", encoding="utf-8") as f:
                                        json.dump(config, f, ensure_ascii=False, indent=4)
                                    await notify_steward_log(f"🎉 [全季杀青] `{src_drama_key}` 已达到设定的 {src_end_ep} 集，监听雷达已自动销毁！")
                                    try: await client.send_message(COMMAND_CENTER_CHAT, f"🎉 **杀青通知**\n🎬 `{src_drama_key}` 全 {src_end_ep} 集已完结入库！\n♻️ 监听雷达已自动回收。")
                                    except: pass
                        except Exception as e:
                            pass

            else:
                if task_lock_key in GLOBAL_CANCEL_TASKS:
                    return 
                    
                asyncio.create_task(bg_upload_retry_task(local_path, target_full, total_bytes, clean_base, file_season, ep_num, is_movie, standard_name, combined_history_tags))
                bg_task_spawned = True
                try: await status.edit_text("⚠️ 云端直推受阻，已强行托管至后台队列，每10分钟自动重试...")
                except: 
                    try: await client.send_message("me", f"⚠️ 云端直推受阻，已托管至后台重推: `{standard_name}`")
                    except: pass
    finally:
        if not bg_task_spawned:
            GLOBAL_ACTIVE_LOCKS.discard(task_lock_key)
        GLOBAL_CANCEL_TASKS.discard(task_lock_key)

# =================================================================
# 🌟 全自动挖掘引擎 (已挂载让路网关)
# =================================================================
async def sweep_existing_history(client, chat_id, drama_key, category, folder_season, file_season, min_ep, min_mb=0, max_mb=999999, fetch_limit=200, year=None):
    try:
        chat_id_int = int(chat_id) if str(chat_id).lstrip('-').isdigit() else chat_id
        config = load_listener_config()
        d_info = config.get("trusted_channels", {}).get(str(chat_id), {}).get("monitored_dramas", {}).get(drama_key, {})
        search_kw = d_info.get("search_kw", drama_key)
        db_version = d_info.get("version", "")
        db_file_version = d_info.get("file_version", "")
        end_ep = d_info.get("end_ep", 9999)
        
        if not year: year, _ = await fetch_tmdb_details(search_kw)
            
        async for old_msg in client.get_chat_history(chat_id_int, limit=fetch_limit):
            media = old_msg.video or old_msg.document
            if not media: continue
            
            text_to_scan = f"{old_msg.caption or ''} {getattr(media, 'file_name', '')}"
            if old_msg.reply_to_message:
                parent_text = old_msg.reply_to_message.caption or old_msg.reply_to_message.text or ""
                text_to_scan = f"{text_to_scan} {parent_text}"
                
            if search_kw.lower() in text_to_scan.lower():
                # 🔥 网关让路补丁：如果当前订阅没写 v=，但别人写了，并且别人命中该文案，我必须让路跳过！
                skip_due_to_exclusive = False
                if not db_version:
                    all_dramas = config.get("trusted_channels", {}).get(str(chat_id), {}).get("monitored_dramas", {})
                    for other_key, other_info in all_dramas.items():
                        if other_info.get("version") and other_info["version"].lower() in text_to_scan.lower() and other_info.get("search_kw", "").lower() in text_to_scan.lower():
                            skip_due_to_exclusive = True
                            break
                if skip_due_to_exclusive: continue

                ep_num = extract_pure_episode(text_to_scan, drama_anchor=search_kw)
                if ep_num is None or ep_num < min_ep: continue
                
                file_size_mb = media.file_size / (1024 * 1024) if getattr(media, "file_size", 0) else 0
                if not (min_mb <= file_size_mb <= max_mb): continue
                
                auto_tags = extract_media_tags(text_to_scan)
                temp_tags = []
                if auto_tags: temp_tags.extend(auto_tags.split('.'))
                if db_file_version: temp_tags.extend(db_file_version.split('.'))
                if db_version: temp_tags.extend(db_version.split('.'))
                
                preview_tags = ".".join(dict.fromkeys(temp_tags))
                
                folder_name = f"{search_kw} ({year})" if year else search_kw
                if db_version: 
                    folder_name = f"{folder_name} {db_version}"  
                
                if check_history(search_kw, file_season, ep_num, preview_tags):
                    continue
                    
                override_info = (folder_name, category, folder_season, file_season, year, ep_num, db_version, db_file_version, str(chat_id), drama_key, end_ep)
                try: status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[哨兵发车]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
                except Exception: status = await client.send_message("me", f"🎯 **[备用嗅探]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
                asyncio.create_task(process_media_transfer(client, old_msg, status, override_info))
                await asyncio.sleep(5) 
    except Exception as e:
        pass

# =================================================================
# 🧠 智能安全巡航进程 
# =================================================================
async def smart_patrol_daemon(client):
    await asyncio.sleep(60) 
    while True:
        try:
            current_hour = datetime.now().hour
            if 0 <= current_hour < 9:
                await notify_steward_log(f"💤 [夜间休眠] 当前 {current_hour} 点，凌晨0-9点不巡逻，打更人深度睡眠中...")
                await asyncio.sleep(3600) 
                continue
            
            await asyncio.sleep(900)
            
            weekday_map = {0: "周一", 1: "周二", 2: "周三", 3: "周四", 4: "周五", 5: "周六", 6: "周日"}
            today_cn = weekday_map[datetime.now().weekday()]
            
            await notify_steward_log(f"🔍 [定时巡航] 开启全盘清算... 今天是: {today_cn}")
            
            config = load_listener_config()
            channels = config.get("trusted_channels", {})
            if not channels:
                continue
                
            drama_count = 0
            for chat_id, info in channels.items():
                for drama_name, d_info in info.get("monitored_dramas", {}).items():
                    
                    freq = d_info.get("frequency", "日更")
                    if freq != "日更" and freq != today_cn:
                        continue
                    
                    folder_s = d_info.get("folder_season", 1)
                    file_s = d_info.get("file_season", folder_s)
                    min_ep = d_info.get("min_ep", 1)
                    cat = d_info.get("category", "未分类")
                    min_mb = d_info.get("min_mb", 0)
                    max_mb = d_info.get("max_mb", 999999)
                    
                    d_year = d_info.get("year")
                    asyncio.create_task(sweep_existing_history(client, chat_id, drama_name, cat, folder_s, file_s, min_ep, min_mb, max_mb, fetch_limit=10, year=d_year))
                    drama_count += 1
            
            if drama_count > 0:
                await notify_steward_log(f"✅ [巡航分流完毕] 今天符合更新条件的剧集共 {drama_count} 部，已派出哨兵翻找。")
            else:
                await notify_steward_log(f"☕ [巡航结束] 今天没有任何剧集达到更新排班，哨兵原地喝茶。")
                
        except Exception as e:
            await notify_steward_log(f"❌ [巡航突发异常] 详情: {str(e)}", level="ERROR")
            await asyncio.sleep(60)

# =================================================================
# 🧠 开机自动补漏清算
# =================================================================
async def startup_catchup_sweep(client):
    await asyncio.sleep(5)
    try:
        config = load_listener_config()
        await notify_steward_log("🚀 [系统启动] 正在进行开机全自动历史补漏扫荡...")
        for chat_id, info in config.get("trusted_channels", {}).items():
            for drama_name, d_info in info.get("monitored_dramas", {}).items():
                folder_s = d_info.get("folder_season", 1)
                file_s = d_info.get("file_season", folder_s)
                min_ep = d_info.get("min_ep", 1)
                cat = d_info.get("category", "未分类")
                min_mb = d_info.get("min_mb", 0)
                max_mb = d_info.get("max_mb", 999999)
                
                d_year = d_info.get("year")
                await sweep_existing_history(client, chat_id, drama_name, cat, folder_s, file_s, min_ep, min_mb, max_mb, fetch_limit=200, year=d_year)
                await asyncio.sleep(2)
        await notify_steward_log("✅ [开机扫荡完成] 历史遗留清点完毕。")
    except Exception:
        pass

# =================================================================
# 📡 远程指令总枢
# =================================================================
STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]

@app.on_message(filters.command(["sub", "unsub", "list", "add", "del", "go", "history", "ping", "rm", "clean", "scan", "rmh", "setdir", "cancel"]) & filters.user("me"))
async def manage_system_commands(client, message):
    command = message.command[0].lower()
    config = load_listener_config()
    
    if command == "ping":
        uptime_minutes = int((time.time() - START_TIME) / 60)
        await message.reply_text(f"🟢 **系统健康度 [优秀]**\n⏱️ 存活: `{uptime_minutes}` 分钟")
        return

    if command == "setdir":
        if len(message.command) < 2:
            return await message.reply_text(f"📁 当前上传目录: `{get_mount_root()}`\n⚠️ 语法：`/setdir [新目录路径]` (例如: `/setdir /family/188_cas`)")
        new_dir = message.command[1]
        set_mount_root(new_dir)
        return await message.reply_text(f"✅ 上传主目录已成功切换至: `{new_dir}`")

    if command == "clean":
        count = 0
        if os.path.exists(LOCAL_TEMP_DIR):
            for f in os.listdir(LOCAL_TEMP_DIR):
                try: os.remove(os.path.join(LOCAL_TEMP_DIR, f)); count += 1
                except: pass
        await message.reply_text(f"🧹 抹除 `{count}` 个遗留碎片！")
        return

    if command == "cancel":
        if len(message.command) < 2: 
            return await message.reply_text("⚠️ 语法：`/cancel [关键字]` 或 `/cancel all` (无差别终止所有任务)")
            
        kw = " ".join(message.command[1:]).lower()
        
        if kw == "all":
            if not GLOBAL_ACTIVE_LOCKS:
                return await message.reply_text("📭 当前大盘很干净，没有任何运行中的任务。")
            matched = list(GLOBAL_ACTIVE_LOCKS)
        else:
            kws = kw.split()
            matched = [name for name in GLOBAL_ACTIVE_LOCKS if all(k in name.lower() for k in kws)]
            
        if not matched: 
            return await message.reply_text(f"❌ 当前运行的任务中，没找到包含 `{kw}` 的。")
            
        for name in matched:
            GLOBAL_CANCEL_TASKS.add(name)
            
        return await message.reply_text(f"🛑 已下达最高追杀令，立刻强杀以下任务：\n" + "\n".join([f"`{name}`" for name in matched]))

    if command == "rm":
        if len(message.command) < 2: return await message.reply_text("⚠️ 语法：`/rm [航线剧名关键字]`")
        kw = " ".join(message.command[1:]).lower()
        routes = load_tg_routes()
        matched_keys = [k for k in routes.keys() if kw in k.lower()]
        if not matched_keys: return await message.reply_text(f"❌ 航线库里没找到包含 `{kw}` 的记录")
        for k in matched_keys: del routes[k]
        save_tg_routes(routes)
        return await message.reply_text(f"🗑️ 已彻底删除 {len(matched_keys)} 条自定义航线！")

    if command == "rmh":
        if len(message.command) < 2: return await message.reply_text("⚠️ `/rmh [剧名关键字]`")
        kw = " ".join(message.command[1:]).lower()
        history = load_history()
        matched_keys = [k for k in history.keys() if kw in k.lower()]
        if not matched_keys: return await message.reply_text(f"❌ 历史库里没找到包含 `{kw}` 的记录")
        for k in matched_keys: del history[k]
        with open(TG_HISTORY_DB, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
        return await message.reply_text(f"🗑️ 已从历史账本中彻底抹除 {len(matched_keys)} 条记录！")

    if command == "history":
        history = load_history()
        if not history: return await message.reply_text("📭 历史账本目前是空的。")
        sorted_hist = sorted(history.items(), key=lambda x: x[1], reverse=True)[:15]
        msg_text = "📜 **[最近入库成功记录 (前15条)]**\n\n"
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
        await message.reply_text(f"🏢 **频道挂载成功**: {c_name} (`{c_id}`)")
        return

    if command == "del":
        if len(message.command) < 2: return
        if message.command[1] in config["trusted_channels"]:
            del config["trusted_channels"][message.command[1]]
            with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
            await message.reply_text("🗑️ 频道已拔除。")
        return

    if command == "list":
        try:
            msg_text = "📡 **[雷达大盘状态]**\n\n"
            for chat_id, info in config.get("trusted_channels", {}).items():
                msg_text += f"🏢 **频道**: {info.get('channel_name', chat_id)}\n"
                dramas = info.get("monitored_dramas", {})
                if not dramas: 
                    msg_text += "  └ 📭 (空)\n"
                else:
                    for d_name, d_info in dramas.items(): 
                        f_season = d_info.get('file_season', 1)
                        try: f_season = int(f_season)
                        except: f_season = 1
                        
                        min_mb = d_info.get('min_mb', 0)
                        max_mb = d_info.get('max_mb', 999999)
                        freq = d_info.get('frequency', '日更')
                        end_ep = d_info.get('end_ep', 9999)
                        
                        search_kw = d_info.get('search_kw', d_name)
                        ver = d_info.get('version', '')
                        
                        s_text = f"S{f_season:02d}"
                        size_txt = f" / {min_mb}-{max_mb if max_mb != 999999 else '上限'}MB" if (min_mb or max_mb != 999999) else ""
                        freq_txt = f" / ⏱️{freq}"
                        ver_txt = f" / 🏷️{ver}" if ver else ""
                        end_txt = f" / 🏁{end_ep}集杀青" if end_ep != 9999 else " / ♾️连载"
                        
                        line = f"  └ 🎬 `{search_kw}` ({s_text}{size_txt}{freq_txt}{end_txt}{ver_txt})\n"
                        
                        if len(msg_text) + len(line) > 4000:
                            await message.reply_text(msg_text)
                            msg_text = ""
                        msg_text += line
            
            if msg_text:
                await message.reply_text(msg_text)
                
        except Exception as e:
            await message.reply_text(f"❌ `/list` 读取失败，原因: {str(e)}")
        return

    if command == "scan":
        args = message.command[1:]
        target_kw = " ".join(args).lower() if args else None
        await message.reply_text("🔍 **[手动扫荡触发]** ➔ 正在全自动翻找目标漏网之鱼...")
        for chat_id, info in config.get("trusted_channels", {}).items():
            for drama_name, d_info in info.get("monitored_dramas", {}).items():
                if target_kw and target_kw not in drama_name.lower(): continue
                folder_s = d_info.get("folder_season", 1)
                file_s = d_info.get("file_season", folder_s)
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
        cn_weekday_map = {"星期一": "周一", "星期二": "周二", "星期三": "周三", "星期四": "周四", "星期五": "周五", "星期六": "周六", "星期日": "周日"}
        frequency = "日更"
        for i, arg in enumerate(args):
            if arg in freq_list: frequency = args.pop(i); break
            elif arg in cn_weekday_map: frequency = cn_weekday_map[args.pop(i)]; break
        
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
        
        if not target_pools:
            return await message.reply_text("⚠️ 你的大盘里还没有任何频道！请先转发一条频道的视频，并对它【回复】 /sub 指令来自动添加！")

        custom_year = next((args.pop(i) for i, arg in enumerate(args) if arg.isdigit() and len(arg) == 4 and (1900 < int(arg) < 2100)), None)

        version_suffix = ""
        file_suffix = ""
        end_ep = None
        
        for i in range(len(args)-1, -1, -1):
            if args[i].lower().startswith("v="):
                version_suffix = args.pop(i)[2:]
            elif args[i].lower().startswith("f="):
                file_suffix = args.pop(i)[2:]
            elif args[i].lower().startswith("end="):
                end_ep = int(args.pop(i)[4:])

        file_season = int(args.pop(-1)[1:]) if args and re.match(r'^s\d+$', args[-1], re.IGNORECASE) else None
        min_ep = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        folder_season = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        if file_season is None: file_season = folder_season
        
        drama_name = " ".join(args) if args else "未知目标"
        drama_key = f"{drama_name}_{version_suffix}" if version_suffix else drama_name
        
        # 🔥 TMDB 自动补全年份与杀青集数
        fetched_year, fetched_end = await fetch_tmdb_details(drama_name)
        if not custom_year: custom_year = fetched_year
        if end_ep is None:
            if category in ["国漫", "日漫", "综艺"]:
                end_ep = 9999 # 动漫和综艺自带免死金牌，永远不杀青
            else:
                end_ep = fetched_end
                
        target_pools = [specific_channel] if specific_channel else list(config["trusted_channels"].keys())
        
        for chat_id in target_pools:
            if "monitored_dramas" not in config["trusted_channels"][chat_id]: config["trusted_channels"][chat_id]["monitored_dramas"] = {}
            config["trusted_channels"][chat_id]["monitored_dramas"][drama_key] = {
                "search_kw": drama_name,  
                "version": version_suffix, 
                "file_version": file_suffix, 
                "category": category, "folder_season": folder_season, 
                "file_season": file_season, "min_ep": min_ep,
                "min_mb": min_mb, "max_mb": max_mb,
                "frequency": frequency,
                "year": custom_year,
                "end_ep": end_ep
            }
            
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        end_display = "♾️无限连载" if end_ep == 9999 else f"{end_ep} 集杀青"
        await message.reply_text(f"✅ **订阅成功**: `{drama_name}`\n🏷️ **版本追踪**: `{version_suffix or '默认雷达嗅探'}`\n⚖️ **画质锁定**: `{min_mb}-{max_mb} MB`\n⏱️ **频率**: `{frequency}`\n📅 **年份**: `{custom_year}`\n🏁 **完结设置**: `{end_display}`\n🚀 **启动扫荡！**")
        
        for chat_id in target_pools:
            asyncio.create_task(sweep_existing_history(client, chat_id, drama_key, category, folder_season, file_season, min_ep, min_mb, max_mb, fetch_limit=200, year=custom_year))
        return
    
    if command in ["unsub", "del"]:
        args = message.command[1:]
        if not args: 
            return await message.reply_text("⚠️ 语法：`/unsub [剧名] [可选: v=版本号] [可选:-100开头的频道ID]`")
        
        specific_channel = next((arg for arg in args if arg.startswith("-100")), None)
        if specific_channel:
            args.remove(specific_channel)
            
        version_suffix = ""
        for i, arg in enumerate(args):
            if arg.lower().startswith("v="):
                version_suffix = args.pop(i)[2:]
                break
                
        drama_name = " ".join(args)
        if not drama_name:
            return await message.reply_text("⚠️ 请输入要取消的剧名！")
            
        drama_key = f"{drama_name}_{version_suffix}" if version_suffix else drama_name
        
        config = load_listener_config()
        channels = config.get("trusted_channels", {})
        
        deleted_count = 0
        target_pools = [specific_channel] if specific_channel else list(channels.keys())
        
        for chat_id in target_pools:
            if chat_id in channels and "monitored_dramas" in channels[chat_id]:
                if drama_key in channels[chat_id]["monitored_dramas"]:
                    del channels[chat_id]["monitored_dramas"][drama_key]
                    deleted_count += 1
                elif drama_name in channels[chat_id]["monitored_dramas"]:
                    del channels[chat_id]["monitored_dramas"][drama_name]
                    deleted_count += 1
                    
        if deleted_count > 0:
            with open(TG_LISTENER_DB, "w", encoding="utf-8") as f:
                json.dump(config, f, ensure_ascii=False, indent=4)
            await message.reply_text(f"✅ **精确打击成功**\n💣 目标: `{drama_key}`\n📉 共从 **{deleted_count}** 个频道中移除了该监听雷达！")
        else:
            await message.reply_text(f"⚠️ **未找到目标**: `{drama_key}`\n💡 请使用 `/list` 命令查看雷达大盘里的准确名称（包含版本）。")
        return

    if command == "go":
        args = message.command[1:]
        if not args: return await message.reply_text("⚠️ 语法：`/go [剧名关键字] [可选:分类/年份/v=版本/季数]`")
        routes = load_tg_routes()
        
        version_suffix = ""
        file_suffix = ""
        for i in range(len(args)-1, -1, -1):
            if args[i].lower().startswith("v="):
                version_suffix = args.pop(i)[2:]
            elif args[i].lower().startswith("f="):
                file_suffix = args.pop(i)[2:]

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
                return await message.reply_text(f"✅ 命中记忆航线！已锁定目标: `{matched_item['folder_name']}`\n请直接转发发车！")
            return await message.reply_text("⚠️ 未匹配到历史，如果要开辟新线请提供标准分类。")
        
        cat = args.pop(cat_idx)
        
        file_season = int(args.pop(-1)[1:]) if args and re.match(r'^s\d+$', args[-1], re.IGNORECASE) else None
        
        folder_season = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        
        if file_season is None: file_season = folder_season 
        
        pure_title = " ".join(args)
        year, _ = await fetch_tmdb_details(pure_title)
        if custom_year: year = custom_year
        
        folder_name = f"{pure_title} ({year})" if year else pure_title
        if version_suffix: 
            folder_name = f"{folder_name} {version_suffix}"
        
        routes[folder_name] = {"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "version": version_suffix, "file_version": file_suffix, "created_at": int(time.time())}
        save_tg_routes(routes)
        
        GLOBAL_ROUTE_CACHE.update({"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "version": version_suffix, "file_version": file_suffix, "expire_time": time.time() + 360, "manual_ep": None})
        
        await message.reply_text(f"✅ 新航线已打通：\n📁 目标锁定: `{folder_name}`\n请直接转发媒体发车！")
        return
    
    if command in ["help", "h"]:
        try:
            help_text = """🤖 **打更人媒体管家 - 终极参数指南** 🤖

🎬 **1. 自动订阅 (/sub)**
💬 **语法**: `/sub [剧名] [分类] [最小-最大MB] [频率] [年份] [v=版本] [f=参数] [end=完结集] [S文件季]`
👉 **示例**: `/sub 将夜 国漫 400-1500 周四 end=36 v=霸王龙 f=H265`

🗑️ **2. 取消订阅 (/unsub)**
💬 **语法**: `/unsub [剧名] [v=版本] [-100频道ID]`
👉 **示例**: `/unsub 将夜 v=霸王龙` (精准取消此压制组)

🚀 **3. 手动发车 (/go)**
💬 **语法**: `/go [剧名] [分类] [年份] [v=版本] [文件夹季]`
👉 **步骤1**: `/go 绝命毒师 欧美剧 2018 v=4K 1` (锁死文件夹)
👉 **步骤2**: 发送 `#S2E8` (随意捏造文件名，不破坏文件夹)
👉 **步骤3**: 狂发视频，自动重命名入库！

📡 **4. 大盘与扫荡**
👉 `/list` : 查看各频道监听排班表，以及何时杀青。
👉 `/scan` : 强行触发一次全网补漏扫荡

❌ **5. 删除记录**
👉 `/rmh` : 删除下载历史记录
👉 `/rm` : 删除历史航线！把不再需要的死档永久抹除"""
            
            await message.reply_text(help_text)
        except Exception as e:
            await message.reply_text(f"❌ 帮助菜单解析失败，原因: {str(e)}")
        return

# =================================================================
# 🎛️ 航线遥控器：手动霸权重塑文件名
# =================================================================
@app.on_message(filters.text & ~filters.media & filters.user("me"))
async def override_episode(client, message):
    text = message.text.strip().upper()
    if text.startswith("#") and GLOBAL_ROUTE_CACHE.get("expire_time", 0) > time.time():
        m_se = re.search(r'^#S(\d+)E(\d+)', text)
        m_s = re.search(r'^#S(\d+)$', text)
        m_e = re.search(r'^#E(\d+)$', text)
        
        reply_msg = ""
        if m_se:
            GLOBAL_ROUTE_CACHE["file_season"] = int(m_se.group(1))
            GLOBAL_ROUTE_CACHE["manual_ep"] = int(m_se.group(2))
            reply_msg = f"🎛️ **文件名重塑**: 季数 ➔ S{int(m_se.group(1)):02d} | 集数 ➔ E{int(m_se.group(2)):02d}\n🔒 (物理文件夹维持原位不变)"
        elif m_s:
            GLOBAL_ROUTE_CACHE["file_season"] = int(m_s.group(1))
            reply_msg = f"🎛️ **文件名重塑**: 季数 ➔ S{int(m_s.group(1)):02d}\n🔒 (物理文件夹维持原位不变)"
        elif m_e:
            GLOBAL_ROUTE_CACHE["manual_ep"] = int(m_e.group(1))
            reply_msg = f"🎛️ **文件名重塑**: 集数 ➔ E{int(m_e.group(1)):02d}"
            
        if reply_msg:
            await message.reply_text(reply_msg)

# =================================================================
# 🎯 被动网关双保险拦截 (已挂载让路网关)
# =================================================================
@app.on_message(filters.video | filters.document)
@app.on_edited_message(filters.video | filters.document)
async def media_routing_gateway(client, message):
    config = load_listener_config()
    
    chat_id_to_check = str(message.chat.id) if message.chat else ""
    original_channel_id = ""
    parent_text = ""
    
    if message.reply_to_message:
        parent = message.reply_to_message
        parent_text = parent.caption or parent.text or ""
        
        if parent.forward_from_chat:
            original_channel_id = str(parent.forward_from_chat.id)
        elif parent.sender_chat:
            original_channel_id = str(parent.sender_chat.id) 
            
        if parent.reply_to_message:
            parent_text += f" {parent.reply_to_message.caption or parent.reply_to_message.text or ''}"

    matched_channel = None
    for k in config.get("trusted_channels", {}):
        if chat_id_to_check and (k in chat_id_to_check or chat_id_to_check in k):
            matched_channel = k
            break
        if original_channel_id and (k in original_channel_id or original_channel_id in k):
            matched_channel = k
            break

    if matched_channel:
        channel_info = config["trusted_channels"][matched_channel]
        media = message.video or message.document
        
        text_to_scan = f"{message.caption or ''} {getattr(media, 'file_name', '')} {parent_text}"

        matched_routes = []
        for drama_key, route_info in channel_info.get("monitored_dramas", {}).items():
            search_kw = route_info.get("search_kw", drama_key)
            if search_kw.lower() in text_to_scan.lower():
                matched_routes.append((drama_key, route_info))
                
        if not matched_routes: return
        
        # 🔥 网关让路补丁：如果有带v=的订阅命中了关键词，普通订阅必须让路！
        best_match = None
        for key, route in matched_routes:
            db_ver = route.get("version", "")
            if db_ver and db_ver.lower() in text_to_scan.lower():
                best_match = (key, route)
                break
                
        if not best_match:
            for key, route in matched_routes:
                if not route.get("version", ""):
                    best_match = (key, route)
                    break
                    
        if not best_match: return
        drama_key, route_info = best_match

        db_version = route_info.get("version", "")
        db_file_version = route_info.get("file_version", "") 
        end_ep = route_info.get("end_ep", 9999)
        search_kw = route_info.get("search_kw", drama_key)
        
        ep_num = extract_pure_episode(text_to_scan, drama_anchor=search_kw)
        if ep_num is not None and ep_num < route_info.get("min_ep", 1): return 
        
        min_mb = route_info.get("min_mb", 0)
        max_mb = route_info.get("max_mb", 999999)
        file_size_mb = media.file_size / (1024 * 1024) if getattr(media, "file_size", 0) else 0
        
        if not (min_mb <= file_size_mb <= max_mb):
            asyncio.create_task(notify_steward_log(f"🚫 [画质拦截] {search_kw} E{ep_num} | 大小: {file_size_mb:.1f}MB", level="WARNING"))
            return 
        
        auto_tags = extract_media_tags(text_to_scan)
        temp_tags = []
        if auto_tags: temp_tags.extend(auto_tags.split('.'))
        if db_file_version: temp_tags.extend(db_file_version.split('.'))
        if db_version: temp_tags.extend(db_version.split('.'))
        
        preview_tags = ".".join(dict.fromkeys(temp_tags))
            
        folder_season = route_info.get("folder_season", 1)
        file_season = route_info.get("file_season", folder_season)
        year = route_info.get("year")
        if not year: year, _ = await fetch_tmdb_details(search_kw)
        
        folder_name = f"{search_kw} ({year})" if year else search_kw
        if db_version: 
            folder_name = f"{folder_name} {db_version}"
        
        if ep_num is not None and check_history(search_kw, file_season, ep_num, preview_tags): 
            return
        
        override_info = (folder_name, route_info["category"], folder_season, file_season, year, ep_num, db_version, db_file_version, matched_channel, drama_key, end_ep)
        try:
            status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[实时发车]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
        except Exception:
            status = await client.send_message("me", f"🎯 **[备用发车]**\n📺 `{search_kw}` ({db_version or '默认'}) ➔ S{file_season:02d}E{ep_num:02d}")
        asyncio.create_task(process_media_transfer(client, message, status, override_info))
        return

    if message.chat and message.chat.type in [enums.ChatType.PRIVATE, enums.ChatType.BOT]:
        if time.time() > GLOBAL_ROUTE_CACHE["expire_time"] or not GLOBAL_ROUTE_CACHE["folder_name"]:
            return
        try: status = await message.reply_text("⚡ 转发航线认证通过，正向引流拉取...")
        except: return
        asyncio.create_task(process_media_transfer(client, message, status))

# =================================================================
# 🚀 引擎启动 
# =================================================================
if __name__ == "__main__":
    clean_orphan_temp_files(max_age_hours=24) 
    load_listener_config()
    
    async def start_system():
        await app.start()
        asyncio.create_task(startup_catchup_sweep(app))
        asyncio.create_task(smart_patrol_daemon(app))
        try:
            await app.send_message(COMMAND_CENTER_CHAT, "🤖 TG下载189上传引擎启动啦！")
            await notify_steward_log("✅ TG下载189上传引擎启动啦！")
        except Exception: pass
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
