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

### 三、autotg.py手动版

```
import os
import re
import time
import json
import asyncio
from datetime import datetime
from urllib.parse import quote
import httpx
from pyrogram import Client, filters
from pyrogram.types import InlineKeyboardMarkup, InlineKeyboardButton
import logging

# =================================================================
# ⚙️ 核心网关、路径与凭证配置区域
# =================================================================
API_ID = 33349348             
API_HASH = "44bde7f01d2b6001589c28cea93716af"     
BOT_TOKEN = "8235305939:AAEg0ICkxUSwRPrg0FlXb4o89In8WUKdM3Y" 

OLIST_URL = "http://127.0.0.1:5244"
OLIST_TOKEN = "openlist-a87614da-32dd-4b80-9150-6447de823da8f33x53ymkrx0aPKG0HUcsFHmjFRYTKFhSADLRhoQLkXa7ogaiByhWRNEXCjpblp9" 

STEWARD_BASE_URL = "http://127.0.0.1:5000"
TARGET_MOUNT_ROOT = "/family/177_cas"

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
TG_ROUTE_DB = os.path.join(BASE_DIR, "tg_manual_routes.json")
LOCAL_TEMP_DIR = os.path.join(BASE_DIR, "tg_temp")
os.makedirs(LOCAL_TEMP_DIR, exist_ok=True)

TMDB_API_KEY = "9c88e18e43543c8ff195c631aaa0d2fa" 

# =================================================================
# 🤫 日志与上报中枢配置
# =================================================================
logging.basicConfig(level=logging.INFO, format='[%(asctime)s] %(message)s', datefmt='%H:%M:%S')
logging.getLogger("pyrogram").setLevel(logging.WARNING)
logging.getLogger("httpx").setLevel(logging.WARNING)

logger = logging.getLogger("TGEngine")
app = Client("tg_robust_leecher_v13", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

def clean_orphan_temp_files():
    msg = "🧹 [引擎自检] 正在全盘清扫专属池历史残留碎片..."
    logger.info(msg)
    try: httpx.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": "INFO", "msg": msg}, timeout=1.0)
    except Exception: pass
    
    if os.path.exists(LOCAL_TEMP_DIR):
        cleaned_count = 0
        for f in os.listdir(LOCAL_TEMP_DIR):
            file_path = os.path.join(LOCAL_TEMP_DIR, f)
            try:
                if os.path.isfile(file_path):
                    os.remove(file_path)
                    cleaned_count += 1
            except Exception: pass
            
        if cleaned_count > 0:
            done_msg = f"✨ [引擎自检] 极致净化完成！共物理超度 {cleaned_count} 个断流碎片文件。"
            logger.info(done_msg)
            try: httpx.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": "INFO", "msg": done_msg}, timeout=1.0)
            except Exception: pass

def load_tg_routes():
    if os.path.exists(TG_ROUTE_DB):
        try:
            with open(TG_ROUTE_DB, 'r', encoding='utf-8') as f: return json.load(f)
        except Exception: pass
    return {}

def save_tg_routes(data):
    try:
        with open(TG_ROUTE_DB, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e: logger.error(f"保存路由库失败: {e}")

GLOBAL_ROUTE_CACHE = {"folder_name": "", "category": "", "season": 1, "year": "", "expire_time": 0, "manual_ep": None}
steward_states = {}

async def notify_steward_log(msg, level="INFO"):
    logger.info(msg)
    try:
        async with httpx.AsyncClient(timeout=2.0) as client:
            await client.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": level, "msg": msg})
    except Exception: pass

async def fetch_tmdb_year(cn_title):
    if not TMDB_API_KEY: return datetime.now().strftime("%Y")
    clean_q = re.sub(r'S\d+$|\s+\d+$', '', cn_title).strip()
    url = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={quote(clean_q)}&page=1"
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            res = await client.get(url)
            results = res.json().get("results")
            if results:
                raw_d = results[0].get("first_air_date") or results[0].get("release_date") or ""
                if len(raw_d) >= 4: return raw_d[:4]
    except Exception: pass
    return datetime.now().strftime("%Y")

# =================================================================
# 🧠 终极兼容集数提取器 (五级防御链)
# =================================================================
def extract_episode(search_text):
    text = search_text.replace("_", ".").replace(" ", ".")
    
    for p in [r'(?i)S\d+E(\d+)', r'第\d+[季部].*?第\s*(\d+)\s*[集话期]', r'(?<!\d)(\d{1,2})x(\d+)(?!\d)']:
        m = re.search(p, text)
        if m: return int(m.group(2) if 'x' in p else m.group(1))
            
    m_mixed = re.search(r'第[一二三四五六七八九十零百]+[集话期]\.?(\d+)', text)
    if m_mixed:
        val = int(m_mixed.group(1))
        if not (1900 < val < 2100): return val

    for p in [r'(?i)(?:EP|E)\.?(\d+)', r'第\s*(\d+)\s*[集话期]', r'(?<=\.\s)-?\s*(\d+)(?=\.)', r'(?<=\[)(\d+)(?=\])']:
        m = re.search(p, text)
        if m:
            val = int(m.group(1))
            if 1900 < val < 2100: continue 
            return val
            
    m_cn = re.search(r'第([一二三四五六七八九十零百]+)[集话期]', text)
    if m_cn:
        cn_str = m_cn.group(1)
        cn_map = {"一":1, "二":2, "三":3, "四":4, "五":5, "六":6, "七":7, "八":8, "九":9, "十":10}
        if cn_str in cn_map: return cn_map[cn_str]
        if cn_str.startswith("十") and len(cn_str) == 2: return 10 + cn_map.get(cn_str[1], 0)
        if len(cn_str) == 3 and cn_str[1] == "十": return cn_map.get(cn_str[0],0)*10 + cn_map.get(cn_str[2],0)
        if cn_str.endswith("十") and len(cn_str) == 2: return cn_map.get(cn_str[0],0)*10

    bare_match = re.search(r'^(\d+)(?:\.|cut|_)|(\d+)\.mp4', text, re.IGNORECASE)
    if bare_match:
        val = int(bare_match.group(1) or bare_match.group(2))
        if not (1900 < val < 2100): return val
        
    return None

# =================================================================
# 📱 移动端 12 大品类全栖交互系统
# =================================================================
@app.on_message(filters.command(["start", "menu"]) | filters.regex(r'^(?:发车|菜单|车票)$'))
async def display_control_panel(client, message):
    kb = InlineKeyboardMarkup([
        [InlineKeyboardButton("🚀 临时发车 / 新剧秒建", callback_data="tg_new_route")],
        [InlineKeyboardButton("📋 历史航线直达", callback_data="tg_history")],
        [InlineKeyboardButton("⚙️ 管理/删除航线", callback_data="tg_manage")],
        [InlineKeyboardButton("❌ 关闭面板", callback_data="tg_close")]
    ])
    await message.reply_text(f"🤖 **[工业级自愈控流台 V13]**\n物理挂载基点：`{TARGET_MOUNT_ROOT}`", reply_markup=kb)

@app.on_callback_query()
async def handle_inline_callbacks(client, callback_query):
    data = callback_query.data
    chat_id = callback_query.message.chat.id
    
    if data == "tg_close":
        await callback_query.message.edit_text("🎯 **操控面板已收起。**")
        return
    
    if data == "tg_back_main":
        steward_states.pop(chat_id, None)
        kb = InlineKeyboardMarkup([
            [InlineKeyboardButton("🚀 临时发车 / 新剧秒建", callback_data="tg_new_route")],
            [InlineKeyboardButton("📋 历史航线直达", callback_data="tg_history")],
            [InlineKeyboardButton("⚙️ 管理/删除航线", callback_data="tg_manage")],
            [InlineKeyboardButton("❌ 关闭面板", callback_data="tg_close")]
        ])
        await callback_query.message.edit_text("👇 请选择对接大门：", reply_markup=kb)

    if data == "tg_new_route":
        cats = [
            "华语剧", "欧美剧", "日韩剧", "短剧",
            "华语电影", "欧美电影", "日韩电影", "演唱会",
            "国漫", "日漫", "综艺", "纪录片"
        ]
        btns = []
        for i in range(0, len(cats), 2):
            btns.append([InlineKeyboardButton(cats[i], callback_data=f"tg_cat_{cats[i]}"), InlineKeyboardButton(cats[i+1], callback_data=f"tg_cat_{cats[i+1]}")])
        btns.append([InlineKeyboardButton("⬅️ 返回上级", callback_data="tg_back_main")])
        await callback_query.message.edit_text("📂 请指定新剧集归属【全栖品类】：", reply_markup=InlineKeyboardMarkup(btns))

    elif data.startswith("tg_cat_"):
        cat = data.split("_")[2]
        steward_states[chat_id] = {"step": "await_title", "cat": cat}
        await callback_query.message.edit_text(f"✅ 品类锁定：{cat}\n✏️ **请输入【纯净影视名】**：\n(提示：发纯文本即可，后台自动查档)")

    elif data == "tg_history" or data == "tg_manage":
        is_del = (data == "tg_manage")
        routes = load_tg_routes()
        if not routes:
            await callback_query.answer("📭 当前路由账本无留存记录。", show_alert=True)
            return
        
        sorted_items = sorted(routes.values(), key=lambda x: x.get("created_at", 0), reverse=True)[:20]
        btns = []
        prefix = "🗑️ 抹除 | " if is_del else "🚀 "
        head = "tg_del_" if is_del else "tg_use_"
        
        for idx, item in enumerate(sorted_items):
            btns.append([InlineKeyboardButton(f"{prefix}{item['category']} | {item['folder_name']}", callback_data=f"{head}{idx}")])
        
        btns.append([InlineKeyboardButton("⬅️ 返回主菜单", callback_data="tg_back_main")])
        title = "🗑️ **[注销模式]** 点击剧名永久抹除：" if is_del else "📋 **[复用模式]** 点击重置路由指针："
        await callback_query.message.edit_text(title, reply_markup=InlineKeyboardMarkup(btns))

    elif data.startswith("tg_use_"):
        idx = int(data.split("_")[2])
        routes = load_tg_routes()
        item = sorted(routes.values(), key=lambda x: x.get("created_at", 0), reverse=True)[idx]
        global GLOBAL_ROUTE_CACHE
        GLOBAL_ROUTE_CACHE.update({"folder_name": item["folder_name"], "category": item["category"], "season": item.get("season", 1), "year": item.get("year", ""), "expire_time": time.time() + 600, "manual_ep": None})
        s_str = "" if "电影" in item["category"] or item["category"] in ["演唱会", "纪录片"] else f"/Season {GLOBAL_ROUTE_CACHE['season']}"
        await callback_query.message.edit_text(f"🎉 **旧档路由已映射 (10分钟有效)**\n🎬 `{item['folder_name']}`\n📁 `{TARGET_MOUNT_ROOT}/{item['category']}/{item['folder_name']}{s_str}`\n\n👉 **通道已畅通，请直接转发源视频！**")

    elif data.startswith("tg_del_"):
        idx = int(data.split("_")[2])
        routes = load_tg_routes()
        target_name = sorted(routes.values(), key=lambda x: x.get("created_at", 0), reverse=True)[idx]["folder_name"]
        if target_name in routes:
            del routes[target_name]
            save_tg_routes(routes)
            await callback_query.answer(f"✅ 成功抹除档案：{target_name}", show_alert=True)
            await handle_inline_callbacks(client, callback_query)

@app.on_message(filters.text & ~filters.media & ~filters.bot, group=-1)
async def process_text_inputs(client, message):
    chat_id = message.chat.id
    text = message.text.strip()
    
    if chat_id in steward_states and steward_states[chat_id].get("step") == "await_title":
        cat = steward_states[chat_id]["cat"]
        steward_states.pop(chat_id, None)
        status = await message.reply_text("🔍 正在校准 TMDB 智库锚点...")
        
        season_num = 1
        s_match = re.search(r'\s+[sS第]?0?(\d{1,2})[季]?$', text)
        if s_match:
            season_num = int(s_match.group(1))
            pure_title = text[:s_match.start()].strip()
        else: pure_title = text
            
        year = await fetch_tmdb_year(pure_title)
        folder_name = pure_title if re.search(r'\(\d{4}\)', pure_title) else f"{pure_title} ({year})"
            
        routes = load_tg_routes()
        routes[folder_name] = {"folder_name": folder_name, "category": cat, "season": season_num, "year": year, "created_at": int(time.time())}
        save_tg_routes(routes)
        
        global GLOBAL_ROUTE_CACHE
        GLOBAL_ROUTE_CACHE.update({"folder_name": folder_name, "category": cat, "season": season_num, "year": year, "expire_time": time.time() + 600, "manual_ep": None})
        
        is_movie = "电影" in cat or cat in ["演唱会", "纪录片"]
        s_str = "" if is_movie else f"/Season {season_num}"
        await status.edit_text(f"✅ **档案创建成功！**\n🎬 `{folder_name}`\n📁 `{TARGET_MOUNT_ROOT}/{cat}/{folder_name}{s_str}`\n\n👉 **大门已开启，请抛入源文件！**")
        return

    if text.upper().startswith("#E"):
        m = re.search(r'^#E(\d+)', text, re.IGNORECASE)
        if m and GLOBAL_ROUTE_CACHE["expire_time"] > time.time():
            GLOBAL_ROUTE_CACHE["manual_ep"] = int(m.group(1))
            await message.reply_text(f"🔢 **物理集数强力覆写成功：E{GLOBAL_ROUTE_CACHE['manual_ep']:02d}**")
        return

# =================================================================
# 🚀 工业级流传输总枢 (V13 安全块精准续流装甲)
# =================================================================
@app.on_message(filters.video | filters.document)
async def handle_robust_transfer(client, message):
    media = message.video or message.document
    if not media: return
    
    global GLOBAL_ROUTE_CACHE
    if time.time() > GLOBAL_ROUTE_CACHE["expire_time"] or not GLOBAL_ROUTE_CACHE["folder_name"]:
        await message.reply_text("⚠️ **连接受阻**：航线锁未设定或失效过期。\n请发送 `/menu` 开启指引。")
        return
        
    raw_file = getattr(media, "file_name", f"TG_{message.id}.mp4")
    _, ext = os.path.splitext(raw_file)
    if not ext: ext = ".mp4"
    
    folder, cat, season, year = GLOBAL_ROUTE_CACHE["folder_name"], GLOBAL_ROUTE_CACHE["category"], GLOBAL_ROUTE_CACHE["season"], GLOBAL_ROUTE_CACHE["year"]
    is_movie = "电影" in cat or cat in ["演唱会", "纪录片"]
    
    if GLOBAL_ROUTE_CACHE["manual_ep"] is not None:
        ep_num = GLOBAL_ROUTE_CACHE["manual_ep"]
        GLOBAL_ROUTE_CACHE["manual_ep"] += 1
    else:
        ep_num = extract_episode(f"{message.caption or ''} {raw_file}")
        
    if is_movie:
        clean_base = re.sub(r'\(\d{4}\)', '', folder).strip()
        standard_name = f"{clean_base}.{year}{ext}" if year else f"{clean_base}{ext}"
        target_dir = f"{TARGET_MOUNT_ROOT}/{cat}/{folder}"
    else:
        clean_base = re.sub(r'\(\d{4}\)', '', folder).strip().replace(" ", ".")
        ep_str = f"{ep_num:02d}" if ep_num is not None else "00"
        standard_name = f"{clean_base}.S{season:02d}E{ep_str}.{year}{ext}" if year else f"{clean_base}.S{season:02d}E{ep_str}{ext}"
        target_dir = f"{TARGET_MOUNT_ROOT}/{cat}/{folder}/Season {season}"
        
    local_path = os.path.join(LOCAL_TEMP_DIR, standard_name)
    target_full = f"{target_dir}/{standard_name}".replace("//", "/")
    total_bytes = media.file_size
    size_mb = total_bytes / (1024 * 1024)
    
    status = await message.reply_text(f"⚡ 解析落定 ➔ `{standard_name}`\n📁 终态航线 ➔ `{target_dir}`")
    
    # =================================================================
    # 📥 阶段一：基于 Chunk 索引的严密对齐续写引擎
    # =================================================================
    chunk_size = 1024 * 1024 # MTProto 1MiB 传输基准块
    last_pct = 0
    retry_count = 0
    max_retries = 50 
    last_stuck_chunks = -1
    stuck_loop_count = 0
    
    while True:
        # 获取当前硬盘真实占用尺寸
        current_bytes = os.path.getsize(local_path) if os.path.exists(local_path) else 0
        
        if current_bytes >= total_bytes:
            downloaded_bytes = current_bytes
            break
            
        if retry_count >= max_retries:
            downloaded_bytes = current_bytes
            break
            
        # 异常阻断：超量脏写当场删除重置
        if current_bytes > total_bytes:
            try: os.remove(local_path)
            except Exception: pass
            current_bytes = 0
            
        # 🌟 终极破局算法：计算完整存活的安全块数量，并强制裁掉末端残块对齐 1MB 边界！
        completed_chunks = current_bytes // chunk_size
        secure_bytes = completed_chunks * chunk_size
        
        if secure_bytes > 0:
            try:
                # 物理切掉尾部断流瞬间可能受损的几百 KB 碎片
                with open(local_path, "r+b") as f:
                    f.truncate(secure_bytes)
            except Exception: pass
            downloaded_bytes = secure_bytes
            open_mode = "ab"
        else:
            downloaded_bytes = 0
            completed_chunks = 0
            open_mode = "wb"
            
        is_resume = downloaded_bytes > 0
        mode_txt = "安全块严密续流中" if is_resume else "全盘原生载入中"
        
        if retry_count == 0:
            await status.edit_text(f"📥 **[{mode_txt}]** `{standard_name}` ({size_mb:.1f} MB)")
            await notify_steward_log(f"拉取: {standard_name} (安全起点: {downloaded_bytes/(1024*1024):.1f}MB)")
            
        try:
            with open(local_path, open_mode) as f:
                # 🎯 满血核心修正：向底层精确传递已完成的 Chunk 数量索引！
                async for chunk in client.stream_media(message, offset=completed_chunks):
                    f.write(chunk)
                    downloaded_bytes += len(chunk)
                    
                    pct = int(downloaded_bytes * 100 / total_bytes)
                    if (pct // 10) > (last_pct // 10):
                        last_pct = pct - (pct % 10)
                        asyncio.create_task(status.edit_text(f"📥 **[{mode_txt}]** `{standard_name}`\n安全续流: **{last_pct}%**"))
                        await notify_steward_log(f"下载推进: {standard_name} ➔ {last_pct}%")
                        
            # 校验本轮接收结束后的总态
            if downloaded_bytes >= total_bytes: break
            else:
                retry_count += 1
                # 容忍阈值防死锁侦测
                if completed_chunks == last_stuck_chunks:
                    stuck_loop_count += 1
                    if stuck_loop_count >= 10:
                        logger.error("❌ 块指针寻轨连续受阻达10次，正物理清除重道载入...")
                        await status.edit_text("⚠️ **[寻址重置]** 块节点持续阻断\n🔄 正在清空受损池，强制发起原生接力...")
                        if os.path.exists(local_path): os.remove(local_path)
                        last_stuck_chunks = -1
                        stuck_loop_count = 0
                        await asyncio.sleep(2.0)
                        continue
                else:
                    last_stuck_chunks = completed_chunks
                    stuck_loop_count = 0
                    
                await status.edit_text(f"⚠️ **[CF 节点静默断流]** 管道安全块寻址中...\n💡 锁定边界刻度 ({downloaded_bytes/(1024*1024):.1f}MB)\n🔄 发起第 {retry_count} 次块接力 (死锁计数: {stuck_loop_count}/10)...")
                await asyncio.sleep(3.0)
                
        except Exception as e:
            retry_count += 1
            await status.edit_text(f"⚠️ **[网络闪断重置]** 路由漂移\n💡 锁定边界刻度 ({downloaded_bytes/(1024*1024):.1f}MB)\n🔄 发起第 {retry_count} 次接力...")
            await asyncio.sleep(3.0)
            
    if downloaded_bytes < total_bytes:
        await status.edit_text(f"❌ 物理流传输遭遇彻底阻死。\n💡 现场残片安全保留 ({downloaded_bytes/(1024*1024):.1f}MB)，可切换较稳节点重发发车指令接力。")
        await notify_steward_log(f"下载严重瘫痪: {standard_name}", level="ERROR")
        return
        
    await status.edit_text(f"✅ 物理块严密对齐竣工！\n🚀 启动底层网关安全推流...")

    # =================================================================
    # 📤 阶段二：安全持久化网关推流
    # =================================================================
    put_url = f"{OLIST_URL}/api/fs/put"
    headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
    
    push_success = False
    up_retries = 0
    max_up_retries = 5 
    
    while not push_success and up_retries < max_up_retries:
        try:
            last_up = 0
            async def file_iter():
                nonlocal last_up
                sent = 0
                with open(local_path, "rb") as f:
                    while True:
                        chunk = f.read(1024 * 1024)
                        if not chunk: break
                        sent += len(chunk)
                        pct = int(sent * 100 / total_bytes)
                        if (pct // 10) > (last_up // 10):
                            last_up = pct - (pct % 10)
                            asyncio.create_task(status.edit_text(f"🚀 **[网关桥接]** `{standard_name}`\n直推: **{last_up}%**" + (f" (重试 {up_retries})" if up_retries>0 else "")))
                        yield chunk

            async with httpx.AsyncClient(timeout=None) as h:
                resp = await h.put(put_url, content=file_iter(), headers=headers)
            
            if resp.json().get("code") == 200:
                push_success = True
                await status.edit_text(f"🎉 终极贯通落户 ➔ `{target_full}`")
                await notify_steward_log(f"云盘彻底接纳: {standard_name}")
            else:
                raise Exception(f"云端鉴权拒绝: {resp.text}")
                
        except Exception as up_err:
            up_retries += 1
            wait_time = up_retries * 4 
            await status.edit_text(f"⚠️ **[底层存储抖动]** 云端偶发超时\n💡 物理原件 100% 完好\n⏳ {wait_time} 秒后重推 (第 {up_retries} 次)...")
            await asyncio.sleep(wait_time)
            
    if not push_success:
        await status.edit_text(f"❌ 底层存储失联。\n💡 **本地完整档已保留**，重新点击车票重发本视频即可触发【0秒下载秒推】！")
        await notify_steward_log(f"网盘失联: {standard_name}，文件已保留", level="ERROR")
    else:
        try:
            if os.path.exists(local_path): os.remove(local_path)
        except Exception: pass

if __name__ == "__main__":
    clean_orphan_temp_files()
    save_tg_routes(load_tg_routes()) 
    logger.info("🤖 工业级全栖搬运中枢 (V13 真·物理对齐续流装甲) 满血上线！")
    app.run()
```
操作流程:

