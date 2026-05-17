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

# 🔥 绝对不能改名！继续沿用 v15 凭证，保证秒连！
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
            with open(TG_HISTORY_DB, 'r', encoding='utf-8') as f: return json.load(f)
        except Exception: pass
    return {}

def check_and_record_history(drama, file_season, ep):
    history = load_history()
    key = f"{drama}_S{file_season:02d}E{ep:02d}"
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
    except Exception: pass

GLOBAL_ROUTE_CACHE = {"folder_name": "", "category": "", "folder_season": 1, "file_season": 1, "year": "", "expire_time": 0, "manual_ep": None}

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

def extract_pure_episode(search_text):
    text = search_text
    m = re.search(r'(?i)E0*(\d+)', text)
    if m: return int(m.group(1))
    m = re.search(r'第\d+[季部].*?第\s*(\d+)\s*[集话期]', text)
    if m: return int(m.group(1))
    m = re.search(r'第\s*(\d+)\s*[集话期]', text)
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
    clean_text = search_text.replace("_", ".").replace(" ", ".")
    for p in [r'(?<=\.)0*(\d{1,3})(?=\.)', r'(?<=\.)0*(\d{1,3})(?=\.mp4|\.mkv|\.ts)']:
        m = re.search(p, clean_text, re.IGNORECASE)
        if m:
            num_val = int(m.group(1))
            if not (1900 < num_val < 2100) and m.group(1) not in ['264', '265']: return num_val
    return None

