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

### 三、脚本autotg.py

```
import os
import re
import time
import asyncio
from urllib.parse import quote
import httpx
from pyrogram import Client, filters
import logging

# =================================================================
# ⚙️ 核心配置区域
# =================================================================
API_ID = 33349348             # ⚠️ 替换为真实 API_ID
API_HASH = "44bde7f01d2b6001589c28cea93716af"     # ⚠️ 替换为真实 API_HASH
BOT_TOKEN = "8235305939:AAEg0ICkxUSwRPrg0FlXb4o89In8WUKdM3Y" # ⚠️ 替换为真实 Bot Token

OLIST_URL = "http://127.0.0.1:5244"
OLIST_TOKEN = "openlist-a87614da-32dd-4b80-9150-6447de823da8f33x53ymkrx0aPKG0HUcsFHmjFRYTKFhSADLRhoQLkXa7ogaiByhWRNEXCjpblp9" # ⚠️ 替换为真实 Token

# 🎭 后台大管家服务联动配置 (playcas.py 监听端口)
STEWARD_BASE_URL = "http://127.0.0.1:5000"

# Termux 本地物理暂存池
LOCAL_TEMP_DIR = "/data/data/com.termux/files/home/tg_temp"
os.makedirs(LOCAL_TEMP_DIR, exist_ok=True)

# 🌟 全局标签记忆缓存池 (存活时间: 5分钟)
GLOBAL_TAG_CACHE = {
    "caption_text": "",
    "expire_time": 0
}

# =================================================================
# 🤫 日志中枢配置：保持终端极度清爽，精准推送至管家大屏
# =================================================================
logging.basicConfig(
    level=logging.INFO,
    format='[%(asctime)s] %(message)s',
    datefmt='%H:%M:%S'
)
# 彻底屏蔽底层网络库和TG框架的开机连接唠叨
logging.getLogger("pyrogram").setLevel(logging.WARNING)
logging.getLogger("httpx").setLevel(logging.WARNING)

logger = logging.getLogger("TGEngine")
app = Client("tg_robust_leecher", api_id=API_ID, api_hash=API_HASH, bot_token=BOT_TOKEN)

# =================================================================
# 📡 跨进程上报中枢：完美咬合管家 /api/remote_log 接口
# =================================================================
async def notify_steward_log(msg, level="INFO"):
    """将战报打包投递至管家专设的上报接口，由面板呈现 💜 [引擎] 动态"""
    logger.info(msg)
    try:
        async with httpx.AsyncClient(timeout=3.0) as client:
            await client.post(
                f"{STEWARD_BASE_URL}/api/remote_log", 
                json={"level": level, "msg": msg}
            )
    except Exception:
        pass

# =================================================================
# 🧠 核心路由引擎：双轨咬合拆解（精准分离剧名与独立年份）
# =================================================================
def get_transfer_meta(raw_name, preset_tag="", msg_caption=""):
    base_name, ext = os.path.splitext(raw_name)
    if not ext: ext = ".mp4"
    
    folder_show_name = ""  
    file_show_name = ""    
    year_str = ""          
    ep_num = None
    target_season = 1 
    upload_folder = "自动转存入库"

    # -------------------------------------------------------------
    # 🎯 路线一：优先用预设车票锁定分类、剧名和季数
    # -------------------------------------------------------------
    route_source = preset_tag if preset_tag else msg_caption
    if route_source:
        cap_clean = re.sub(r'[\s_]+', ' ', route_source)
        
        # 严格对应盘内真实的物理监控大厅
        if "#动漫" in cap_clean or "#国漫" in cap_clean: upload_folder = "国漫"
        elif "#华语剧" in cap_clean or "#电视剧" in cap_clean: upload_folder = "华语剧"
        elif "#日韩剧" in cap_clean: upload_folder = "日韩剧"
        elif "#短剧" in cap_clean: upload_folder = "短剧"
        elif "#华语电影" in cap_clean or "#电影" in cap_clean: upload_folder = "华语电影"
        elif "#欧美电影" in cap_clean: upload_folder = "欧美电影"
        elif "#日韩电影" in cap_clean: upload_folder = "日韩电影"
        elif "#演唱会" in cap_clean: upload_folder = "演唱会"
            
        tags = re.findall(r'#([\w\u4e00-\u9fa5\s\(\)\d]+)', cap_clean)
        exclude_tags = {"动漫", "国漫", "电视剧", "华语剧", "日韩剧", "短剧", "电影", "华语电影", "欧美电影", "日韩电影", "演唱会", "S1", "S2", "S3", "S4"}
        
        for t in tags:
            tag_val = t.strip()
            if not any(tag_val.startswith(ex) for ex in exclude_tags) and not re.match(r'^S\d+$', tag_val, re.I):
                folder_show_name = tag_val  
                year_match = re.search(r'\s*\(?(\d{4})\)?$', tag_val)
                if year_match:
                    year_str = year_match.group(1) 
                    file_show_name = re.sub(r'\s*\(?\d{4}\)?$', '', tag_val).strip()
                else:
                    file_show_name = tag_val
                break
        
        season_match = re.search(r'(?:#)?(?:S|季|第)(\d+)(?:季)?', cap_clean, re.IGNORECASE)
        if season_match:
            s_val = int(season_match.group(1))
            if s_val < 20: target_season = s_val

    # -------------------------------------------------------------
    # 🔍 路线二：集数强制从视频原生配文中精准挖掘
    # -------------------------------------------------------------
    if msg_caption:
        msg_clean = re.sub(r'[\s_]+', ' ', msg_caption)
        ep_match = re.search(r'(?:EP|E|第)\s*(\d+)\s*(?:集)?', msg_clean, re.IGNORECASE)
        if ep_match:
            ep_num = int(ep_match.group(1))

    # -------------------------------------------------------------
    # 兜底雷达：若配文彻底没写集数，才从文件名读取
    # -------------------------------------------------------------
    cleaned_file = re.sub(r'[\s_]+', '.', base_name)
    if ep_num is None:
        file_match = re.search(r'^(.*?)\.?(?:EP|E|第|-)\.?(\d+)', cleaned_file, re.IGNORECASE)
        if file_match:
            ep_num = int(file_match.group(2))
            if not folder_show_name:
                raw_show = file_match.group(1).replace('.', ' ').strip()
                folder_show_name = raw_show
                year_match = re.search(r'\s*\(?(\d{4})\)?$', raw_show)
                if year_match:
                    year_str = year_match.group(1)
                    file_show_name = re.sub(r'\s*\(?\d{4}\)?$', '', raw_show).strip()
                else:
                    file_show_name = raw_show
        else:
            bare_match = re.search(r'^(\d+)(?:\.|cut|_)', cleaned_file, re.IGNORECASE)
            if bare_match: ep_num = int(bare_match.group(1))

    # 终态组装
    final_folder = folder_show_name if folder_show_name else "未知剧集"
    final_file = file_show_name if file_show_name else "未知剧集"
    final_ep = f"{ep_num:02d}" if ep_num is not None else "00"
    
    # 完美迎合收割正则切分：剧名.S季E集.年份.后缀
    if year_str:
        standard_name = f"{final_file}.S{target_season:02d}E{final_ep}.{year_str}{ext}"
    else:
        standard_name = f"{final_file}.S{target_season:02d}E{final_ep}{ext}"
    
    # 带着完整带括号年份的物理归档目录穿透自建
    dynamic_path = f"/family/177_cas/{upload_folder}/{final_folder}/Season {target_season}"
    return standard_name, dynamic_path

# =================================================================
# 🎯 阶段零：监听纯文本指令，开启 5 分钟全局车票锁
# =================================================================
@app.on_message(filters.text & ~filters.media)
async def handle_tag_preset(client, message):
    global GLOBAL_TAG_CACHE
    text = message.text or ""
    
    if "#" in text:
        GLOBAL_TAG_CACHE["caption_text"] = text
        GLOBAL_TAG_CACHE["expire_time"] = time.time() + 300
        
        await message.reply_text(
            f"🎯 **全域路由车票已成功刷入中枢内存 (5分钟内持续生效)**:\n"
            f"🏷️ 记忆指令: `{text}`\n\n"
            f"👉 **现在请直接回 Telegram 频道，长按勾选多集源文件批量转发给我**，将并发执行自动化改名直推落盘！"
        )
    else:
        await message.reply_text("💡 发送带 # 的规则口令即可锁定接下来的转发路由。例如：`#华语剧 #低智商犯罪 (2026) #S1`")