1./menu 发车  车票
转换集数：#EP1或#E01
2.转发视频
3.下载上传

### 四、autotg.py订阅功能
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
TARGET_MOUNT_ROOT = "/family/177_cas"

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
TG_ROUTE_DB = os.path.join(BASE_DIR, "tg_manual_routes.json")
LOCAL_TEMP_DIR = os.path.join(BASE_DIR, "tg_temp")
TG_LISTENER_DB = os.path.join(BASE_DIR, "tg_listener_config.json") 
TG_HISTORY_DB = os.path.join(BASE_DIR, "tg_download_history.json") 
os.makedirs(LOCAL_TEMP_DIR, exist_ok=True)

TMDB_API_KEY = "9c88e18e43543c8ff195c631aaa0d2fa" 
START_TIME = time.time()

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
    except Exception: 
        pass

# 🌟 V15.20 智能垃圾回收引擎
def clean_orphan_temp_files(max_age_hours=24):
    """
    智能清理：只删除存放超过 max_age_hours 小时的死碎片
    """
    if os.path.exists(LOCAL_TEMP_DIR):
        now = time.time()
        cleaned_count = 0
        for f in os.listdir(LOCAL_TEMP_DIR):
            file_path = os.path.join(LOCAL_TEMP_DIR, f)
            try:
                if os.path.isfile(file_path):
                    # 获取文件最后修改时间
                    mtime = os.path.getmtime(file_path)
                    if (now - mtime) > (max_age_hours * 3600):
                        os.remove(file_path)
                        cleaned_count += 1
            except Exception: pass
        if cleaned_count > 0:
            logger.info(f"🧹 [智能洗地] 已清理 {cleaned_count} 个滞留超过 {max_age_hours} 小时的陈旧碎片。")

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
            with open(TG_HISTORY_DB, 'r', encoding='utf-8') as f: return json.load(f)
        except Exception: pass
    return {}

