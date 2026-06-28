---

title: "Termux之qBittorrent"

author: "xxsky"

type: "posts"

date: 2026-06-23T21:09:07+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

# 摘要

<!--more-->

### 一、安装qBittorrent

1.安装 qBittorrent (无头版)
打开 Termux，先确保软件包列表是最新的，然后执行安装命令：
```
pkg update -y
pkg install qbittorrent -y
```
2.首次启动与获取临时密码

注意： 新版本的 qBittorrent 已经取消了默认的 adminadmin 密码，首次运行会在屏幕上生成一个随机临时密码。

2.1在 Termux 中直接输入以下命令进行首次启动：
```
qbittorrent-nox
```
2.2屏幕上会弹出一长串英文的免责声明。按下键盘上的 y 然后回车同意。

2.3接着，屏幕上会刷出几行日志，请仔细寻找包含以下字眼的两行：

The WebUI administrator username is: admin
The WebUI administrator password was set to: (这里会有一串随机字符)

2.4把这串随机字符复制下来，这就是你第一次登录网页端的密码。

注：此时 qBittorrent 已经在前台运行了，千万不要把这个 Termux 窗口关掉。

3.网页端登录与基础设置

3.1打开手机浏览器，访问：http://127.0.0.1:8080

3.2用户名输入 admin，密码粘贴你刚才复制的那串随机字符，点击 Login。

3.3改中文： 点击顶部的齿轮图标 (Options) -> 左侧选择 Web UI -> Language 选 简体中文 -> 划到最底部点击 Save。

3.4改密码： 网页变成中文后，再次进入 设置 -> Web UI -> 在“验证”栏目下，把临时的随机密码改成你自己容易记住的密码，再次点击底部 保存。

4.用 PM2 接管后台运行

现在我们已经配置好了，需要让它在后台默默运行，不占用我们 Termux 的前台窗口。

4.1回到 Termux 界面。

4.2按下 Ctrl + C（按住输入法栏上的 CTRL，再按 C），此时会提示进程已结束，停止刚才的前台运行。

4.3PM2 启动它：
```
pm2 start qbittorrent-nox --name "qbt"
```
4.4保存 PM2 的当前状态，确保下次 Termux 重启时它也能跟着启动：
```
pm2 save
```
5.安装报错解决方法

5.1这是一个非常经典的“包名陷阱”！别担心，你的网络和镜像源现在完全没问题（提示里已经明确写着 [*] https://mirrors... : ok）。

报错 Unable to locate package qbittorrent 的真正原因其实是包名没对上：

* 名字差了几个字母：在 Termux 这种纯命令行环境里，并不存在带桌面图形界面的普通版 qbittorrent。官方提供的是专门用于服务器后台挂机的无头版本，它的确切包名叫做 qbittorrent-nox（nox 代表 No X-server，即无图形环境）。

* 需要扩展软件源：因为这个软件底层依赖了一些 Qt 的核心库，而这些库通常存放在 Termux 的 X11 扩展仓库中。如果只在默认的基础仓库（main）里找，是找不到的。

请按顺序直接复制、粘贴这三行命令并执行，就能完美解决：
```
# 1. 安装 x11 扩展仓库（如果提示已安装则无视）
pkg install x11-repo -y

# 2. 刷新一下包列表，让系统读取到新仓库的内容
pkg update -y

# 3. 安装正确的包名
pkg install qbittorrent-nox -y
```

安装完成后，你就可以直接输入 qbittorrent-nox 进行首次启动并获取那个关键的随机密码了。

二、Termux程序与代理的问题

1.由于下载不需要代理，解决方法在flclash中把Termux排除在外也就termux绕行。然后需要代理的脚本通过下面方式启动，直接指向代理软件端口。比如：
```
http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 no_proxy="localhost,127.0.0.1" pm2 start 你的脚本名.py --name "自动采集"
```
2.termux按需开启与关闭代理

终极解法：制作 proxy 和 unproxy 快捷开关
只要在 Termux 里配置一次，以后你只需要敲五个字母，就能随时切换网络。