# =================================================================
# 🚀 主流程：捕获文件流与高弹性无阻碍推流
# =================================================================
@app.on_message(filters.video | filters.document)
async def handle_robust_transfer(client, message):
    media = message.video or message.document
    if not media: return
    
    raw_file_name = getattr(media, "file_name", None) or f"TG_Vid_{message.id}.mp4"
    
    # 提取全局缓存车票与单条原生配文
    global GLOBAL_TAG_CACHE
    preset_tag = ""
    if GLOBAL_TAG_CACHE["caption_text"] and time.time() < GLOBAL_TAG_CACHE["expire_time"]:
        preset_tag = GLOBAL_TAG_CACHE["caption_text"]
        
    msg_caption = message.caption or ""
    file_name, target_dir = get_transfer_meta(raw_file_name, preset_tag=preset_tag, msg_caption=msg_caption)
    
    local_file_path = os.path.join(LOCAL_TEMP_DIR, file_name)
    target_full_path = f"{target_dir}/{file_name}".replace("//", "/")
    size_mb = media.file_size / (1024 * 1024)
    
    status_msg = await message.reply_text(
        f"⚡ 指令接收 ➔ `{file_name}`\n"
        f"📁 投递航线: `{target_dir}`\n"
        f"🔍 正在扫掠物理本地是否存在现成实体..."
    )

    # -------------------------------------------------------------
    # 🛡️ 阶段一：智能本地雷达感知与断点续传保护
    # -------------------------------------------------------------
    file_exists = os.path.exists(local_file_path)
    is_fully_cached = file_exists and os.path.getsize(local_file_path) == media.file_size

    if is_fully_cached:
        await status_msg.edit_text(
            f"⚡ 命中本地极度完整缓存！\n"
            f"文件: `{file_name}` ({size_mb:.1f} MB) 现成可用。\n"
            f"🚀 智能跳过远端拉取阶段，当场拉起网关直推管道！"
        )
        await notify_steward_log(f"命中完整缓存: {file_name}，直接启动空中直推")
    else:
        await status_msg.edit_text(
            f"📥 [阶段一：极速稳健落盘]\n"
            f"目标: `{file_name}` ({size_mb:.1f} MB)\n"
            f"正建立底层连接，全速写入暂存闪存池..."
        )
        await notify_steward_log(f"开始下载: {file_name} ({size_mb:.1f}MB)")
        
        async def termux_progress(current, total):
            percent = current * 100 / total
            cur_mb = current / (1024 * 1024)
            tot_mb = total / (1024 * 1024)
            print(f"📥 稳健落盘中... {cur_mb:.1f}MB / {tot_mb:.1f}MB ({percent:.1f}%)", end="\r")

        start_time = time.time()
        try:
            print("\n")
            await client.download_media(message, file_name=local_file_path, progress=termux_progress)
            print("\n")
            elapsed = int(time.time() - start_time)
            await status_msg.edit_text(
                f"✅ [阶段一完成] 本地落盘成功，耗时: {elapsed}s\n"
                f"🚀 [阶段二：空中直推] 正在无感打通 Openlist 鉴权挂载网关..."
            )
            await notify_steward_log(f"下载完成: {file_name}，耗时 {elapsed}s，启动全速推流")
        except Exception as down_err:
            await status_msg.edit_text(f"❌ 下载阶段底层链路崩溃: {down_err}")
            await notify_steward_log(f"下载异常中断: {file_name} ({down_err})", level="ERROR")
            return

    # -------------------------------------------------------------
    # 📤 阶段二：流式异步生成器直写 (内存消耗极致低)
    # -------------------------------------------------------------
    put_url = f"{OLIST_URL}/api/fs/put"
    headers = {
        "Authorization": OLIST_TOKEN,
        "File-Path": quote(target_full_path),
        "Content-Length": str(os.path.getsize(local_file_path)),
        "Content-Type": "application/octet-stream"
    }

    upload_success = False
    try:
        async def file_iterable():
            total_size = os.path.getsize(local_file_path)
            uploaded_size = 0
            with open(local_file_path, "rb") as f:
                while True:
                    chunk = f.read(1024 * 1024)
                    if not chunk: break
                    uploaded_size += len(chunk)
                    percent = uploaded_size * 100 / total_size
                    cur_mb = uploaded_size / (1024 * 1024)
                    tot_mb = total_size / (1024 * 1024)
                    print(f"🚀 极速空中直推中... {cur_mb:.1f}MB / {tot_mb:.1f}MB ({percent:.1f}%)", end="\r")
                    yield chunk

        async with httpx.AsyncClient(timeout=None) as h_client:
            print("\n")
            resp = await h_client.put(put_url, content=file_iterable(), headers=headers)
            print("\n")
            
        res_data = resp.json()
        if res_data.get("code") == 200:
            upload_success = True
            await status_msg.edit_text(
                f"🎉 全域跨界转存完美竣工！\n"
                f"📁 云端入口锚点: `{target_full_path}`\n"
                f"*(伴生凭证已吐出，静候 auto189.py 巡回接力归档)*\n"
                f"🧹 正在执行本地暂存空间的物理自毁释放..."
            )
            # 🌟 终极纯净态：只往 UI 面板推一句落地成功的文本简报即可！
            await notify_steward_log(f"直推落地成功: {file_name} 已进入监控大厅待收割")
            
            # ⚠️ 之前写在这里的 await notify_steward_sync(target_dir) 已经彻底删掉！绝不提前触发生成！
        else:
            err_msg = res_data.get('message', res_data)
            await status_msg.edit_text(f"❌ 穿透 Openlist 失败。错误: {err_msg}")
            await notify_steward_log(f"直推网关拒绝: {file_name} ({err_msg})", level="ERROR")
            
    except Exception as up_err:
        await status_msg.edit_text(f"💥 推流阶段网络连接断流，本地原件安全保留。错误: {up_err}")
        await notify_steward_log(f"直推断流中断: {file_name} ({up_err})", level="ERROR")

    # -------------------------------------------------------------
    # 🧹 阶段三：无痕安全收尾
    # -------------------------------------------------------------
    if upload_success:
        try:
            if os.path.exists(local_file_path):
                os.remove(local_file_path)
                logger.info(f"🗑️ 已彻底抹除本地暂存死缓存: {local_file_path}")
        except Exception as clean_err:
            logger.info(f"⚠️ 临时垃圾物理抹除失败: {clean_err}")

if __name__ == "__main__":
    logger.info("🤖 终极私人定制版极速转存引擎 (含后台管家无缝通信架构) 已就位！")
    logger.info("💡 请随时向我发送带 # 的指令预设车票，体验酣畅淋漓的批量无脑并发转存...")
    app.run()
```