def check_and_record_history(drama, season, ep):
    history = load_history()
    key = f"{drama}_S{season:02d}E{ep:02d}"
    if key in history: return True
    history[key] = int(time.time())
    try:
        with open(TG_HISTORY_DB, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
    except Exception: pass
    return False

def load_tg_routes():
    if os.path.exists(TG_ROUTE_DB):
        try:
            with open(TG_ROUTE_DB, 'r', encoding='utf-8') as f: return json.load(f)
        except Exception: pass
    return {}

def save_tg_routes(data):
    try:
        with open(TG_ROUTE_DB, 'w', encoding='utf-8') as f: json.dump(data, f, ensure_ascii=False, indent=2)
    except Exception as e: logger.error(f"保存路由失败: {e}")

GLOBAL_ROUTE_CACHE = {"folder_name": "", "category": "", "season": 1, "year": "", "expire_time": 0, "manual_ep": None}

async def fetch_tmdb_year(cn_title):
    if not TMDB_API_KEY: return datetime.now().strftime("%Y")
    clean_q = re.sub(r'S\d+$|\s+\d+$', '', cn_title).strip()
    url = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={quote(clean_q)}&page=1"
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            res = await client.get(url)
            results = res.json().get("results")
            if results and len(results[0].get("first_air_date") or results[0].get("release_date") or "") >= 4:
                return (results[0].get("first_air_date") or results[0].get("release_date"))[:4]
    except Exception: pass
    return datetime.now().strftime("%Y")

def extract_episode(search_text):
    text = search_text
    
    for p in [r'(?i)S\d+E(\d+)', r'第\d+[季部].*?第\s*(\d+)\s*[集话期]', r'第\s*(\d+)\s*[集话期]']:
        m = re.search(p, text)
        if m: return int(m.group(1))

    m_cn = re.search(r'第\s*([一二三四五六七八九十零百]+)\s*[集话期]', text)
    if m_cn:
        cn_str = m_cn.group(1)
        cn_map = {"一":1, "二":2, "三":3, "四":4, "五":5, "六":6, "七":7, "八":8, "九":9, "十":10}
        if cn_str in cn_map: return cn_map[cn_str]
        if cn_str.startswith("十") and len(cn_str) == 2: return 10 + cn_map.get(cn_str[1], 0)
        if len(cn_str) == 3 and cn_str[1] == "十": return cn_map.get(cn_str[0],0)*10 + cn_map.get(cn_str[2],0)
        if cn_str.endswith("十") and len(cn_str) == 2: return cn_map.get(cn_str[0],0)*10

    for p in [r'(?i)(?:^|[^a-zA-Z0-9])(?:EP|E)0*(\d+)(?:$|[^a-zA-Z0-9])', r'(?<!\d)(\d{1,2})x(\d+)(?!\d)']:
        m = re.search(p, text)
        if m: return int(m.group(2) if 'x' in p else m.group(1))
        
    for p in [r'\[\s*0*(\d+)\s*\]', r'【\s*0*(\d+)\s*】']:
        m = re.search(p, text)
        if m and not (1900 < int(m.group(1)) < 2100): return int(m.group(1))

    clean_text = search_text.replace("_", ".").replace(" ", ".").replace("-", ".")
    for p in [r'(?<=\.)0*(\d{1,3})(?=\.)', r'(?<=\.)0*(\d{1,3})(?=\.mp4|\.mkv|\.ts)']:
        m = re.search(p, clean_text, re.IGNORECASE)
        if m:
            num_val = int(m.group(1))
            if not (1900 < num_val < 2100) and m.group(1) not in ['264', '265']:
                return num_val

    return None

# =================================================================
# 📡 远程指令总枢
# =================================================================
STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]