第一步：打开配置文件
在 Termux 里输入以下命令，打开你的基础环境变量配置文件（使用内置的 nano 编辑器）：
```
nano ~/.bashrc
```
第二步：写入开关代码
在打开的界面最下面，粘贴下面这四行代码（假设你的 Flclash 本地端口是 7890，如果是别的请自己改一下数字）：

```
# 开启代理的快捷键
alias proxy="export http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890; echo '代理已开启 🟢'"

# 关闭代理的快捷键
alias unproxy="unset http_proxy https_proxy; echo '代理已关闭 🔴'"
```
第三步：保存并生效

2.1按 Ctrl + O（字母 O），然后按回车键保存。

2.2按 Ctrl + X 退出编辑器。

2.3输入 source ~/.bashrc 让刚才的修改立刻生效。

以后你怎么操作？（日常使用场景）
现在你的 Termux 已经拥有了魔法开关。平时它默认是直连的（qBittorrent 正常下 BT）。

场景 1：你需要安装国外的 Python 包或者用 Git 拉取国外的脚本代码
你只需要在敲安装命令前，先输入 proxy。
```
~ $ proxy
代理已开启 🟢
~ $ pip install you-get   # 此时安装走的是翻墙网络，速度飞快，且不影响后台的 qBT
```
场景 2：安装完脚本了，你想恢复原本的直连状态测试
你只需要输入 unproxy。
```
~ $ unproxy
代理已关闭 🔴
~ $ python test.py      # 此时又恢复了真实的局域网环境
```
为什么这招这么好用？
因为这种环境变量（export）的作用域是“仅限当前终端窗口”。
当你敲下 proxy 时，只有你当前正在操作的这个前台窗口翻墙了。你之前用 pm2 扔到后台挂机的 qBittorrent 属于另外独立的进程，它根本不知道你在这个窗口里开了代理，所以它依然在老老实实地用真实宽带下电影。

通过 PM2 控制后台自动化脚本走代理，通过 proxy 开关控制前台手动安装走代理，这套“内外隔离”的架构跑通后，你的 Termux 服务器网络环境就彻底毕业了。

3.把需要走代理的脚本通过传递--update-env参数达到开启代理端口的方法

“手动一个一个加”才是真正最牛、最安全的解法。因为你的 openlist、nginx、vscode、cloudflared、aria2 本来就跑得好好的，咱们根本不需要去动它们，更不需要删它们。

PM2 其实自带一个“无损注入”的神器参数：--update-env。它可以让你在不删除进程、不知道原先路径的情况下，直接把代理强行塞给指定的几个程序。

咱们现在就用你最踏实的手动一键注入法，只对需要代理的 4 个程序精准开刀：

第一步：在当前窗口拉起代理天网
先把代理变量在当前的黑框框里临时生效：
```
export http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890 no_proxy="localhost,127.0.0.1"
```
第二步：一个一个精准注入（无损重启）
直接利用 --update-env 参数，把刚才的代理变量强行灌进这 4 个需要翻墙的进程里。直接顺次执行这四行：
```
pm2 restart emby --update-env
pm2 restart tgms --update-env
pm2 restart auto189 --update-env
pm2 restart autotg --update-env
```
第三步：保存战果
```
pm2 save
```
为什么这个“手动注入法”最完美？
3.1绝对安全：你最担心的 openlist、nginx、vscode 还有下载器们，从头到尾连动都没动过，依然保持它们原本最完美的直连状态。

3.2免改路径：PM2 会在原地重启 emby 或 auto189，并把代理变量像打针一样注入进去，不需要你输入任何长串的绝对路径。

弄完之后，你可以敲一下 pm2 describe auto189，拉到最底下的 Environment variables 这一块，你会看到 http_proxy 已经带着 7890 稳稳当当地写进去了。

这次绝对不画饼了，这 5 行命令敲完，这套网络折腾就真正安全落地了！

3.3怎么看真正的代理软件端口？
要揪出它到底有没有成功带上 Flclash 的 7890 端口，不能看 describe 的这个小表格，得看它的完整环境报告。

请在终端里输入这句命令（5 是你 auto189 的进程 ID）：
```
pm2 env 5
```
这会打印出密密麻麻的一大篇底层信息。你直接往上翻，在里面找 http_proxy 和 https_proxy 这两行。