# =================================================================
# 🚀 涡轮传输核心逻辑 (支持双轨季数)
# =================================================================
async def process_media_transfer(client, message, status, override_info=None):
    media = message.video or message.document
    if not media: return
    raw_file = getattr(media, "file_name", f"TG_{message.id}.mp4")
    _, ext = os.path.splitext(raw_file)
    if not ext: ext = ".mp4"
    
    if override_info:
        folder, cat, folder_season, file_season, year, ep_num = override_info
    else:
        folder, cat = GLOBAL_ROUTE_CACHE["folder_name"], GLOBAL_ROUTE_CACHE["category"]
        folder_season, file_season = GLOBAL_ROUTE_CACHE["folder_season"], GLOBAL_ROUTE_CACHE["file_season"]
        year = GLOBAL_ROUTE_CACHE["year"]
        
        if GLOBAL_ROUTE_CACHE["manual_ep"] is not None:
            ep_num = GLOBAL_ROUTE_CACHE["manual_ep"]
            GLOBAL_ROUTE_CACHE["manual_ep"] += 1
        else: 
            ep_num = extract_pure_episode(f"{message.caption or ''} {raw_file}")
            
    is_movie = "电影" in cat or cat in ["演唱会", "纪录片"]
        
    if is_movie:
        clean_base = re.sub(r'\(\d{4}\)', '', folder).strip()
        standard_name = f"{clean_base}.{year}{ext}" if year else f"{clean_base}{ext}"
        target_dir = f"{TARGET_MOUNT_ROOT}/{cat}/{folder}"
    else:
        clean_base = re.sub(r'\(\d{4}\)', '', folder).strip().replace(" ", ".")
        ep_str = f"{ep_num:02d}" if ep_num is not None else "00"
        
        target_dir = f"{TARGET_MOUNT_ROOT}/{cat}/{folder}/Season {folder_season}"
        standard_name = f"{clean_base}.S{file_season:02d}E{ep_str}.{year}{ext}" if year else f"{clean_base}.S{file_season:02d}E{ep_str}{ext}"
        
    local_path = os.path.join(LOCAL_TEMP_DIR, standard_name)
    target_full = f"{target_dir}/{standard_name}".replace("//", "/")
    total_bytes = media.file_size
    
    await notify_steward_log(f"📥 [涡轮启动] 拉取: {standard_name}")
    
    chunk_size = 1024 * 1024
    last_pct = 0
    retry_count = 0
    
    while True:
        current_bytes = os.path.getsize(local_path) if os.path.exists(local_path) else 0
        if current_bytes >= total_bytes or retry_count >= 50: break
        if current_bytes > total_bytes:
            try: os.remove(local_path)
            except: pass
            current_bytes = 0
            
        completed_chunks = current_bytes // chunk_size
        secure_bytes = completed_chunks * chunk_size
        if secure_bytes > 0:
            with open(local_path, "r+b") as f: f.truncate(secure_bytes)
            downloaded_bytes, open_mode = secure_bytes, "ab"
        else: 
            downloaded_bytes, open_mode = 0, "wb"
            
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
            async for chunk in client.stream_media(message, offset=completed_chunks):
                if writer_error: raise writer_error 
                await buffer_queue.put(chunk)
                downloaded_bytes += len(chunk)
                pct = int(downloaded_bytes * 100 / total_bytes)
                if (pct // 10) > (last_pct // 10):
                    last_pct = pct - (pct % 10)
                    asyncio.create_task(status.edit_text(f"🚀 **[极速拉取]** `{standard_name}`\n涡轮进度: **{last_pct}%**"))
            await buffer_queue.put(None); await writer_task; break
        except Exception:
            retry_count += 1
            await status.edit_text(f"⚠️ 网络中断，正在进行第 {retry_count} 次重连...")
            await asyncio.sleep(3.0)
            
    if downloaded_bytes < total_bytes: return await status.edit_text("❌ 下载崩溃。")
    await status.edit_text("✅ 本地落盘完成，上传云端...")
    
    put_url = f"{OLIST_URL}/api/fs/put"
    headers = {"Authorization": OLIST_TOKEN, "File-Path": quote(target_full), "Content-Length": str(total_bytes), "Content-Type": "application/octet-stream"}
    push_success, up_retries = False, 0
    
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
            await status.edit_text(f"⚠️ 云端传输受阻，正在进行第 {up_retries} 次重连...")
            await asyncio.sleep(up_retries*4)
            
    if not push_success: await status.edit_text("❌ 云盘上传彻底失败。")
    else:
        await status.edit_text(f"🎉 **入库成功** ➔ `{target_full}`")
        if not is_movie and ep_num is not None:
            check_and_record_history(folder.split(" (")[0], file_season, ep_num)
        try: os.remove(local_path)
        except: pass

# =================================================================
# 🌟 全自动订阅与历史挖坟引擎
# =================================================================
async def sweep_existing_history(client, chat_id, drama_name, category, folder_season, file_season, min_ep):
    try:
        chat_id_int = int(chat_id) if str(chat_id).lstrip('-').isdigit() else chat_id
        await notify_steward_log(f"🔍 开始向后挖掘频道 {chat_id} 中的现成历史剧集...")
        
        async for old_msg in client.get_chat_history(chat_id_int, limit=200):
            media = old_msg.video or old_msg.document
            if not media: continue
            
            text_to_scan = f"{old_msg.caption or ''} {getattr(media, 'file_name', '')}"
            if drama_name.lower() in text_to_scan.lower():
                ep_num = extract_pure_episode(text_to_scan)
                if ep_num is None or ep_num < min_ep: continue
                
                year = await fetch_tmdb_year(drama_name)
                folder_name = f"{drama_name} ({year})" if year else drama_name
                
                if f"{folder_name.split(' (')[0]}_S{file_season:02d}E{ep_num:02d}" in load_history(): 
                    continue
                    
                status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[现成老集数挖出]**\n📺 `{drama_name}` ➔ S{file_season:02d}E{ep_num:02d}\n🚀 已自动点火，暴力拉取中...")
                
                override_info = (folder_name, category, folder_season, file_season, year, ep_num)
                await process_media_transfer(client, old_msg, status, override_info)
                await asyncio.sleep(3) 
    except Exception as e:
        await notify_steward_log(f"⚠️ 扫荡现成记录出错: {e}", level="WARNING")

# =================================================================
# 📡 远程指令总枢
# =================================================================
STANDARD_CATS = ["华语剧", "欧美剧", "日韩剧", "短剧", "华语电影", "欧美电影", "日韩电影", "演唱会", "国漫", "日漫", "综艺", "纪录片"]

@app.on_message(filters.command(["sub", "unsub", "list", "add_channel", "del_channel", "go", "history", "ping", "rm", "clean", "scan"]) & filters.user("me"))
async def manage_system_commands(client, message):
    command = message.command[0].lower()
    config = load_listener_config()
    
    if command == "ping":
        uptime_minutes = int((time.time() - START_TIME) / 60)
        await message.reply_text(f"🟢 **系统健康度 [优秀]**\n⏱️ 存活: `{uptime_minutes}` 分钟")
        return

    if command == "scan":
        args = message.command[1:]
        target_kw = " ".join(args).lower() if args else None # 获取你输入的特定剧名
        
        total_tasks = 0
        msg = f"🔍 **[精准扫荡下达]** ➔ 目标: `{target_kw}`" if target_kw else "🔍 **[全网大扫荡下达]** ➔ 目标: `所有订阅`"
        await message.reply_text(f"{msg}\n🚀 正在全自动翻找频道历史捡漏...")
        
        for chat_id, info in config.get("trusted_channels", {}).items():
            for drama_name, d_info in info.get("monitored_dramas", {}).items():
                
                # 🔥 核心：如果你加了剧名，名字对不上的就直接跳过，只扫你想扫的！
                if target_kw and target_kw not in drama_name.lower():
                    continue
                    
                folder_s = d_info.get("folder_season", 1)
                file_s = d_info.get("file_season", folder_s)
                min_ep = d_info.get("min_ep", 1)
                cat = d_info.get("category", "未分类")
                
                asyncio.create_task(sweep_existing_history(client, chat_id, drama_name, cat, folder_s, file_s, min_ep))
                total_tasks += 1
                
        if total_tasks > 0:
            return await message.reply_text(f"✅ 已在后台启动 `{total_tasks}` 组扫荡雷达！")
        else:
            return await message.reply_text("⚠️ 未匹配到订阅记录，请使用 /list 检查监控大盘。")
        
    if command == "clean":
        count = 0
        if os.path.exists(LOCAL_TEMP_DIR):
            for f in os.listdir(LOCAL_TEMP_DIR):
                try: os.remove(os.path.join(LOCAL_TEMP_DIR, f)); count += 1
                except: pass
        await message.reply_text(f"🧹 抹除 `{count}` 个遗留碎片！")
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
            msg_text += f"🏢 **频道**: {info.get('channel_name', chat_id)}\n"
            dramas = info.get("monitored_dramas", {})
            if not dramas: msg_text += "  └ 📭 (空)\n"
            else:
                for d_name, d_info in dramas.items(): 
                    s_text = f"文件夹Season {d_info['folder_season']} / 文件名S{d_info['file_season']:02d}"
                    msg_text += f"  └ 🎬 `{d_name}` ({s_text})\n"
        await message.reply_text(msg_text)
        return

    if command == "sub":
        args = message.command[1:]
        if len(args) < 2: return await message.reply_text("⚠️ 语法：`/sub [剧名] [分类] [文件夹季数] [起步集] [可选:S文件名季数]`")
            
        cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
        if cat_idx == -1: return await message.reply_text("⚠️ 请提供标准分类。")
        category = args.pop(cat_idx) 
        
        specific_channel = next((args.pop(i) for i, arg in enumerate(args) if arg.startswith("-100")), None)
        
        file_season = None
        if args and re.match(r'^s\d+$', args[-1], re.IGNORECASE):
            file_season = int(args.pop(-1)[1:])
            
        min_ep = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        folder_season = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        
        if file_season is None: file_season = folder_season
        
        drama_name = " ".join(args) if args else "未知目标"
        
        target_pools = [specific_channel] if specific_channel else list(config["trusted_channels"].keys())
        for chat_id in target_pools:
            if "monitored_dramas" not in config["trusted_channels"][chat_id]: config["trusted_channels"][chat_id]["monitored_dramas"] = {}
            config["trusted_channels"][chat_id]["monitored_dramas"][drama_name] = {
                "category": category, 
                "folder_season": folder_season,
                "file_season": file_season, 
                "min_ep": min_ep
            }
            
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        
        s_mode_txt = f"同调: Season {folder_season} & S{file_season:02d}" if folder_season == file_season else f"强制分离: Season {folder_season} & S{file_season:02d}"
        await message.reply_text(f"✅ **订阅成功**: `{drama_name}`\n🛠️ 模式: `{s_mode_txt}`\n🚀 **正在自动清点频道历史，挖出现成剧集马上开拉！**")
        
        for chat_id in target_pools:
            asyncio.create_task(sweep_existing_history(client, chat_id, drama_name, category, folder_season, file_season, min_ep))
        return

    if command == "unsub":
        if len(message.command) < 2: return
        for chat_id in config["trusted_channels"]:
            if message.command[1] in config["trusted_channels"][chat_id].get("monitored_dramas", {}):
                del config["trusted_channels"][chat_id]["monitored_dramas"][message.command[1]]
        with open(TG_LISTENER_DB, "w", encoding="utf-8") as f: json.dump(config, f, ensure_ascii=False, indent=4)
        await message.reply_text(f"🗑️ 已解除监控。")
        return

    if command == "go":
        args = message.command[1:]
        if not args: return await message.reply_text("⚠️ 语法：`/go [剧名关键字]` (复用历史)\n或 `/go [剧名] [分类] [文件夹季数] [可选:S文件名季数]` (开辟新线)")
        
        routes = load_tg_routes()
        cat_idx = next((i for i, arg in enumerate(args) if arg in STANDARD_CATS), -1)
        
        # 🔥 完美找回：如果没有填分类，说明是想复用历史库！
        if cat_idx == -1:
            search_kw = " ".join(args).lower()
            matched_item = None
            for k, v in routes.items():
                if search_kw in k.lower() or search_kw in v.get("folder_name", "").lower():
                    matched_item = v
                    break
            
            if matched_item:
                folder_s = matched_item.get("folder_season", 1)
                file_s = matched_item.get("file_season", folder_s)
                GLOBAL_ROUTE_CACHE.update({
                    "folder_name": matched_item["folder_name"], 
                    "category": matched_item["category"], 
                    "folder_season": folder_s, 
                    "file_season": file_s, 
                    "year": matched_item.get("year", ""), 
                    "expire_time": time.time() + 3600, 
                    "manual_ep": None
                })
                s_mode_txt = f"Season {folder_s} & S{file_s:02d}"
                return await message.reply_text(f"✅ **[唤醒历史记忆]** 命中！\n🎬 路线：`{matched_item['folder_name']}`\n🛠️ 模式：`{s_mode_txt}`\n👉 请直接转发视频发车！")
            else:
                return await message.reply_text("⚠️ 未匹配到历史，如果要开辟新线请提供标准分类。")
        
        # 下面是开辟新线的双轨逻辑
        cat = args.pop(cat_idx)
        
        file_season = None
        if args and re.match(r'^s\d+$', args[-1], re.IGNORECASE):
            file_season = int(args.pop(-1)[1:])
            
        folder_season = int(args.pop(-1)) if args and args[-1].isdigit() else 1
        
        if file_season is None: file_season = folder_season 
        
        pure_title = " ".join(args)
        year = await fetch_tmdb_year(pure_title)
        folder_name = f"{pure_title} ({year})"
        
        routes[folder_name] = {"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "created_at": int(time.time())}
        save_tg_routes(routes)
        
        GLOBAL_ROUTE_CACHE.update({"folder_name": folder_name, "category": cat, "folder_season": folder_season, "file_season": file_season, "year": year, "expire_time": time.time() + 3600, "manual_ep": None})
        s_mode_txt = f"同调: Season {folder_season} & S{file_season:02d}" if folder_season == file_season else f"分离: Season {folder_season} & S{file_season:02d}"
        await message.reply_text(f"✅ **新航线已打通** ➔ 模式: `{s_mode_txt}`\n👉 请直接转发媒体，或先发 `#E数字` 锁定集数！")

@app.on_message(filters.text & ~filters.media & filters.user("me"))
async def override_episode(client, message):
    text = message.text.strip()
    if text.upper().startswith("#E") and GLOBAL_ROUTE_CACHE["expire_time"] > time.time():
        m = re.search(r'^#E(\d+)', text, re.IGNORECASE)
        if m:
            GLOBAL_ROUTE_CACHE["manual_ep"] = int(m.group(1))
            await message.reply_text(f"🔢 **集数霸权锁死：下一集强制命名为 E{GLOBAL_ROUTE_CACHE['manual_ep']:02d}**")

# =================================================================
# 🎯 路由入口大总管
# =================================================================
@app.on_message(filters.video | filters.document)
async def media_routing_gateway(client, message):
    config = load_listener_config()
    chat_id_to_check = str(message.chat.id) if message.chat else (str(message.forward_from_chat.id) if message.forward_from_chat else "")
    
    matched_channel = next((k for k in config.get("trusted_channels", {}) if k in chat_id_to_check or chat_id_to_check in k), None)
        
    if matched_channel:
        channel_info = config["trusted_channels"][matched_channel]
        media = message.video or message.document
        text_to_scan = f"{message.caption or ''} {getattr(media, 'file_name', '')}"
        
        for drama_name, route_info in channel_info.get("monitored_dramas", {}).items():
            if drama_name.lower() in text_to_scan.lower():
                ep_num = extract_pure_episode(text_to_scan)
                if ep_num is not None and ep_num < route_info.get("min_ep", 1): continue 
                
                folder_season = route_info.get("folder_season", 1)
                file_season = route_info.get("file_season", folder_season)
                
                year = await fetch_tmdb_year(drama_name)
                folder_name = f"{drama_name} ({year})" if year else drama_name
                
                if ep_num is not None and f"{folder_name.split(' (')[0]}_S{file_season:02d}E{ep_num:02d}" in load_history(): return 
                
                override_info = (folder_name, route_info["category"], folder_season, file_season, year, ep_num)
                status = await client.send_message(COMMAND_CENTER_CHAT, f"🎯 **[雷达嗅探新更]**\n📺 `{drama_name}` ➔ S{file_season:02d}E{ep_num:02d}\n🚀 根本不看原生命名，全自动拦截发车...")
                await process_media_transfer(client, message, status, override_info)
                return 

    if time.time() > GLOBAL_ROUTE_CACHE["expire_time"] or not GLOBAL_ROUTE_CACHE["folder_name"]:
        if message.chat and message.chat.type in [enums.ChatType.PRIVATE, enums.ChatType.BOT]: 
            await message.reply_text("⚠️ 无可用锁定航线。请先输入 /go 或 /sub。")
        return
        
    status = await message.reply_text("⚡ 航线认证通过，正向引流拉取...")
    await process_media_transfer(client, message, status)

# =================================================================
# 🚀 引擎启动 (完美复原 V15 预热打通模块！)
# =================================================================
if __name__ == "__main__":
    clean_orphan_temp_files(max_age_hours=24) 
    load_listener_config()
    
    async def start_system():
        await app.start()
        await notify_steward_log("🔄 [系统预热] 正在强制打通指挥所通讯隧道...")
        try:
            peer = await app.get_users(COMMAND_CENTER_CHAT)
            await notify_steward_log(f"✅ 指挥所坐标锁定: {peer.first_name} (ID: {peer.id})")
            await app.send_message(COMMAND_CENTER_CHAT, "🤖 报告老板：终极双轨定制版已满血重启，后台通讯隧道畅通！")
        except Exception as e:
            await notify_steward_log(f"⚠️ 致命警告：解析指挥所 {COMMAND_CENTER_CHAT} 失败 - {e}", level="ERROR")
            
        try:
            async for _ in app.get_dialogs(limit=200): pass
            await notify_steward_log("✅ 频道记忆库同步完成。")
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

/add_channel [频道ID] [别名] ➔ 🏢 把一个发资源的群加入信任大名单。

(例：/add_channel -1001234567 顶级原盘群)

/del_channel [频道ID] ➔ 🗑️ 把某个群从大名单里踢出去。

/sub [分类] [剧名] [季数] [起步集] ➔ 🎯 部署雷达！（分类和剧名谁在前都可以，闭眼发）。

完整语法：/sub [剧名关键字] [分类] [文件夹季数] [起步集数] [可选: 文件名季数]

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

/clean 清除下载碎片