@app.on_message(filters.command(["sub", "unsub", "list", "add_channel", "del_channel", "go", "history", "ping", "rm", "clean"]) & filters.user("me"))
async def manage_system_commands(client, message):
    command = message.command[0].lower()
    config = load_listener_config()
    
    if command == "ping":
        uptime_minutes = int((time.time() - START_TIME) / 60)
        channel_count = len(config.get("trusted_channels", {}))
        drama_count = sum(len(c.get("monitored_dramas", {})) for c in config.get("trusted_channels", {}).values())
        await message.reply_text(f"🟢 **系统健康度 [优秀]**\n⏱️ 已稳定存活: `{uptime_minutes}` 分钟\n📡 雷达阵列: 盯防 `{channel_count}` 个频道中的 `{drama_count}` 部剧")
        return
        
    # 🌟 V15.20 物理核弹清理指令
    if command == "clean":
        count = 0
        if os.path.exists(LOCAL_TEMP_DIR):
            for f in os.listdir(LOCAL_TEMP_DIR):
                file_path = os.path.join(LOCAL_TEMP_DIR, f)
                try:
                    if os.path.isfile(file_path):
                        os.remove(file_path)
                        count += 1
                except Exception: pass
        await message.reply_text(f"🧹 **[物理核弹清理]**\n成功抹除了 `{count}` 个遗留在硬盘上的滞留碎片！现在硬盘如丝般顺滑。")
        await notify_steward_log(f"🧹 执行物理核弹清理，抹除 {count} 个死碎片")
        return

    if command == "add_channel":
        if len(message.command) < 3: return await message.reply_text("⚠️ `/add_channel [频道ID] [别名]`")
        c_id, c_name = message.command[1], " ".join(message.command[2:])
        if c_id not in config["trusted_channels"]: config["trusted_channels"][c_id] = {"monitored_dramas": {}}
        config["trusted_channels"][c_id]["channel_name"] = c_name
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        await message.reply_text(f"🏢 **频道挂载成功**: {c_name} (`{c_id}`)")
        return

    if command == "del_channel":
        if len(message.command) < 2: return
        if message.command[1] in config["trusted_channels"]:
            del config["trusted_channels"][message.command[1]]
            with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
            await message.reply_text("🗑️ 频道已拔除。")
        return

    if command == "list":
        msg_text = "📡 **[雷达大盘状态]**\n\n"
        for chat_id, info in config.get("trusted_channels", {}).items():
            msg_text += f"🏢 **频道**: {info.get('channel_name', chat_id)} (`{chat_id}`)\n"
            dramas = info.get("monitored_dramas", {})
            if not dramas: msg_text += "  └ 📭 (空)\n"
            else:
                for d_name, d_info in dramas.items(): msg_text += f"  └ 🎬 `{d_name}` (从 E{d_info.get('min_ep', 1)} 起)\n"
            msg_text += "\n"
        await message.reply_text(msg_text)
        return

    if command == "sub":
        args = message.command[1:]
        if len(args) < 2: return await message.reply_text("⚠️ 语法：`/sub [剧名] [分类] [季数] [起始集]`")
            
        cat_idx = -1
        for i, arg in enumerate(args):
            if arg in STANDARD_CATS:
                cat_idx = i
                break
        
        if cat_idx == -1: return await message.reply_text(f"⚠️ 无法识别分类，请提供标准大类。")
            
        category = args.pop(cat_idx) 
        
        specific_channel = None
        for i, arg in enumerate(args):
            if arg.startswith("-100"):
                specific_channel = args.pop(i) 
                break
                
        min_ep = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        season = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        drama_name = " ".join(args) if args else "未知目标"
        
        target_pools = [specific_channel] if specific_channel else list(config["trusted_channels"].keys())
        for chat_id in target_pools:
            if "monitored_dramas" not in config["trusted_channels"][chat_id]: config["trusted_channels"][chat_id]["monitored_dramas"] = {}
            config["trusted_channels"][chat_id]["monitored_dramas"][drama_name] = {"category": category, "season": season, "min_ep": min_ep}
            
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        await message.reply_text(f"✅ **[雷达部署成功]**: `{drama_name}` ({category})！")
        return

    if command == "unsub":
        if len(message.command) < 2: return
        for chat_id in config["trusted_channels"]:
            if message.command[1] in config["trusted_channels"][chat_id].get("monitored_dramas", {}):
                del config["trusted_channels"][chat_id]["monitored_dramas"][message.command[1]]
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        await message.reply_text(f"🗑️ 已解除监控。")
        return

    if command == "history":
        routes = load_tg_routes()
        if not routes: return await message.reply_text("📭 没有留存记录。")
        sorted_items = sorted(routes.values(), key=lambda x: x.get("created_at", 0), reverse=True)[:20]
        msg_text = "📋 **[历史航线留存]**\n\n"
        for item in sorted_items: msg_text += f"🔸 `{item['category']}` | `{item['folder_name']}`\n"
        msg_text += "\n💡 删除多余航线请使用 `/rm [关键字]`"
        await message.reply_text(msg_text)
        return

    if command == "rm":
        if len(message.command) < 2: return await message.reply_text("⚠️ 语法：`/rm [剧名关键字]`")
        search_kw = " ".join(message.command[1:]).lower()
        routes = load_tg_routes()
        keys_to_del = [k for k, v in routes.items() if search_kw in k.lower() or search_kw in v.get("folder_name", "").lower()]
        if not keys_to_del: return await message.reply_text(f"🤷‍♂️ 未找到包含 `{search_kw}` 的航线。")
        for k in keys_to_del: del routes[k]
        save_tg_routes(routes)
        await message.reply_text("🗑️ **已从历史中抹除：**\n" + "\n".join([f"❌ `{k}`" for k in keys_to_del]))
        return

    if command == "go":
        args = message.command[1:]
        if not args: return await message.reply_text("⚠️ 语法：`/go [剧名] [分类]`")
            
        search_kw = " ".join(args).lower()
        routes = load_tg_routes()
        matched_item = None
        has_cat = any(arg in STANDARD_CATS for arg in args)
        
        if not has_cat:
            for k, v in routes.items():
                if search_kw in k.lower() or search_kw in v.get("folder_name", "").lower():
                    matched_item = v
                    break
        
        if matched_item:
            cat, folder_name = matched_item["category"], matched_item["folder_name"]
            season_num, year = matched_item.get("season", 1), matched_item.get("year", "")
            await message.reply_text(f"✅ **[唤醒历史记忆]** 命中！\n🎬 路线：`{folder_name}` ({cat})\n👉 请转发视频发车！")
        else:
            cat = None
            title_parts = []
            for arg in args:
                if arg in STANDARD_CATS and cat is None:
                    cat = arg  
                else:
                    title_parts.append(arg)  
                    
            if not cat: return await message.reply_text("⚠️ 请提供标准分类。")

            pure_title = " ".join(title_parts)
            status_lookup = await message.reply_text("🔍 正在请求 TMDB 校准首播年份...")
            
            season_num = 1
            s_match = re.search(r'\s+[sS第]?0?(\d{1,2})[季]?$', pure_title)
            if s_match: 
                season_num = int(s_match.group(1))
                pure_title = pure_title[:s_match.start()].strip() 
                
            year = await fetch_tmdb_year(pure_title)
            folder_name = pure_title if re.search(r'\(\d{4}\)', pure_title) else f"{pure_title} ({year})"
            
            routes[folder_name] = {"folder_name": folder_name, "category": cat, "season": season_num, "year": year, "created_at": int(time.time())}
            save_tg_routes(routes)
            await status_lookup.edit_text(f"✅ **[新航线已开辟]**\n🎬 最终建档：`{folder_name}` ({cat})\n👉 VIP 通道已打通，请转发视频入库！")

        global GLOBAL_ROUTE_CACHE
        GLOBAL_ROUTE_CACHE.update({"folder_name": folder_name, "category": cat, "season": season_num, "year": year, "expire_time": time.time() + 3600, "manual_ep": None})
        return