* 如果能看到 http://127.0.0.1:7890，说明它已经稳稳当当地带上 Flclash 的车票在跑了。

* 如果里面依然找不到，说明刚才的注入没生效。你就把刚才那三步（先 export，再 pm2 restart auto189 --update-env，最后 pm2 save）连着敲一遍，接着再用 pm2 env 5 就能亲眼看到它了。

### 三、qbt_delivery.py

1.网页端bt的相关设置

1.1左边栏里的分类设置与autotg.py一样，也是分类目录映射关系。

如：国漫、华语剧、欧美剧、华语电影、欧美电影等。

1.2左边栏里的标签对应剧名文件夹里的版本号以区别

如：HDR、SDR、HQ、HFR、DV等。

1.3bt的设置选项修改

* 高级：

  libtorrent 相关
  * 验证 HTTPS tracker 证书： (?)	
  * 服务器端请求伪造（SSRF）攻击缓解： (?)

  取消勾选

* BitTorrent：

  隐私
  *  启用 DHT (去中心化网络) 以找到更多用户
  *  启用用户交换 (PeX) 以找到更多用户
  *  启用本地用户发现以找到更多用户

  取消勾选

* RSS：

  RSS Torrent 自动下载器

  勾选

1.4RSS订阅

* 在bitTorrent种子发布网站获取RSS链接
* 添加到RSS订阅里，这里会获取到Torrent列表
* 在右边的RSS下载器里添加下载规则

* 电视剧分集下载规则
  * 添加规则名称如莫离追更
  * 规则定义里勾选使用正则表达式
    
    必须包含：The First Jasmine.*E(3[1-9]|[4-9]\d)

    （剧名.E31-99）

    或者是：The First Jasmine.*E([3-9]\d)

    (这就代表 3 开头到 9 开头的所有两位数，也就是 30 到 99 集，把 30 也包进去了)。

    指定分类如华语剧

    添加标签如HFR

    对以下订阅源应用规则勾选订阅源

1.5配置 qBittorrent 接力开关

* 代码搞定后，去 qBittorrent 的网页端，只需要做这一个动作：

* 点击网页端顶部的 齿轮（选项卡）。

* 左侧点击 下载 (Downloads)，拉到页面最底下。

* 勾选 “Torrent 完成时运行外部程序”。

* 在文本框里，完整复制并粘贴这行命令（注意后面加入了 %G 标签参数）：
```
python /data/data/com.termux/files/home/189py/qbt_delivery.py "%N" "%F" "%L" "%G"
```
* 点击最下方的 “保存”。