@app.on_message(filters.text & ~filters.media & filters.user("me"))
async def override_episode(client, message):
    text = message.text.strip()
    if text.upper().startswith("#E") and GLOBAL_ROUTE_CACHE["expire_time"] > time.time():
        m = re.search(r'^#E(\d+)', text, re.IGNORECASE)
        if m:
            GLOBAL_ROUTE_CACHE["manual_ep"] = int(m.group(1))
            await message.reply_text(f"🔢 **集数已强行锁死：E{GLOBAL_ROUTE_CACHE['manual_ep']:02d}**")

# =================================================================
# 🚀 V14 涡轮引擎核心
# =================================================================
async def process_media_transfer(client, message, status):
    media = message.video or message.document
    raw_file = getattr(media, "file_name", f"TG_{message.id}.mp4")
    
    _, ext = os.path.splitext(raw_file)
    if not ext: ext = ".mp4"
    
    folder, cat, season, year = GLOBAL_ROUTE_CACHE["folder_name"], GLOBAL_ROUTE_CACHE["category"], GLOBAL_ROUTE_CACHE["season"], GLOBAL_ROUTE_CACHE["year"]
    is_movie = "电影" in cat or cat in ["演唱会", "纪录片"]
    
    if GLOBAL_ROUTE_CACHE["manual_ep"] is not None:
        ep_num = GLOBAL_ROUTE_CACHE["manual_ep"]
        GLOBAL_ROUTE_CACHE["manual_ep"] += 1
    else: 
        ep_num = extract_episode(f"{message.caption or ''} {raw_file}")
        
    if is_movie:
        clean_base = re.sub(r'\(\d{4}\)', '', folder).strip()
        standard_name = f"{clean_base}.{year}{ext}" if year else f"{clean_base}{ext}"
        target_dir = f"{TARGET_MOUNT_ROOT}/{cat}/{folder}"
    else:
        clean_base = re.sub(r'\(\d{4}\)', '', folder).strip().replace(" ", ".")
        ep_str = f"{ep_num:02d}" if ep_num is not None else "00"
        standard_name = f"{clean_base}.S{season:02d}E{ep_str}.{year}{ext}" if year else f"{clean_base}.S{season:02d}E{ep_str}{ext}"
        target_dir = f"{TARGET_MOUNT_ROOT}/{cat}/{folder}/Season {season}"
        
    local_path = os.path.join(LOCAL_TEMP_DIR, standard_name)
    target_full = f"{target_dir}/{standard_name}".replace("//", "/")
    total_bytes = media.file_size
    size_mb = total_bytes / (1024 * 1024)
    
    await notify_steward_log(f"📥 [涡轮启动] 开始拉取: {standard_name} ({size_mb:.1f} MB)")
    
    chunk_size = 1024 * 1024
    last_pct = 0
    retry_count = 0
    max_retries = 50
    last_stuck_chunks = -1
    stuck_loop_count = 0
    
    while True:
        current_bytes = os.path.getsize(local_path) if os.path.exists(local_path) else 0
        if current_bytes >= total_bytes: break
        if retry_count >= max_retries: break
        if current_bytes > total_bytes:
            try: os.remove(local_path)
            except Exception: pass
            current_bytes = 0
            
        completed_chunks = current_bytes // chunk_size
        secure_bytes = completed_chunks * chunk_size
        if secure_bytes > 0:
            try:
                with open(local_path, "r+b") as f: f.truncate(secure_bytes)
            except Exception: pass
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
                            f.write(chunk)
                            buffer_queue.task_done()
                except Exception as e: writer_error = e

            writer_task = asyncio.create_task(disk_writer())
            
            async for chunk in client.stream_media(message, offset=completed_chunks):
                if writer_error: raise writer_error 
                await buffer_queue.put(chunk)
                downloaded_bytes += len(chunk)
                pct = int(downloaded_bytes * 100 / total_bytes)
                if (pct // 10) > (last_pct // 10):
                    last_pct = pct - (pct % 10)
                    asyncio.create_task(status.edit_text(f"🚀 **[极速拉取]** `{standard_name}`\n涡轮进度: **{last_pct}%**"))
            
            await buffer_queue.put(None)
            await writer_task
            
            if downloaded_bytes >= total_bytes: break
            else:
                retry_count += 1
                if completed_chunks == last_stuck_chunks:
                    stuck_loop_count += 1
                    if stuck_loop_count >= 10:
                        if os.path.exists(local_path): os.remove(local_path)
                        last_stuck_chunks = -1; stuck_loop_count = 0; continue
                    
                    await status.edit_text(f"⚠️ **[节点堵塞]** 正在重新校准物理块...\n🔄 第 **{retry_count}** 次死磕底层数据...")
                else: 
                    last_stuck_chunks = completed_chunks; stuck_loop_count = 0
                await asyncio.sleep(3.0)
        
        except Exception as e:
            if 'writer_task' in locals() and not writer_task.done(): writer_task.cancel()
            retry_count += 1
            await status.edit_text(f"⚠️ **[网络闪断]** 传输突发中断！\n🛠️ 正在为您进行第 **{retry_count}** 次热线重联，死磕到底...")
            await notify_steward_log(f"⚠️ [防抖保护] 触发第 {retry_count} 次物理块断点重联: {standard_name}", level="WARNING")
            await asyncio.sleep(3.0)
            
    if downloaded_bytes < total_bytes:
        await status.edit_text("❌ 下载断链，已安全保留残片。")
        await notify_steward_log(f"❌ [拉取失败] 传输崩溃: {standard_name}", level="ERROR")
        return
        
    await status.edit_text("✅ 本地落盘完成，正在上传云端...")
    await notify_steward_log(f"✅ [落盘完成] 准备直推天翼云: {standard_name}")
    
    put_url = f"{OLIST_URL}/api/fs/put"
    headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
    push_success = False
    up_retries = 0
    
    while not push_success and up_retries < 5:
        try:
            last_up = 0
            async def file_iter():
                nonlocal last_up
                sent = 0
                with open(local_path, "rb") as f:
                    while True:
                        chunk = f.read(1024 * 1024)
                        if not chunk: break
                        sent += len(chunk)
                        pct = int(sent * 100 / total_bytes)
                        if (pct // 10) > (last_up // 10):
                            last_up = pct - (pct % 10)
                            asyncio.create_task(status.edit_text(f"🚀 **[直推天翼云]** `{standard_name}`\n云端进度: **{last_up}%**"))
                        yield chunk
            
            async with httpx.AsyncClient(timeout=None) as h: 
                resp = await h.put(put_url, content=file_iter(), headers=headers)
                
            if resp.json().get("code") == 200: push_success = True
            else: raise Exception()
        except Exception: 
            up_retries += 1
            await status.edit_text(f"⚠️ **[云端推流阻断]**\n🔄 正在进行第 **{up_retries}** 次网关重联...")
            await notify_steward_log(f"⚠️ [上传防抖] 触发第 {up_retries} 次重联推送", level="WARNING")
            await asyncio.sleep(up_retries*4)
            
    if not push_success: 
        await status.edit_text("❌ 云盘连接彻底超时，物理残件已保存。")
        await notify_steward_log(f"❌ [云端失联] 推流桥接失败: {standard_name}", level="ERROR")
    else:
        await status.edit_text(f"🎉 **终极入库成功** ➔ `{target_full}`")
        await notify_steward_log(f"🎉 [入库成功] {target_full}")
        if not is_movie and ep_num is not None:
            check_and_record_history(folder.split(" (")[0], season, ep_num)
        try:
            if os.path.exists(local_path): os.remove(local_path)
        except Exception: pass

# =================================================================
# 🎯 路由入口大总管
# =================================================================
@app.on_message(filters.video | filters.document)
async def media_routing_gateway(client, message):
    config = load_listener_config()
    chat_id_to_check = None
    
    if message.chat and str(message.chat.id) in config.get("trusted_channels", {}): 
        chat_id_to_check = str(message.chat.id)
    elif message.forward_from_chat and str(message.forward_from_chat.id) in config.get("trusted_channels", {}): 
        chat_id_to_check = str(message.forward_from_chat.id)
        
    if chat_id_to_check:
        channel_info = config["trusted_channels"][chat_id_to_check]
        media = message.video or message.document
        text_to_scan = f"{message.caption or ''} {getattr(media, 'file_name', '')}"
        
        await notify_steward_log(f"🕵️ [雷达嗅探] 频道 {chat_id_to_check} 传来视频，携带文本: 【{text_to_scan}】")
        
        for drama_name, route_info in channel_info.get("monitored_dramas", {}).items():
            if drama_name in text_to_scan:
                ep_num = extract_episode(text_to_scan)
                
                if ep_num is not None and ep_num < route_info.get("min_ep", 1): 
                    await notify_steward_log(f"⏭️ [集数太老丢弃] {drama_name} E{ep_num} < 设定的起步集数 {route_info.get('min_ep')}")
                    continue 
                if ep_num is not None:
                    if f"{drama_name}_S{route_info['season']:02d}E{ep_num:02d}" in load_history(): 
                        await notify_steward_log(f"⏭️ [重复下载丢弃] {drama_name} E{ep_num} 账本里已存在，拒绝重复拉取！")
                        return 
                
                year = await fetch_tmdb_year(drama_name)
                
                global GLOBAL_ROUTE_CACHE
                GLOBAL_ROUTE_CACHE.update({
                    "folder_name": f"{drama_name} ({year})" if year else drama_name, 
                    "category": route_info["category"], 
                    "season": route_info["season"], 
                    "year": year, 
                    "expire_time": time.time() + 3600, 
                    "manual_ep": ep_num
                })
                
                channel_name_str = channel_info.get('channel_name', chat_id_to_check)
                ep_str_display = f"E{ep_num:02d}" if ep_num is not None else "未知(当做E00)"
                
                try:
                    status = await client.send_message(
                        COMMAND_CENTER_CHAT, 
                        f"🎯 **[雷达静默捕获]**\n📺 剧名：`{drama_name}`\n🔢 集数：`{ep_str_display}`\n📡 源头：{channel_name_str}\n🚀 涡轮产线已在后台全自动点火..."
                    )
                except Exception as e:
                    await notify_steward_log(f"⚠️ [发信失败] 无法联系 {COMMAND_CENTER_CHAT}，报错: {e}，将转投收藏夹", level="WARNING")
                    status = await client.send_message(
                        "me", 
                        f"⚠️ **[发信失败]** 无法送达指挥所！\n\n🎯 **[雷达静默捕获]**\n📺 剧名：`{drama_name}`\n🔢 集数：`{ep_str_display}`\n🚀 产线照常运行..."
                    )
                
                await notify_steward_log(f"🎯 [雷达命中] 捕获新剧集: {drama_name} {ep_str_display} (来自 {channel_name_str})")
                await process_media_transfer(client, message, status)
                return 

    if time.time() > GLOBAL_ROUTE_CACHE["expire_time"] or not GLOBAL_ROUTE_CACHE["folder_name"]:
        if message.chat.type in [enums.ChatType.PRIVATE, enums.ChatType.BOT]: 
            await message.reply_text("⚠️ **当前无锁定航线**\n\n复用老剧：`/go 剧名关键字`\n开辟新剧：`/go [剧名] [分类]`\n清理硬盘：`/clean`\n清理账本：`/rm 剧名关键字`")
        return
        
    status = await message.reply_text("⚡ 航线认证通过，正向引流拉取...")
    await process_media_transfer(client, message, status)

# =================================================================
# 🚀 引擎启动与防失忆系统热身
# =================================================================
if __name__ == "__main__":
    # 🌟 开机只清理超过 24 小时的陈年死碎片，刚断网的统统保留！
    clean_orphan_temp_files(max_age_hours=24) 
    
    load_listener_config()
    
    async def start_system():
        await app.start()
        
        await notify_steward_log("🔄 [系统预热] 正在强制打通指挥所通讯隧道...")
        try:
            peer = await app.get_users(COMMAND_CENTER_CHAT)
            await notify_steward_log(f"✅ 指挥所坐标锁定: {peer.first_name} (ID: {peer.id})")
            await app.send_message(COMMAND_CENTER_CHAT, "🤖 报告老板：全栖搬运中枢已满血重启，通讯隧道畅通！断点续传防线已就绪！")
        except Exception as e:
            await notify_steward_log(f"⚠️ 致命警告：解析指挥所 {COMMAND_CENTER_CHAT} 失败 - {e}", level="ERROR")
            
        try:
            async for _ in app.get_dialogs(limit=200): pass
            await notify_steward_log("✅ 频道记忆库同步完成。")
        except Exception as e:
            pass
            
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

/add_channel [频道ID] [别名] ➔ 🏢 把一个发资源的群加入信任大名单。

(例：/add_channel -1001234567 顶级原盘群)

/del_channel [频道ID] ➔ 🗑️ 把某个群从大名单里踢出去。

/sub [分类] [剧名] [季数] [起步集] ➔ 🎯 部署雷达！（分类和剧名谁在前都可以，闭眼发）。

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

(例：视频名字太乱提取不出集数，你直接发 #E12，下一集强制按 12 集命名)

🧹 三、 清理与维护（本次为你紧急加装！）
/rm [剧名关键字] ➔ 💥 删除历史航线！ 把不再需要的死档从 /history 里永久抹除。

(例：/rm 卧底 ➔ 瞬间清理账本，保持后台极度干净)

/clean 清除下载碎片