2.脚本
```
import os
import re
import sys
import time
import json
import asyncio
from urllib.parse import quote
import httpx

# =================================================================
# ⚙️ 核心网关与路径映射配置
# =================================================================
OLIST_URL = "http://127.0.0.1:5244"
OLIST_TOKEN = "openlist-a87614da-32dd-4b80-9150-6447de823da8f33x53ymkrx0aPKG0HUcsFHmjFRYTKFhSADLRhoQLkXa7ogaiByhWRNEXCjpblp9"
STEWARD_BASE_URL = "http://127.0.0.1:5000" 
TMDB_API_KEY = "9c88e18e43543c8ff195c631aaa0d2fa"

STAGING_BASE_DIR = "/storage/emulated/0/Download/189cas"
TG_SETTINGS_DB = os.path.join(os.path.dirname(os.path.abspath(__file__)), "tg_settings.json")

# 🔔 【新增配置】独立推送网关与局部代理
PUSHPLUS_TOKEN = "" 
TG_BOT_TOKEN = "7548615667:AAHn0ls4aBPKBPI2-gpwykwVdEKd0ywOlsc"
TG_CHAT_ID = "-1002906711199"
# 🎯 局部代理：由于 Termux 被直连，这里单独给 TG 和 TMDB 指定 FlClash 的本地代理端口。
# 注意：核对你的 FlClash 设置里的 HTTP 端口，通常是 7890 或 8080
PROXY_URL = "http://127.0.0.1:8080" 

def get_mount_root():
    if os.path.exists(TG_SETTINGS_DB):
        try:
            with open(TG_SETTINGS_DB, "r", encoding="utf-8") as f:
                return json.load(f).get("mount_root", "/f180/177_cas")
        except: pass
    return "/f180/177_cas"

async def notify_steward_log(msg, level="INFO"):
    """安全桥梁：走打更人本地Web中枢上报，规避SQLite数据库锁死"""
    print(f"[{level}] {msg}")
    try:
        # 本地汇报，绝对不走代理
        async with httpx.AsyncClient(timeout=2.0) as client:
            await client.post(f"{STEWARD_BASE_URL}/api/remote_log", json={"level": level, "msg": msg})
    except Exception: pass

async def send_push_notification(title, content):
    """独立的微信/TG消息推送通道，只推送核心成功/失败结果"""
    # 1. 微信 PushPlus 推送 (国内网络，无需代理)
    if PUSHPLUS_TOKEN:
        try:
            url = "http://www.pushplus.plus/send"
            data = {"token": PUSHPLUS_TOKEN, "title": title, "content": content, "template": "html"}
            async with httpx.AsyncClient(timeout=10.0) as client:
                await client.post(url, json=data)
        except: pass
        
    # 2. TG 推送 (需要翻墙，注入局部代理)
    if TG_BOT_TOKEN and TG_CHAT_ID:
        try:
            # 💡 核心修复：TG 的 HTML 模式不支持 <br> 标签，将其替换为标准的 \n 换行符
            tg_content = content.replace("<br>", "\n")
            url = f"https://api.telegram.org/bot{TG_BOT_TOKEN}/sendMessage"
            data = {"chat_id": TG_CHAT_ID, "text": f"{title}\n\n{tg_content}", "parse_mode": "HTML"}
            # 这里挂载代理，让 TG 消息强行穿过 FlClash 发出去
            async with httpx.AsyncClient(timeout=10.0, proxy=PROXY_URL if PROXY_URL else None) as client:
                resp = await client.post(url, json=data)
                if resp.status_code != 200:
                    await notify_steward_log(f"⚠️ [TG推送失败] 状态码: {resp.status_code}, 详情: {resp.text}", level="WARNING")
        except Exception as e:
            await notify_steward_log(f"⚠️ [TG推送异常] 网络或代理出错: {e}", level="WARNING")

# =================================================================
# 🧠 智能属性提取引擎
# =================================================================
def get_hdr_sdr_tag(torrent_name):
    t = torrent_name.upper()
    if re.search(r'(DV|DOVI|DOLBY VISION|HDR10\+|HDR10|HDR)', t): return "HDR"
    if re.search(r'(SDR)', t): return "SDR"
    return ""

def extract_pure_episode(text, drama_anchor=None):
    if drama_anchor:
        try: text = re.compile(re.escape(drama_anchor), re.IGNORECASE).sub(' ', text)
        except: pass
    m = re.search(r'(?i)E(?:P)?0*(\d+)', text)
    if m: return int(m.group(1))
    m = re.search(r'第\s*(\d+)\s*[集话期更]', text)
    if m: return int(m.group(1))
    return None

async def fetch_tmdb_year(title):
    if not TMDB_API_KEY: return time.strftime("%Y")
    clean_q = re.sub(r'S\d+$|\s+\d+$', '', title).strip()
    url = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={quote(clean_q)}&page=1"
    try:
        # 恢复你原来正常的直连模式，去掉了 proxy 参数
        async with httpx.AsyncClient(timeout=5.0) as client:
            res = await client.get(url)
            results = res.json().get("results")
            if results:
                item = results[0]
                year = (item.get("first_air_date") or item.get("release_date") or "")[:4]
                if year: return year
    except: pass
    return time.strftime("%Y")

# =================================================================
# 🛡️ 猎犬级 CAS 强力镜像下发引擎 (死磕到底)
# =================================================================
async def bg_fetch_cas_task(cas_target_full, final_cas_path, sub_path, cas_file_name):
    await notify_steward_log(f"🔎 [CAS嗅探启动] 正在捕获云端特征码: `{cas_file_name}`")
    await asyncio.sleep(15) 
    get_info_url = f"{OLIST_URL}/api/fs/get"
    headers_get = {"Authorization": OLIST_TOKEN, "Content-Type": "application/json"}
    
    max_attempts = 45 
    cas_downloaded = False
    
    for attempt in range(max_attempts):
        try:
            # 本地获取 CAS，走直连
            async with httpx.AsyncClient(timeout=20.0, follow_redirects=True) as client:
                resp = await client.post(get_info_url, json={"path": cas_target_full}, headers=headers_get)
                resp_data = resp.json()
                
                if resp_data.get("code") == 200:
                    raw_url = resp_data["data"]["raw_url"]
                    resp_dl = await client.get(raw_url)
                    
                    if resp_dl.status_code == 200:
                        temp_cas_path = final_cas_path + ".tmp"
                        with open(temp_cas_path, "wb") as f:
                            f.write(resp_dl.content)
                        os.rename(temp_cas_path, final_cas_path)
                        
                        await notify_steward_log(f"🔗 [CAS镜像成功] ➔ `{sub_path}/{cas_file_name}` (耗时: {(attempt+1) * 10 + 15}秒)")
                        cas_downloaded = True
                        break
        except: pass
        await asyncio.sleep(10)
        
    if not cas_downloaded:
        await notify_steward_log(f"🚨🚨 [CAS拉取彻底失败] 降级放弃！目标: `{cas_file_name}`", level="CRITICAL")

# =================================================================
# 🚀 核心装卸接力赛
# =================================================================
async def main():
    if len(sys.argv) < 4:
        print("⚠️ 语法规范: python qbt_delivery.py [种子名] [保存路径] [分类] [可选:标签]")
        return

    torrent_name = sys.argv[1]
    save_path = sys.argv[2]
    category = sys.argv[3]
    custom_tag = sys.argv[4].strip() if len(sys.argv) > 4 else ""

    # 1. 基础剧名洗白
    pure_drama_name = re.sub(r'^\[.*?\]|\(.*?\)', '', torrent_name).strip()
    pure_drama_name = pure_drama_name.split('.')[0].split(' ')[0].strip()
    pure_drama_name = re.sub(r'第.*?季|S\d+', '', pure_drama_name, flags=re.IGNORECASE).strip()

    m_season = re.search(r'(?i)S(\d+)', torrent_name)
    folder_season = int(m_season.group(1)) if m_season else 1
    
    year = await fetch_tmdb_year(pure_drama_name)
    current_mount = get_mount_root()
    is_movie = "电影" in category or category in ["演唱会", "纪录片"]

    # 2. 剧名文件夹标准化决策 (坚如磐石的地基 + 自定义后缀)
    clean_base = pure_drama_name.replace(" ", ".")
    base_folder = f"{clean_base} ({year})" if year else clean_base
    
    if custom_tag:
        folder_name = f"{base_folder} {custom_tag}"
    else:
        folder_name = base_folder

    # 3. 🎯 核心升级：扫描捕获所有的视频实体文件
    video_tasks = []
    if os.path.isdir(save_path):
        for root, dirs, files in os.walk(save_path):
            for f in files:
                if f.lower().endswith(('.mp4', '.mkv', '.ts', '.flv')):
                    video_tasks.append((os.path.join(root, f), f))
        # 排序：确保第1集先传
        video_tasks.sort(key=lambda x: x[1])
    else:
        if save_path.lower().endswith(('.mp4', '.mkv', '.ts', '.flv')):
            video_tasks.append((save_path, os.path.basename(save_path)))

    if not video_tasks:
        await notify_steward_log(f"⚠️ [交接终止] 未在路径 `{save_path}` 下发现任何有效视频文件。")
        return

    await notify_steward_log(f"📦 [大包交接启动] 监测到当前种子共包含 {len(video_tasks)} 个剧集文件，开启串行排队搬运...")

    # 4. 🔄 串行循环：一集一集扎实推进，稳过新手考核
    for actual_video_path, actual_video_name in video_tasks:
        if not os.path.exists(actual_video_path):
            continue
            
        total_bytes = os.path.getsize(actual_video_path)
        
        ep_num = extract_pure_episode(actual_video_name, drama_anchor=pure_drama_name)
        if ep_num is None:
            ep_num = extract_pure_episode(torrent_name, drama_anchor=pure_drama_name)

        if is_movie:
            target_dir = f"{current_mount}/{category}/{folder_name}"
        else:
            if ep_num is None:
                await notify_steward_log(f"⚠️ [单集跳过] 文件 `{actual_video_name}` 无法识别集数，做跳过处理。", level="WARNING")
                continue
            target_dir = f"{current_mount}/{category}/{folder_name}/Season {folder_season}"

        target_full = f"{target_dir}/{actual_video_name}".replace("//", "/")
        
        # ========================================================
        # 🛑 核心新增：智能查重，已传过的坚决跳过
        # ========================================================
        cas_file_name = f"{actual_video_name}.cas"
        cas_target_full = f"{target_full}.cas"
        sub_path = target_dir.replace(current_mount, "", 1)
        local_cas_dir = f"{STAGING_BASE_DIR}{sub_path}".replace("//", "/")
        final_cas_path = os.path.join(local_cas_dir, cas_file_name)
        
        if os.path.exists(final_cas_path):
            await notify_steward_log(f"⏭️ [智能跳过] 侦测到本地已存在镜像 `{cas_file_name}`，该集已上传过，执行跳过处理。")
            continue 
        # ========================================================

        await notify_steward_log(f"🚚 [队列流转] 正在原样运送: `{actual_video_name}` ➔ 云端")
        
        put_url = f"{OLIST_URL}/api/fs/put"
        headers = {
            "Authorization": OLIST_TOKEN,
            "File-Path": quote(target_full),
            "Content-Length": str(total_bytes),
            "Content-Type": "application/octet-stream"
        }

        # ========================================================
        # 🛡️ 核心新增：带推送的异步后台重试机制 (最多试5次)
        # ========================================================
        max_upload_retries = 5
        upload_success = False
        
        for attempt in range(max_upload_retries):
            try:
                with open(actual_video_path, "rb") as f_in:
                    async def file_iter():
                        while True:
                            chunk = f_in.read(2 * 1024 * 1024)
                            if not chunk: break
                            yield chunk

                    async with httpx.AsyncClient(timeout=httpx.Timeout(connect=10.0, read=1800.0, write=60.0, pool=None)) as client:
                        resp = await client.put(put_url, content=file_iter(), headers=headers)
                        
                if resp.json().get("code") == 200:
                    upload_success = True
                    os.makedirs(local_cas_dir, exist_ok=True)
                    
                    # 🎉 【关键修改】：使用 create_task 让推送在后台独立运行，绝不卡死主线程
                    success_msg = f"<b>{actual_video_name}</b><br>在第 {attempt + 1} 次尝试后成功传至云端！"
                    asyncio.create_task(send_push_notification("✅ PT入库成功", success_msg))
                    
                    # 就算上面的推送失败了，也绝对不影响下面拉取 CAS
                    await bg_fetch_cas_task(cas_target_full, final_cas_path, sub_path, cas_file_name)
                    break 
                else:
                    err_json = resp.json().get('message')
                    await notify_steward_log(f"⚠️ [上传遭拒] `{actual_video_name}` 第 {attempt+1} 次失败: {err_json}", level="WARNING")
                    
            except Exception as e:
                await notify_steward_log(f"⚠️ [网络异常] `{actual_video_name}` 第 {attempt+1} 次崩溃: {e}", level="WARNING")
                
            if attempt < max_upload_retries - 1:
                wait_sec = 2 ** (attempt + 1)
                await notify_steward_log(f"⏳ 正在后台打盹等待 {wait_sec} 秒后进行第 {attempt+2} 次重试...")
                await asyncio.sleep(wait_sec)

        if not upload_success:
            err_msg = f"<b>{actual_video_name}</b><br>经过 {max_upload_retries} 次重试依旧崩溃！请速查日志，疑似网盘断开或网络阻断。"
            await notify_steward_log(f"❌ [彻底失败] {err_msg}", level="ERROR")
            # 失败推送同样丢进后台
            asyncio.create_task(send_push_notification("🚨 PT入库彻底失败", err_msg))
        # ========================================================
            
    await notify_steward_log(f"🏁 [大包搬运完毕] 该种子的所有有效增量视频文件已全部完成云端接力！")

if __name__ == "__main__":
    asyncio.run(main())
```



