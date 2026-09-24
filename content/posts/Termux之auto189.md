---

title: "Termux之auto189"

author: "xxsky"

type: "posts"

date: 2026-08-03T16:11:48+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

订阅转存收割

<!--more-->

### 一、初始化 Termux 环境

1.更新系统并安装基础工具
```
pkg update -y && pkg upgrade -y
```
2.安装Python与核心编译工具
```
pkg install python clang make libffi -y
pip install requests pycryptodome python-dotenv schedule
```
（如果提示 [Y/n]，直接敲 y 回车)

3.安装必要的 Python 引擎库
```
pip install flask requests
```
4.库出错
这个 ModuleNotFoundError: No module named ‘Crypto’ 是 Python 界一个极其经典且烦人的“坑”，几乎所有第一次折腾加密库的人都会踩中
原因很简单：代码里调用的名字叫 Crypto，但它对应的现代库其实叫 pycryptodome。有时候 Python 环境会犯傻，或者之前残留了一些废弃的老库（比如老古董 pycrypto），导致它“认错人”了
直接复制下面这两行命令，依次在 Termux 里回车（第一行可能会提示未找到某些库，不用管，直接让它执行完）：
```
pip uninstall crypto pycrypto pycryptodome -y
pip install pycryptodome
```
现在冒出来的这个 No module named ‘schedule’ 报错，是因为咱们最开始那步批量装库的时候，可能因为网络波动中断了，导致 schedule（用来做定时任务的库）没装上。

咱们现在就玩“打地鼠”，它缺啥咱们补啥！为了防止等会儿它再报别的库没装，咱们干脆把脚本需要的剩下几个第三方库一次性全补齐。
直接复制这行回车：
```
pip install schedule python-dotenv requests
```

5.其它补充

在 Termux 中安装第三方库经常需要现场编译，因此 clang（C/C++ 编译器）是必不可少的。
```
pkg install python -y
pkg install clang make cmake -y
```
安装关键的底层系统运行库（最容易报错的环节）
结合 auto_189（通常涉及 requests, 数据解析, 可能涉及 RSA/AES 加密登录）和 cas_server（认证服务端，通常需要密码学库、JWT 生成等），你需要安装以下底层依赖：
```
# 1. 密码学与安全连接底层库 (为 cryptography, pyOpenSSL, bcrypt 等库准备)
pkg install libffi openssl -y

# 2. 网页/XML 解析底层库 (为 lxml, beautifulsoup4 等库准备)
pkg install libxml2 libxslt -y

# 3. 图像处理底层库 (如果脚本包含验证码识别，通常需要 Pillow 库)
pkg install libjpeg-turbo zlib freetype -y
```
升级 pip 并安装 Python 依赖
底层库安装完毕后，就可以使用 pip 安装 Python 的第三方库了。建议先升级 pip：
```
python -m pip install --upgrade pip
```
手动安装以下核心库：
```
# 基础网络与服务端库 (如 Flask, FastAPI, Requests)
pip install requests flask uvicorn

# 加解密与认证库 (CAS Server 必备)
pip install cryptography pyjwt

# 网页解析 (Auto 189 必备)
pip install beautifulsoup4 lxml
```
注：在 Termux 中安装 cryptography 或 lxml 可能会花费较长时间（几分钟），因为它正在调用 clang 进行本地编译，请耐心等待，不要中断。

# 3.拼音库
```
pip install pypinyin
```

### 二、自动转存auto189

1.建立专属工作台与配置文件
```
# 1. 创建专属文件夹并进去

mkdir -p ~/189py/db
cd ~/189py

# 2. 创建环境变量文件并编辑
nano sys.env
```
执行完 nano sys.env 后，屏幕会变成黑底白字的编辑器。把下面这段内容修改成你自己的真实信息后，粘贴进去（注意等号两边不要有空格）：
```
# 你的天翼云盘账号和密码
ENV_189_CLIENT_ID=17707372266
ENV_189_CLIENT_SECRET=1127&xxskY
# 你的 TG 机器人配置
ENV_TG_BOT_TOKEN=7548615667:AAHn0ls4aBPKBPI2-gpwykwVdEKd0ywOlsc
ENV_TG_ADMIN_USER_ID=-1002906711199

# 新增这行
ENV_TMDB_API_KEY=9c88e18e43543c8ff195c631aaa0d2fa
```
填完后，按 Ctrl + O（字母O），回车保存；然后按 Ctrl + X 退出。

### 四、添加机器人指令
1.打开BotFather机器人

2.发指令/setcommands

3.选择自己的机器人

4.粘贴如下内容：
```
sub - 📥 [订阅/绑定] 绑定外部链接追剧
dropbox - 🚜 [投递/本地扫描/扫箱子] 洗名并入库本地CAS文件
harvest - 🚜 [收割/处理/添加] 洗名并入库云端CAS文件
feed - 📡 [动态/广场] 订阅中心最新情报
search - 🔍 [搜 关键词] 穿甲雷达搜索
check - 🔍 [查 剧名] 剧名查找
author - 🕵️‍♂️ [查作者\查人] 大佬真实时间线
info - 🎞 [查剧\信息] TMDB影视资料
refresh- 🔄 [刷新\入库] 刷新入库某剧
sync - 🔄 [同步订阅] 强制检查所有更新
list - 📋 [列表] 查看当前追剧清单
ascan - ✔️ [开启自动收割] 自动收割扫描
sscan - ⭕️ [关闭自动收割] 关闭收割扫描
asub - ✅ [开启订阅检查] 开启订阅检查
ssub - ❎ [关闭订阅检查] 关闭订阅检查
ldir - 🔍 [查目录] 查看收割家庭云目录
adir - ➕ [加目录] 增加收割家庭云目录
ddir - ❌ [删目录] 删除收割家庭云目录
listcopy - 🔍 [查个人云] 查看收割个人云目录
addcopy - ➕ [加个人云] 增加收割个人云目录
rmcopy - ❌ [删个人云] 删除收割个人云目录
hsub - ➕ [加库] 增加收割入库记录
dsub - ❌ [删库] 删除收割入库记录
lsub - 🔍 [查库] 查看收割入库清单
mode - ⚙️ [设置模式] 切换189管家STRM生成模式(A/B/C)
recloud - ☁️ [恢复云端] 触发云端增量生成STRM文件
reall - ☁️ [全库重建] 重建媒体库
dict - 🔍 [查字典] 当前系统动态路由字典
adddict - ➕ [加字典] 增加动态路由字典
deldict - ❌ [删字典] 删除动态路由字典
config - 🔍 [查配置] 当前系统核心配置
setconfig - ⚙️ [设配置] 修改系统核心配置
cancel - 🚫 [取消] 解除监控任务并清理关联记忆
139hv - ❇️ [139收割 139加工] 触发5255端口openlist生成cas
sync139 - ☁️ [139同步] 触发139移动云盘专属STRM生成
scan139 - ♻️ [139全库] 全域扫荡雷达专属STRM生成
mode139 - ⚙️ [139设置模式] 切换139管家STRM生成模式(A/B/C)
```

### 四、auto189.py脚本

```
import os
import sys
import json
import time
import requests
import urllib3
# 🚨 终极核武器：直接在底层网络库中物理阉割 IPv6，防止天翼云 IP 漂移拦截
urllib3.util.connection.HAS_IPV6 = False
import re
import subprocess
import random
import socket
import sqlite3
import shutil
from urllib import parse
from Crypto.Cipher import PKCS1_v1_5 as Cipher_pksc1_v1_5
from Crypto.PublicKey import RSA
import logging
import schedule
from dotenv import load_dotenv
from datetime import datetime
import threading
from flask import Flask, request

# ==========================================
# ⚙️ 全局核心变量与服务配置 (统一在这里修改，告别死固定地址)
# ==========================================
# --- 🔗 内部微服务与 API 地址 ---
API_5000_URL = "http://127.0.0.1:5000"  # 189管家服务地址
API_5244_URL = "http://127.0.0.1:5244"  # OpenList 本地地址
OLIST_USER = "admin"                    # OpenList 账号
OLIST_PASS = "xxsky1127"                # OpenList 密码
TRIGGER_PORT = 5555                     # 本地监听触发端口

# --- 📂 本地物理路径与环境配置 (可在 settings.json 中覆盖) ---
DEFAULT_BASH_PATH = "/data/data/com.termux/files/usr/bin/bash"
DEFAULT_REFRESH_SH = "/data/data/com.termux/files/home/refresh.sh"
DEFAULT_LOCAL_DROPBOX = "/storage/emulated/0/Download/189cas"
DEFAULT_LOCAL_STRM = "/storage/emulated/0/Download/cas_strm"
DEFAULT_139_LOCAL_STRM = "/storage/emulated/0/Download/139_strm"
DEFAULT_LOCAL_ROOT = "/storage/"  # 用来智能判定是否为本地物理路径的通用前缀

# --- 139专属加工厂配置 ---
API_5255_URL = "http://127.0.0.1:5255"
DIR_139_SOURCE = "/141/141source"
DIR_139_TARGET = "/139/139cas"
DIR_LOCAL_CAS = "/storage/emulated/0/Download/139cas" 

# ==========================================
# 🛡️ 网络底层与 IPv6 拦截
# ==========================================
old_getaddrinfo = socket.getaddrinfo
def new_getaddrinfo(host, port, family=0, type=0, proto=0, flags=0):
    responses = old_getaddrinfo(host, port, family, type, proto, flags)
    # 🌟 智能放行：如果 Flask 试图监听所有 IPv6 通配符，直接放行
    if host == '::':
        return responses
    # 🛡️ 强制锁定：外部请求（天翼云等）全部强杀 IPv6，只保留 IPv4 (AF_INET)
    return [res for res in responses if res[0] == socket.AF_INET]
socket.getaddrinfo = new_getaddrinfo

# ==========================================
# 🛡️ 基础配置与绝对路径定位 (防乱窜装甲)
# ==========================================
BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_DIR = os.path.join(BASE_DIR, "db")
os.makedirs(DB_DIR, exist_ok=True)

logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(message)s')
logger = logging.getLogger(__name__)

# --- 📡 远程对讲机模块 (将日志实时发送给 Web 看板) ---
class RemoteLogHandler(logging.Handler):
    def emit(self, record):
        try:
            msg = self.format(record)
            requests.post(f"{API_5000_URL}/api/remote_log", 
                          json={'level': record.levelname, 'msg': msg}, timeout=0.3)
        except: pass

remote_handler = RemoteLogHandler()
remote_handler.setFormatter(logging.Formatter('%(message)s'))
logger.addHandler(remote_handler)

load_dotenv(dotenv_path=os.path.join(BASE_DIR, "sys.env"), override=True)

ENV_189_CLIENT_ID = os.getenv("ENV_189_CLIENT_ID", "")
ENV_189_CLIENT_SECRET = os.getenv("ENV_189_CLIENT_SECRET", "")
TG_BOT_TOKEN = os.getenv("ENV_TG_BOT_TOKEN", "")
TG_ADMIN_USER_ID = os.getenv("ENV_TG_ADMIN_USER_ID", "")

# ==========================================
# 🌟 精准局部代理通道 (只给境外 API 开小灶)
# ==========================================
LOCAL_PROXIES = {
    "http": "http://127.0.0.1:7890",
    "https": "http://127.0.0.1:7890"
}

# ==========================================
# 🎬 新增：TMDB 终极翻译与扩搜引擎 (带拼音与英文反查)
# ==========================================
TMDB_API_KEY = os.getenv("ENV_TMDB_API_KEY", "")

def get_tmdb_info(keyword):
    """输入中文，自动反查英文名、TMDB ID 和拼音特征"""
    if not TMDB_API_KEY: return None
    
    info = {
        "id": "",
        "cn_name": "",
        "en_name": "",
        "pinyin_full": "",
        "pinyin_initial": ""
    }
    
    try:
        from pypinyin import pinyin, Style
        info["pinyin_full"] = "".join([p[0] for p in pinyin(keyword, style=Style.NORMAL)])
        info["pinyin_initial"] = "".join([p[0][0] for p in pinyin(keyword, style=Style.FIRST_LETTER)])
    except ImportError:
        pass 

    url_cn = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={parse.quote(keyword)}&page=1"
    try:
        res_cn = requests.get(url_cn, timeout=5, proxies=LOCAL_PROXIES).json()
        if res_cn.get("results"):
            top = res_cn["results"][0]
            media_type = top.get("media_type", "tv") 
            tmdb_id = str(top.get("id"))
            info["id"] = tmdb_id
            info["cn_name"] = top.get("name") or top.get("title", "")
            
            url_en = f"https://api.themoviedb.org/3/{media_type}/{tmdb_id}?api_key={TMDB_API_KEY}&language=en-US"
            res_en = requests.get(url_en, timeout=5).json()
            info["en_name"] = res_en.get("name") or res_en.get("title") or top.get("original_name") or ""
            
            return info
    except Exception as e:
        logger.error(f"TMDB检索与反查异常: {e}")
        
    return info if (info["pinyin_full"] or info["pinyin_initial"]) else None

def translate_folder_name(folder_name):
    """提取文件夹名中的TMDB ID，翻译成人话(中文剧名)"""
    if not TMDB_API_KEY: return folder_name
    match = re.search(r'(?i)tmdb[-_]?(\d+)', folder_name)
    if not match: return folder_name
    
    tmdb_id = match.group(1)
    try:
        res_tv = requests.get(f"https://api.themoviedb.org/3/tv/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN", timeout=3, proxies=LOCAL_PROXIES).json()
        if "name" in res_tv:
            cn_name = res_tv["name"]
            s_match = re.search(r'(?i)(S\d+|Season\s*\d+)', folder_name)
            s_tag = f" {s_match.group(1)}" if s_match else ""
            return f"📺 {cn_name}{s_tag} (TMDB-{tmdb_id})"
            
        res_movie = requests.get(f"https://api.themoviedb.org/3/movie/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN", timeout=3, proxies=LOCAL_PROXIES).json()
        if "title" in res_movie:
            return f"🎬 {res_movie['title']} (TMDB-{tmdb_id})"
    except: pass
    return folder_name

def fetch_tmdb_rich_info(keyword):
    """通过 TMDB 获取影视剧的详细多维信息"""
    if not TMDB_API_KEY:
        return "❌ 系统未配置 TMDB_API_KEY，无法查询。"
        
    try:
        url_search = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={parse.quote(keyword)}&page=1"
        res = requests.get(url_search, timeout=5, proxies=LOCAL_PROXIES).json()
        if not res.get("results"):
            return f"📭 TMDB 数据库中未找到关于【{keyword}】的信息。"
            
        top = res["results"][0]
        media_type = top.get("media_type", "tv")
        tmdb_id = top.get("id")
        
        detail_url = f"https://api.themoviedb.org/3/{media_type}/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN"
        detail_res = requests.get(detail_url, timeout=5, proxies=LOCAL_PROXIES).json()
        
        title = detail_res.get("name") or detail_res.get("title", "未知")
        original_title = detail_res.get("original_name") or detail_res.get("original_title", "")
        overview = detail_res.get("overview", "暂无简介")
        if len(overview) > 200: overview = overview[:197] + "..." 
        vote = round(detail_res.get("vote_average", 0), 1)
        genres = ", ".join([g["name"] for g in detail_res.get("genres", [])])
        country = ", ".join(detail_res.get("origin_country", []))
        
        status_trans = {
            "Returning Series": "📺 连载中", "Ended": "✅ 已完结", 
            "Canceled": "❌ 已砍掉", "Released": "✅ 已上映", 
            "Post Production": "🛠 后期制作中", "In Production": "🎥 拍摄中"
        }
        raw_status = detail_res.get("status", "未知")
        status = status_trans.get(raw_status, raw_status)

        if media_type == "tv":
            first_air = detail_res.get("first_air_date", "未知")
            year = first_air[:4] if first_air != '未知' else '未知'
            seasons = detail_res.get("number_of_seasons", 0)
            episodes = detail_res.get("number_of_episodes", 0)
            
            msg = (
                f"📺 <b>{title} ({year})</b>\n"
                f"🏷 <b>原名:</b> {original_title}\n"
                f"🌍 <b>国家:</b> {country}\n"
                f"🎭 <b>类型:</b> {genres}\n"
                f"⭐ <b>评分:</b> {vote} / 10\n"
                f"🎬 <b>状态:</b> {status}\n"
                f"📚 <b>规模:</b> 共 {seasons} 季, {episodes} 集\n"
                f"🔗 <b>TMDB ID:</b> <code>{tmdb_id}</code>\n"
                f"────────────────\n"
                f"📖 <b>简介:</b>\n{overview}"
            )
        else:
            release_date = detail_res.get("release_date", "未知")
            year = release_date[:4] if release_date != '未知' else '未知'
            runtime = detail_res.get("runtime", 0)
            
            msg = (
                f"🎬 <b>{title} ({year})</b>\n"
                f"🏷 <b>原名:</b> {original_title}\n"
                f"🌍 <b>国家:</b> {country}\n"
                f"🎭 <b>类型:</b> {genres}\n"
                f"⭐ <b>评分:</b> {vote} / 10\n"
                f"⏳ <b>时长:</b> {runtime} 分钟\n"
                f"🎬 <b>状态:</b> {status}\n"
                f"🔗 <b>TMDB ID:</b> <code>{tmdb_id}</code>\n"
                f"────────────────\n"
                f"📖 <b>简介:</b>\n{overview}"
            )
        return msg
    except Exception as e:
        logger.error(f"TMDB 详情查询异常: {e}")
        return f"❌ 查询 TMDB 时发生异常，可能是网络超时。"

# ==========================================
# 📁 核心目录、挂载配置与动态路由引擎
# ==========================================
SUBS_FILE = os.path.join(DB_DIR, "subscriptions.json")
HARVEST_SUBS_FILE = os.path.join(DB_DIR, "harvest_subs.json") 
HISTORY_FILE = os.path.join(DB_DIR, "history.json")
COOKIES_FILE = os.path.join(DB_DIR, "cookies.json")
SETTINGS_FILE = os.path.join(DB_DIR, "settings.json") 
ROUTER_FILE = os.path.join(DB_DIR, "router.json") # 🌟 新增独立的字典文件

last_login_time = 0

def load_json(filepath):
    if os.path.exists(filepath):
        with open(filepath, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {}

def save_json(filepath, data):
    with open(filepath, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

# 💥 核心黑科技：全局变量热拔插引擎 (不需要重启脚本，随时覆盖内存变量)
def reload_globals():
    global CAT_ROUTER, DIR_CAS_ROOT, DIR_VIDEO_ROOT, DIR_MEDIA_PREFIX
    global OPENLIST_MOUNT_POINT, DEFAULT_LOCAL_DROPBOX, DEFAULT_LOCAL_STRM
    global DEFAULT_139_LOCAL_STRM, DIR_139_SOURCE, DIR_139_TARGET, DIR_LOCAL_CAS
    
    s = load_json(SETTINGS_FILE)
    # 动态覆盖基础路径 (如果 settings 里没设，就用后面的默认值)
    DIR_CAS_ROOT = s.get("DIR_CAS_ROOT", "/177-秒传")
    DIR_VIDEO_ROOT = s.get("DIR_VIDEO_ROOT", "/177-视频")
    DIR_MEDIA_PREFIX = s.get("DIR_MEDIA_PREFIX", "/177-")
    OPENLIST_MOUNT_POINT = s.get("OPENLIST_MOUNT_POINT", "177")
    DEFAULT_LOCAL_DROPBOX = s.get("DEFAULT_LOCAL_DROPBOX", "/storage/emulated/0/Download/189cas")
    DEFAULT_LOCAL_STRM = s.get("DEFAULT_LOCAL_STRM", "/storage/emulated/0/Download/cas_strm")
    DEFAULT_139_LOCAL_STRM = s.get("DEFAULT_139_LOCAL_STRM", "/storage/emulated/0/Download/139_strm")
    DIR_139_SOURCE = s.get("DIR_139_SOURCE", "/141/141source")
    DIR_139_TARGET = s.get("DIR_139_TARGET", "/139/139cas")
    DIR_LOCAL_CAS = s.get("DIR_LOCAL_CAS", "/storage/emulated/0/Download/139cas")
    
    # 动态加载分类字典
    default_router = {
        "华语剧": ["电视剧", "0-电视剧"], "大陆剧": ["电视剧", "0-电视剧"], "港剧": ["电视剧", "0-电视剧"], "台剧": ["电视剧", "0-电视剧"],
        "华语剧2601": ["电视剧", "0-电视剧"], "华语剧2602": ["电视剧", "0-电视剧"], "华语剧2701": ["电视剧", "0-电视剧"],
        "欧美剧": ["电视剧", "1-电视剧"], "美剧": ["电视剧", "1-电视剧"], "英剧": ["电视剧", "1-电视剧"],
        "欧美剧2601": ["电视剧", "1-电视剧"], "欧美剧2701": ["电视剧", "1-电视剧"],
        "日韩剧": ["电视剧", "2-电视剧"], "韩剧": ["电视剧", "2-电视剧"], "日剧": ["电视剧", "2-电视剧"],
        "日韩剧2601": ["电视剧", "2-电视剧"], "日韩剧2701": ["电视剧", "2-电视剧"],
        "华语电影": ["电影", "0-电影"], "国语电影": ["电影", "0-电影"],
        "华语电影2601": ["电影", "0-电影"], "华语电影2701": ["电影", "0-电影"],
        "欧美电影": ["电影", "1-电影"], "大片": ["电影", "1-电影"],
        "欧美电影2601": ["电影", "1-电影"], "欧美电影2701": ["电影", "1-电影"],
        "日韩电影": ["电影", "2-电影"],
        "国漫": ["动漫", "0-动漫"], "国漫2601": ["动漫", "0-动漫"], "国漫2602": ["动漫", "0-动漫"], "国漫2701": ["动漫", "0-动漫"],
        "日漫": ["动漫", "1-动漫"], "番剧": ["动漫", "1-动漫"],
        "综艺": ["综艺", ""], "纪录片": ["纪录片", ""], "演唱会": ["演唱会", ""], "短剧": ["短剧", ""]
    }
    if not os.path.exists(ROUTER_FILE):
        save_json(ROUTER_FILE, default_router)
        CAT_ROUTER = default_router
    else:
        CAT_ROUTER = load_json(ROUTER_FILE)

# 脚本启动时立即执行一次全量覆盖
reload_globals()

if not os.path.exists(SETTINGS_FILE):
    save_json(SETTINGS_FILE, {"auto_scan_cas": False, "auto_check_subs": True})

def get_openlist_path(cloud189_path):
    clean_path = cloud189_path.strip("/")
    if clean_path.startswith(f"{OPENLIST_MOUNT_POINT}/") or clean_path == OPENLIST_MOUNT_POINT:
        return f"/{clean_path}"
    return f"/{OPENLIST_MOUNT_POINT}/{clean_path}"

def clean_filename(name):
    illegal_chars = '"\\/:*?|<>'
    for char in illegal_chars:
        name = name.replace(char, '')
    return name[:255]

def get_match_key(text): 
    clean = re.sub(r'[（\(\[\{]?\d{4}[）\)\]\}]?', '', text)
    clean = re.sub(r'(?i)\b(4k|1080p|2160p|web-dl|sdr|hdr)\b', '', clean)
    clean = re.sub(r'(完结|连载中|全\d+集|打包|修正)', '', clean)
    clean = re.sub(r'[^\w\u4e00-\u9fa5]', '', clean)
    return clean.lower()

def rsaEncrpt(password, public_key):
    rsakey = RSA.importKey(public_key)
    cipher = Cipher_pksc1_v1_5.new(rsakey)
    return cipher.encrypt(password.encode()).hex()

def generate_smart_name(original_filename, sub_path):
    valid_media_exts = ['.mp4', '.mkv', '.ts', '.avi', '.rmvb', '.flv', '.wmv', '.srt', '.ass', '.iso']
    lower_name = original_filename.lower()
    final_ext = ""
    
    for me in valid_media_exts:
        if lower_name.endswith(f"{me}.cas"):
            final_ext = f"{me}.cas"  
            break
            
    if not final_ext:
        _, ext = os.path.splitext(original_filename)
        if ext.lower() in valid_media_exts or ext.lower() == '.cas':
            final_ext = ext.lower()
            
    if not final_ext:
        _, ext = os.path.splitext(original_filename)
        if ext.lower() in ['.jpg', '.jpeg', '.png', '.nfo', '.txt', '.torrent', '.html']:
            return None
        final_ext = '.cas'
        
    path_parts = sub_path.strip('/').split('/')
    folder_name = path_parts[-1]
    for part in reversed(path_parts):
        if re.match(r'(?i)^Season\s*\d+$|^S\d+$', part.strip()):
            continue
        folder_name = part.strip()
        break
        
    year_in_path = re.search(r'\((\d{4})\)', folder_name)
    year_str = year_in_path.group(1) if year_in_path else ""
    
    clean_show_name = folder_name
    clean_show_name = re.sub(r'\s*\(\d{4}\).*$', '', clean_show_name)
    clean_show_name = re.sub(r'(?i)[_\-\s]*(HQ|IQ|DV|4K|1080[pP]|720[pP]|2160[pP]|WEB-DL|HDR|SDR|HD|H\.?26[45]|x\.?26[45]|BluRay|Remux)[_\-\s]*', '', clean_show_name)
    clean_show_name = re.sub(r'(?i)[\[{\(]?tmdb[-_=]?\w+[\]}\)]?', '', clean_show_name)
    clean_show_name = re.sub(r'[-_\s]+$', '', clean_show_name).strip()
    clean_show_name = clean_show_name.replace(' ', '.') 
    
    tags_match = re.findall(r'(?i)\b(1080p|2160p|4K|DV|HQ|HDR|SDR|IQ|H\.?26[45]|x\.?26[45])\b', original_filename)
    tags = []
    for t in tags_match:
        t_upper = t.upper().replace('.', '')
        if t_upper == '1080P': t_upper = '1080p'
        elif t_upper == '2160P': t_upper = '2160p'
        elif t_upper == 'X264': t_upper = 'H264'
        elif t_upper == 'X265': t_upper = 'H265'
        
        if t_upper not in tags:
            tags.append(t_upper)
            
    tag_str = "." + ".".join(tags) if tags else ""

    if any(k in sub_path for k in ["电影", "movie", "演唱会", "纪录片"]):
        part_match = re.search(r'(?i)(part\d+|cd\d+)', original_filename)
        part_str = f".{part_match.group(1).lower()}" if part_match else ""
        year_part = f".{year_str}" if year_str else ""
        return f"{clean_show_name}{year_part}{part_str}{tag_str}{final_ext}".replace('..', '.')

    ep_patterns = [
        r'(?i)E(?:P)?\s*(\d+)', r'第\s*(\d+)\s*[集话期]',
        r'(?:\[|\()(\d+)(?:\]|\))', r'\s+0*(\d{1,3})\s*(?:\.|$)', r'^0*(\d{1,3})\s*(?:\.|$)'  
    ]
    ep_num = None
    for pattern in ep_patterns:
        match = re.search(pattern, original_filename)
        if match:
            ep_num = int(match.group(1))
            break
            
    if ep_num is None: return original_filename
    season_num = 1
    s_match_file = re.search(r'(?i)S0*(\d+)', original_filename)
    if s_match_file:
        season_num = int(s_match_file.group(1))
    else:
        s_match_path = re.search(r'(?i)Season\s*(\d+)', sub_path)
        if s_match_path:
            season_num = int(s_match_path.group(1))
            
    year_part = f".{year_str}" if year_str else ""
    return f"{clean_show_name}.S{season_num:02d}E{ep_num:02d}{year_part}{tag_str}{final_ext}".replace('..', '.')

# ==========================================
# 🌟 升级版 TelegramNotifier (兼容V4.8日志机制+交互按钮)
# ==========================================
class TelegramNotifier:
    def __init__(self, bot_token, user_id):
        self.bot_token = bot_token
        self.user_id = user_id
        self.base_url = f"https://api.telegram.org/bot{self.bot_token}/" if self.bot_token else None

    def send_message(self, message, reply_markup=None):
        clean_msg = message.replace('\n', '  |  ')
        logger.info(f"📤 [TG推送] {clean_msg}")
        if not self.bot_token: return None
        payload = {"chat_id": self.user_id, "text": message, "parse_mode": "HTML"}
        if reply_markup: payload["reply_markup"] = json.dumps(reply_markup)
        try:
            res = requests.post(f"{self.base_url}sendMessage", json=payload, timeout=10, proxies=LOCAL_PROXIES).json()
            return res.get("result", {}).get("message_id")
        except: return None

    def edit_message(self, message_id, text, reply_markup=None):
        if not self.bot_token: return
        payload = {"chat_id": self.user_id, "message_id": message_id, "text": text, "parse_mode": "HTML"}
        if reply_markup: payload["reply_markup"] = json.dumps(reply_markup)
        try: requests.post(f"{self.base_url}editMessageText", json=payload, timeout=10, proxies=LOCAL_PROXIES)
        except: pass

    def answer_callback(self, callback_query_id, text=""):
        if not self.bot_token: return
        try: requests.post(f"{self.base_url}answerCallbackQuery", json={"callback_query_id": callback_query_id, "text": text}, timeout=5, proxies=LOCAL_PROXIES)
        except: pass

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
        if folder_id is None: folder_id = self.shareDirFileId
        fileList, folders = [], []
        pageNumber = 1
        while True:
            result = self.session.get("https://cloud.189.cn/api/open/share/listShareDir.action", params={
                "pageNum": pageNumber, "pageSize": "10000", "fileId": folder_id,
                "shareDirFileId": self.shareDirFileId, "isFolder": "true",
                "shareId": self.shareId, "shareMode": self.shareMode,
                "orderBy": "lastOpTime", "descending": "true", "accessCode": self.accessCode,
            }).json()
            if result.get('res_code', -1) != 0: break
            fileListAO = result.get("fileListAO", {})
            fileList += fileListAO.get("fileList", [])
            folders += fileListAO.get("folderList", [])
            if fileListAO.get("fileListSize", 0) == 0 and len(fileListAO.get("folderList", [])) == 0: break
            pageNumber += 1
        return {"files": fileList, "folders": folders}

    def saveShareFiles(self, tasksInfos, targetFolderId):
        try:
            response = self.session.post("https://cloud.189.cn/api/open/batch/createBatchTask.action", data={
                "type": "SHARE_SAVE", "taskInfos": json.dumps(tasksInfos, ensure_ascii=False),
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
        except Exception as e: return str(e)

class Cloud189:
    def __init__(self):
        self.session = requests.session()
        self.session.headers = {
            'User-Agent': "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Accept": "application/json;charset=UTF-8",
        }

    def load_cookies(self):
        if os.path.exists(COOKIES_FILE):
            try:
                with open(COOKIES_FILE, 'r') as f:
                    self.session.cookies.update(json.load(f))
                res = self.session.post("https://cloud.189.cn/api/portal/getObjectFolderNodes.action", data={"id": -11, "orderBy": 1, "order": "ASC"}).json()
                if not isinstance(res, dict): return True
            except: pass
        return False

    def save_cookies(self):
        with open(COOKIES_FILE, 'w') as f:
            json.dump(requests.utils.dict_from_cookiejar(self.session.cookies), f)

    def getEncrypt(self):
        return self.session.post("https://open.e.189.cn/api/logbox/config/encryptConf.do", data={'appId': 'cloud'}, timeout=15).json()['data']['pubKey']

    def getRedirectURL(self):
        rsp = self.session.get('https://cloud.189.cn/api/portal/loginUrl.action?redirectURL=https://cloud.189.cn/web/redirect.html?returnURL=/main.action', timeout=15)
        return parse.parse_qs(parse.urlparse(rsp.url).query)

    def login(self, username, password):
        if self.load_cookies():
            logger.info("🍪 [系统] 成功加载本地免密通行证，跳过高危密码登录！")
            return

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
            self.save_cookies()
        else: raise Exception(result['msg'])

    def getShareInfo(self, link):
        url = parse.urlparse(link)
        try: code = parse.parse_qs(url.query)["code"][0]
        except: code = url.path.split('/')[-1]
        pwd = parse.parse_qs(url.query).get('pwd', [''])[0]
        result = self.session.get("https://cloud.189.cn/api/open/share/getShareInfoByCodeV2.action", params={"shareCode": code}).json()
        
        res_code = str(result.get('res_code', ''))
        
        if res_code == 'ShareAuditWaiting' or 'audit waiting' in str(result).lower():
            raise Exception(f"SHARE_AUDIT: 天翼云官方审核风控拦截 [{result.get('res_message', '等待审核')}]")

        if res_code in ['8001', 'ShareNotFound', 'ShareAuditNotPass', 'ShareUserInvalid'] or \
           any(kw in str(result).lower() for kw in ["失效", "取消", "不存在", "审核", "invalid", "not found", "not pass"]):
            raise Exception(f"SHARE_DEAD: 分享已失效或被和谐 [{result.get('res_message', '未知原因')}]")
            
        if result.get('res_code') != 0: raise Exception(f"获取分享失败，可能掉线: {result}")
        file_id = result.get("fileId")
        share_mode = result.get("shareMode", 1)
        share_id = result.get("shareId")
        raw_is_folder = result.get("isFolder")
        is_folder = True if raw_is_folder is None else str(raw_is_folder).lower() in ['true', '1']
        file_name = result.get("fileName", "未命名文件")
        if pwd:
            verify_res = self.session.get("https://cloud.189.cn/api/open/share/checkAccessCode.action", params={"shareCode": code, "accessCode": pwd}).json()
            if verify_res.get('res_code') != 0: raise Exception(f"提取码错误或失效: {verify_res}")
            share_id = verify_res.get("shareId")
        if not share_id: raise Exception("未能获取到 shareId，疑似掉线拦截。")
        return Cloud189ShareInfo(file_id, share_id, share_mode, self, pwd, is_folder, file_name)

    def createFolder(self, name, parentFolderId=-11):
        result = self.session.post("https://cloud.189.cn/api/open/file/createFolder.action", data={"parentFolderId": parentFolderId, "folderName": name}).json()
        return result.get("id", result.get("fileId", "-11"))

    def getObjectFolderNodes(self, folderId=-11):
        res = self.session.post("https://cloud.189.cn/api/portal/getObjectFolderNodes.action", data={"id": folderId, "orderBy": 1, "order": "ASC"}).json()
        if isinstance(res, dict): raise Exception(f"获取目录被网盘拦截或风控: {res}")
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

    def listPrivateFiles(self, folderId):
        all_files = []
        page_num = 1
        while True:
            try:
                res = self.session.get("https://cloud.189.cn/api/open/file/listFiles.action", params={"folderId": folderId, "pageNum": page_num, "pageSize": 100}, timeout=10).json()
                if res.get("res_code") == 0:
                    file_list = res.get("fileListAO", {}).get("fileList", [])
                    if not file_list: break
                    all_files.extend(file_list)
                    page_num += 1
                else: break
            except Exception: break
        return all_files

    def renameFile(self, fileId, destFileName):
        try:
            res = self.session.post("https://cloud.189.cn/api/open/file/renameFile.action", data={"fileId": fileId, "destFileName": destFileName}).json()
            return res.get("res_code") == 0
        except: return False

# ==========================================
# 🤖 核心巡逻、更新检查系统
# ==========================================
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

def auto_relogin(client_obj, force=False):
    global last_login_time
    current_time = time.time()
    
    if not force and (current_time - last_login_time < 1800):
        logger.warning("⏳ [系统] 检测到接口报错，防风控冷却锁生效，跳过登录！")
        return False
        
    logger.info("🔄 [系统] 触发保活机制：正在彻底重洗内存与协议握手...")
    try:
        client_obj.session = requests.session()
        client_obj.session.headers = {
            'User-Agent': "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Accept": "application/json;charset=UTF-8",
        }
        if os.path.exists(COOKIES_FILE):
            os.remove(COOKIES_FILE)

        client_obj.login(ENV_189_CLIENT_ID, ENV_189_CLIENT_SECRET)
        last_login_time = time.time()
        logger.info("✅ [系统] 彻底洗牌重新登录成功！安全冷却锁已重置。")
        return True
    except Exception as e:
        logger.error(f"❌ [系统] 重新登录失败: {e}")
        return False
# ==========================================
# 🌟 139 专属独立全自动流水线 (挂载在 5255 端口 - 终极对齐 189)
# ==========================================
def process_139_pipeline():
    try:
        r_log = requests.post(f"{API_5255_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=10)
        if r_log.json().get("code") != 200: return
        headers_139 = {"Authorization": r_log.json()["data"]["token"], "Content-Type": "application/json"}
    except Exception as e:
        logger.debug(f"⚠️ [139加工] 无法连接 5255 端口: {e}")
        return

    # 1. 递归扫描源目录寻找原始视频
    def scan_source(path):
        files = []
        try:
            res = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": path, "refresh": True}, headers=headers_139, timeout=20).json()
            if res.get("code") == 200:
                for item in (res.get("data") or {}).get("content") or []:
                    item_path = f"{path}/{item['name']}".replace("//", "/")
                    if item["is_dir"]: files.extend(scan_source(item_path))
                    else: files.append({"name": item["name"], "dir": path})
        except: pass
        return files

    source_files = scan_source(DIR_139_SOURCE)
    if not source_files: return

    logger.info(f"🛸 [139加工] 源区发现 {len(source_files)} 个待处理文件，启动流水线...")
    
    strm_dirs_to_refresh = set()
    
    for f in source_files:
        orig_name = f["name"]
        src_dir = f["dir"]
        
        # 过滤非视频文件 (🌟 已经支持 .iso 原盘)
        if not any(orig_name.lower().endswith(ext) for ext in ['.mp4', '.mkv', '.ts', '.iso']): continue
            
        # ==========================================
        # 2. 预计算标准剧名与路径构建
        # ==========================================
        clean_name = orig_name
        rel_path = src_dir.replace(DIR_139_SOURCE, "").strip("/")
        parts = rel_path.split("/")
        
        if len(parts) >= 1:
            raw_show_name = parts[-1]
            season_str = "S01"
            
            for i, p in enumerate(parts):
                if "season" in p.lower():
                    s_num = re.search(r'\d+', p)
                    if s_num: season_str = f"S{s_num.group(0).zfill(2)}"
                    if i > 0: raw_show_name = parts[i-1] 
                    break
                    
            show_name = re.sub(r'\s*[\(\[]\d{4}[\)\]].*', '', raw_show_name).strip()
                    
            if show_name not in orig_name:
                m = re.match(r'^(?:S\d{1,2}E|EP|E|第)?(\d{1,4})(?:集|话)?[\s\._\-~]*(.*)', orig_name, re.IGNORECASE)
                if m:
                    ep_num = m.group(1).zfill(2)
                    remainder = m.group(2).strip(" ~._-")
                    if remainder: clean_name = f"{show_name}.{season_str}E{ep_num}.{remainder}"
                    else: clean_name = f"{show_name}.{season_str}E{ep_num}{orig_name[orig_name.rfind('.'):]}"
                    clean_name = clean_name.replace(" ", ".").replace("..", ".").replace(".~", ".").replace("~.", ".")
                else:
                    try: clean_name = generate_smart_name(orig_name, src_dir) or orig_name
                    except: pass
        
        target_dir = f"{DIR_139_TARGET}/{rel_path}".strip().replace("//", "/")
        is_movie = any(k in rel_path.lower() for k in ["电影", "movie"])
        has_season = any("season" in p.lower() or "季" in p for p in parts)
        if not is_movie and not has_season:
            target_dir = f"{target_dir}/Season 1".replace("//", "/")
            
        requests.post(f"{API_5255_URL}/api/fs/mkdir", json={"path": target_dir}, headers=headers_139).close()
        
        # ==========================================
        # 3. 双轨雷达：兼容秒传与跨盘测速防卡死
        # ==========================================
        max_retries = 3
        task_success = False
        actual_cas_name = ""
        
        for attempt in range(max_retries):
            already_queued = False
            target_task_id = None
            try:
                check_resp = requests.get(f"{API_5255_URL}/api/task/copy/undone", headers={"Authorization": headers_139["Authorization"]}, timeout=5)
                if check_resp.status_code == 200 and check_resp.json().get("code") == 200:
                    for t in check_resp.json().get("data", []):
                        t_name = str(t.get("name", ""))
                        if orig_name in t_name:
                            already_queued = True
                            target_task_id = t.get("id")
                            break
            except: pass

            if already_queued:
                logger.info(f"🔄 [139加工] 发现 `{orig_name}` 已在队列，接管监控。")
            else:
                logger.info(f"🚚 [139加工] 第 {attempt+1} 次发起转存: {orig_name}")
                requests.post(f"{API_5255_URL}/api/fs/copy", json={"src_dir": src_dir, "dst_dir": target_dir, "names": [orig_name]}, headers=headers_139).close()
                time.sleep(2.0)
            
            stuck_count = 0
            task_failed = False
            task_ever_seen = False
            
            last_check_time = time.time()
            last_loaded_bytes = 0
            is_first_calc = True
            task_running_start_time = 0 
            wait_start_time = time.time()
            
            while True:
                time.sleep(5.0)
                task_in_undone = False
                current_state_val = -1
                
                try:
                    undone_resp = requests.get(f"{API_5255_URL}/api/task/copy/undone", headers={"Authorization": headers_139["Authorization"]}, timeout=10)
                    if undone_resp.status_code == 200 and undone_resp.json().get("code") == 200:
                        tasks = undone_resp.json().get("data", [])
                        if isinstance(tasks, list):
                            for t in tasks:
                                t_name = str(t.get("name", ""))
                                if orig_name in t_name:
                                    task_in_undone = True
                                    target_task_id = t.get("id")
                                    current_state_val = t.get("state", -1) 
                                    current_progress = float(t.get("progress", 0))
                                    total_bytes = int(t.get("total_bytes", 0))
                                    current_loaded = int(total_bytes * (current_progress / 100.0))
                                    break
                except Exception as e:
                    pass
                
                if task_in_undone:
                    task_ever_seen = True
                    wait_start_time = time.time()
                    now = time.time()
                    
                    if current_state_val != 1:
                        stuck_count = 0
                        last_check_time = now
                        continue
                        
                    if task_running_start_time == 0:
                        task_running_start_time = now
                        
                    if now - task_running_start_time < 15.0:
                        last_check_time = now
                        last_loaded_bytes = current_loaded
                        continue
                    
                    duration = now - last_check_time
                    current_speed_bytes = 0
                    
                    if is_first_calc:
                        is_first_calc = False
                    else:
                        if duration > 0:
                            current_speed_bytes = (current_loaded - last_loaded_bytes) / duration
                        if current_speed_bytes < 0: current_speed_bytes = 0
                        
                        if current_speed_bytes < 1048576:
                            stuck_count += 1
                        else:
                            stuck_count = 0
                            
                        if stuck_count >= 6:
                            if target_task_id:
                                try:
                                    requests.post(f"{API_5255_URL}/api/task/copy/cancel?tid={target_task_id}", headers=headers_139, timeout=5)
                                    time.sleep(1.0)
                                    requests.post(f"{API_5255_URL}/api/task/copy/delete?tid={target_task_id}", headers=headers_139, timeout=5)
                                    time.sleep(3.0) 
                                except: pass
                            task_failed = True
                            break
                            
                    last_loaded_bytes = current_loaded
                    last_check_time = now
                    continue
                
                try:
                    fs_resp = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": target_dir, "refresh": True}, headers=headers_139, timeout=10)
                    fs_data = fs_resp.json()
                    if fs_data.get("code") == 200:
                        contents = [item["name"] for item in (fs_data.get("data", {}).get("content", []) or [])]
                        
                        if orig_name in contents:
                            actual_cas_name = orig_name
                            task_success = True
                            break
                        elif f"{orig_name}.cas" in contents:
                            actual_cas_name = f"{orig_name}.cas"
                            task_success = True
                            break
                        elif f"{orig_name.rsplit('.', 1)[0]}.cas" in contents:
                            actual_cas_name = f"{orig_name.rsplit('.', 1)[0]}.cas"
                            task_success = True
                            break
                except Exception as e:
                    pass
                
                if task_ever_seen:
                    task_failed = True
                    break
                
                if time.time() - wait_start_time > 15.0:
                    task_failed = True
                    break
                
            if task_success: break
            elif task_failed: continue
                
        if not task_success:
            logger.error(f"❌ [139加工] 连续重试均失败，放弃: {orig_name}")
            continue

        # ==========================================
        # 4. 真实下载、销毁源文件与 189对齐版的 STRM 造物引擎
        # ==========================================
        target_cas_name = f"{clean_name}.cas"
        if actual_cas_name and actual_cas_name != target_cas_name:
            requests.post(f"{API_5255_URL}/api/fs/rename", json={"name": target_cas_name, "path": f"{target_dir}/{actual_cas_name}"}, headers=headers_139).close()
            time.sleep(1.0)
            
        try:
            local_target_dir = target_dir.replace(DIR_139_TARGET, DIR_LOCAL_CAS)
            os.makedirs(local_target_dir, exist_ok=True)
            local_cas_path = os.path.join(local_target_dir, target_cas_name)
            cloud_cas_path = f"{target_dir}/{target_cas_name}"
            
            r_get = requests.post(f"{API_5255_URL}/api/fs/get", json={"path": cloud_cas_path}, headers=headers_139, timeout=10)
            if r_get.status_code == 200 and r_get.json().get("code") == 200:
                raw_url = r_get.json().get("data", {}).get("raw_url")
                if raw_url:
                    r_download = requests.get(raw_url, stream=True, timeout=60)
                    if r_download.status_code == 200:
                        with open(local_cas_path, 'wb') as local_f:
                            for chunk in r_download.iter_content(chunk_size=8192):
                                if chunk: local_f.write(chunk)
                        logger.info(f"✨ [139加工] 本地真实 CAS 下载成功: {local_cas_path}")
                        
                        # ==========================================
                        # 🎯 终极整合：139 专属 API 造物引擎 (189完全对齐架构)
                        # ==========================================
                        s = load_json(SETTINGS_FILE)
                        strm_mode_139 = s.get("139_strm_mode", "mode_a_139")
                        local_139_strm_dir = s.get("local_139_strm_dir", DEFAULT_139_LOCAL_STRM)
                        
                        # --------- 提取原始带标签信息 ---------
                        rel_path_strm = target_dir.replace(DIR_139_TARGET, "").strip("/")
                        parts_strm = rel_path_strm.split("/")
                        category_key = parts_strm[0] if len(parts_strm) > 0 else None
                        show_folder_name = parts_strm[-1] # 完全保留带年份/4K等所有标签的纯正剧名
                        
                        local_season_num = 1
                        if len(parts_strm) > 1:
                            for part in reversed(parts_strm[:-1]):
                                s_match_dir = re.match(r'(?i)^(?:season\s*|s)(\d+)$', part.strip())
                                if s_match_dir:
                                    local_season_num = int(s_match_dir.group(1))
                                    continue
                                show_folder_name = part.strip()
                                break
                                
                        # --------- 跨月断档防拆分扫描 ---------
                        current_ym = datetime.now().strftime("%Y%m")
                        if os.path.exists(local_139_strm_dir):
                            for root, dirs, files in os.walk(local_139_strm_dir):
                                if show_folder_name in dirs:
                                    ym_match = re.search(r'/(\d{6})$', root.replace('\\', '/'))
                                    if ym_match:
                                        current_ym = ym_match.group(1)
                                        break
                                        
                        # --------- CAT_ROUTER 基建分层翻译 ---------
                        b_large, b_sub = "未分类", "0-未分类"
                        if category_key and category_key in CAT_ROUTER:
                            b_large, b_sub = CAT_ROUTER[category_key]
                        else:
                            for cat_k, (l, sub) in CAT_ROUTER.items():
                                if cat_k in rel_path_strm:
                                    b_large, b_sub = l, sub
                                    break
                            if b_large == "未分类": b_large, b_sub = "电视剧", "0-电视剧"
                            
                        # --------- STRM 树状目录拼装 ---------
                        virtual_cloud_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{current_ym}/{show_folder_name}".replace("//", "/")
                        
                        if b_large in ["电视剧", "动漫", "短剧"]:
                            virtual_cloud_path = f"{virtual_cloud_path}/Season {local_season_num}"
                        elif local_season_num > 1 and b_large not in ["电影", "演唱会"]:
                            virtual_cloud_path = f"{virtual_cloud_path}/Season {local_season_num}"
                            
                        strm_sub_dir = virtual_cloud_path.replace(DIR_CAS_ROOT, "").strip("/")
                        t_strm_dir = os.path.join(local_139_strm_dir, strm_sub_dir).replace("\\", "/")
                        
                        base_clean_name = target_cas_name[:-4] if target_cas_name.lower().endswith('.cas') else target_cas_name
                        
                        if strm_mode_139 == "both_139":
                            strm_n = base_clean_name + ".strm"
                        elif strm_mode_139 == "mode_b_139":
                            strm_n = base_clean_name + "-139B.strm"
                        else:
                            strm_n = base_clean_name + "-139A.strm"

                        # 坚决不干扰洗名，原汁原味透传
                        payload = {
                            "source_cas_path": local_cas_path,
                            "target_local_dir": t_strm_dir,
                            "cloud_cas_path": cloud_cas_path,
                            "strm_name": strm_n,
                            "show_name": show_folder_name,
                            "mode": strm_mode_139
                        }
                            
                        try:
                            res = requests.post(f"{API_5000_URL}/api/make_strm", json=payload, timeout=5)
                            if res.status_code == 200:
                                logger.info(f"✨ [139加工] 管家 API 造物成功: {strm_n} (模式: {strm_mode_139})")
                                strm_dirs_to_refresh.add(t_strm_dir)

                                # --- 🔤 无损迁移：外挂字幕同传体系 ---
                                try:
                                    src_dir_sub = os.path.dirname(local_cas_path)
                                    valid_exts = ('.srt', '.ass', '.ssa', '.vtt', '.idx', '.sub')
                                    core_n = target_cas_name[:-4] if target_cas_name.lower().endswith('.cas') else target_cas_name
                                    for ext in ['.mp4', '.mkv', '.ts', '.iso', '.rmvb', '.avi']:
                                        if core_n.lower().endswith(ext):
                                            core_n = core_n[:-len(ext)]
                                            break
                                            
                                    for sub_f in os.listdir(src_dir_sub):
                                        if sub_f.lower().endswith(valid_exts) and sub_f.startswith(core_n):
                                            sub_src = os.path.join(src_dir_sub, sub_f)
                                            sub_dst = os.path.join(t_strm_dir, sub_f)
                                            os.makedirs(t_strm_dir, exist_ok=True)
                                            if not os.path.exists(sub_dst) or os.stat(sub_src).st_mtime > os.stat(sub_dst).st_mtime:
                                                shutil.copy2(sub_src, sub_dst)
                                                logger.info(f"🔤 [139字幕] 无缝同步外挂字幕: {sub_f}")
                                except Exception as sub_e:
                                    logger.error(f"❌ [139字幕] 抓取失败: {sub_e}")
                            else:
                                logger.error(f"❌ [139加工] 管家 API 拒绝: {res.text}")
                        except Exception as e:
                            logger.error(f"❌ [139加工] 呼叫管家 API 失败: {e}")

                    else:
                        logger.error(f"❌ [139加工] 下载 CAS 直链失败，HTTP状态码: {r_download.status_code}")
                else:
                    logger.error(f"❌ [139加工] 未能获取到云端 CAS 文件的 raw_url")
            else:
                logger.error(f"❌ [139加工] 获取云端文件信息接口报错")
        except Exception as e:
            logger.error(f"❌ [139加工] 本地 CAS 镜像下载异常: {e}")
        
        logger.info(f"💥 [139加工] 流水线闭环，销毁源视频: {orig_name}")
        requests.post(f"{API_5255_URL}/api/fs/remove", json={"dir": src_dir, "names": [orig_name]}, headers=headers_139).close()

    try: requests.post(f"{API_5255_URL}/api/task/copy/clear_done", headers={"Authorization": headers_139["Authorization"]}, timeout=5).close()
    except: pass
    
    if strm_dirs_to_refresh:
        try:
            s = load_json(SETTINGS_FILE)
            bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
            refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
            
            def delayed_139_refresh(dirs):
                time.sleep(5.0)
                for d in dirs:
                    subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                    logger.info(f"⚡ [139聚合刷新] 已精准下发 Emby 刷新指令: {d}")
                    time.sleep(1.0) 
                    
            threading.Thread(target=delayed_139_refresh, args=(strm_dirs_to_refresh,)).start()
        except Exception as e:
            logger.error(f"❌ [139加工] 唤醒 Emby 刷新异常: {e}")

# ==========================================
# 🌟 独立新增的 CAS 收割模块 (终极暴力认亲 + 缓存阵列极速批次版)
# ==========================================
def process_cas_via_olist_api(specific_dir=None):
    s = load_json(SETTINGS_FILE)
    WATCH_DIRS = s.get("watch_dirs", ["/family/177_cas", "/local_cas"])
    
    if specific_dir:
        WATCH_DIRS = [specific_dir]
    NO_DELETE_DIRS = s.get("no_delete_dirs", [])
    
    WATCH_DIRS = list(set(WATCH_DIRS + NO_DELETE_DIRS))
    processed_names = []
    if not WATCH_DIRS: return processed_names 

    # --- SQLite3 毫秒级本地高速引擎 ---
    db_path = os.path.join(DB_DIR, "harvest_ledger.db")
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    cursor.execute('CREATE TABLE IF NOT EXISTS ledger (filename TEXT PRIMARY KEY)')
    
    history_file = os.path.join(DB_DIR, "harvest_ledger.json")
    if os.path.exists(history_file):
        try:
            old_data = load_json(history_file)
            if old_data:
                for k in old_data.keys():
                    cursor.execute('INSERT OR IGNORE INTO ledger (filename) VALUES (?)', (k,))
                conn.commit()
            os.remove(history_file)
            logger.info("♻️ [系统] 旧版臃肿 JSON 账本已成功升级为 SQLite 高速数据库！")
        except: pass

    def check_in_ledger(name):
        cursor.execute('SELECT 1 FROM ledger WHERE filename = ?', (name,))
        return cursor.fetchone() is not None

    def add_to_ledger(name):
        cursor.execute('INSERT OR IGNORE INTO ledger (filename) VALUES (?)', (name,))
        conn.commit()

    strm_ledger_file = os.path.join(DB_DIR, "strm_ledger.json")
    strm_ledger = load_json(strm_ledger_file)
    strm_ledger_changed = False

    try:
        r_log = requests.post(f"{API_5244_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=10)
        login_res = r_log.json()
        r_log.close()
        if login_res.get("code") != 200:
            logger.error(f"❌ [登录] OList 登录失败: {login_res.get('message')}")
            return processed_names
        token = login_res["data"]["token"]
        headers = {"Authorization": token, "Content-Type": "application/json"}
    except Exception as e:
        logger.error(f"❌ [登录] OList 接口异常: {e}")
        return processed_names

    olist_dir_cache = {}
    def get_olist_dir_cached(path_val):
        if path_val in olist_dir_cache:
            return olist_dir_cache[path_val]
        try:
            r = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": path_val, "password": "", "page": 1, "per_page": 2000, "refresh": True}, headers=headers, timeout=15)
            if r.json().get("code") == 200:
                content = (r.json().get("data") or {}).get("content", [])
                olist_dir_cache[path_val] = content
                r.close()
                return content
            r.close()
        except: pass
        olist_dir_cache[path_val] = []
        return []

    def scan_olist_dir(path):
        files = []
        try:
            r = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": path, "password": "", "page": 1, "per_page": 3000, "refresh": True}, headers=headers, timeout=45)
            res = r.json()
            r.close()
            if res.get("code") != 200:
                logger.warning(f"⚠️ [扫描] 目录访问失败: {path} - {res.get('message')}")
                return files
            
            content = (res.get("data") or {}).get("content") or []
            for item in content:
                item_path = f"{path}/{item['name']}".replace("//", "/")
                if item["is_dir"]: 
                    files.extend(scan_olist_dir(item_path))
                else:
                    if item["name"].lower().endswith(".cas"):
                        if check_in_ledger(item["name"]):
                            pass 
                        else:
                            files.append({"name": item["name"], "dir": path, "full_path": item_path})
        except Exception as e:
            logger.warning(f"⚠️ [警告] 扫描目录 {path} 时 OpenList 接口超时无响应: {e}")
        return files

    cas_files = []
    for watch_dir in WATCH_DIRS:
        logger.info(f"🌾 [收割] 正在巡逻云端目录: {watch_dir}")
        cas_files.extend(scan_olist_dir(watch_dir))

    if not cas_files: 
        logger.info("ℹ️ [收割] 暂无需要处理的 CAS 文件")
        conn.close()
        return processed_names

    subs = load_json(SUBS_FILE)
    current_ym = datetime.now().strftime("%Y%m")
    updated_paths = set()
    created_dirs = set()
    
    batch_groups = {} 
    for cas in cas_files:
        filename = cas["name"]
        raw_dir = cas["dir"]
        active_watch_dir = next((wd for wd in WATCH_DIRS if raw_dir.startswith(wd)), WATCH_DIRS[0])
        rel_dir = raw_dir.replace(active_watch_dir, "").strip("/")
        parts = rel_dir.split("/") if rel_dir else []
        
        category_key = None
        show_folder_name = parts[0] if parts else "未分类手动入库"
        if len(parts) > 1 and parts[0] in CAT_ROUTER:
            category_key, show_folder_name = parts[0], parts[1]
        elif "#" in show_folder_name:
            for cat in CAT_ROUTER.keys():
                if f"#{cat}" in show_folder_name:
                    category_key = cat
                    show_folder_name = show_folder_name.replace(f"#{cat}", "").strip()
                    break

        raw_season_match = re.search(r'(?i)Season\s*(\d+)|S(\d+)(?!\d)', raw_dir.split('/')[-1])
        if raw_season_match: true_season_num = int(raw_season_match.group(1) or raw_season_match.group(2))
        else:
            file_s_match = re.search(r'(?i)S0*(\d+)', filename)
            true_season_num = int(file_s_match.group(1)) if file_s_match else 1
            
        temp_ep_num = int(re.search(r'(?i)E(?:P)?\s*(\d+)', filename).group(1)) if re.search(r'(?i)E(?:P)?\s*(\d+)', filename) else 1
        
        search_key = get_match_key(show_folder_name)
        target_cloud_path = None
        is_tv_show = False
        
        ignore_words = {get_match_key(DIR_CAS_ROOT), get_match_key(DIR_VIDEO_ROOT), "season", "s"}
        for cat_key, (large_cat, sub_cat) in CAT_ROUTER.items():
            ignore_words.add(get_match_key(cat_key))
            ignore_words.add(get_match_key(large_cat))
            if sub_cat: ignore_words.add(get_match_key(sub_cat))

        best_match_path = None
        try:
            h_data = load_json(HARVEST_SUBS_FILE)
            if search_key in h_data: best_match_path = h_data[search_key]
        except Exception: pass

        if not best_match_path:
            db_possible_matches = []
            for sid, info_dict in subs.items():
                if isinstance(info_dict, dict):
                    db_path = info_dict.get("path", "")
                    if DIR_CAS_ROOT not in db_path: continue 
                    db_folders = db_path.split('/')
                    for idx, f_name in enumerate(db_folders):
                        pure_f = get_match_key(f_name)
                        if not pure_f or len(pure_f) < 2 or re.match(r'^\d{4,6}$', pure_f) or pure_f in ignore_words: continue
                        if search_key == pure_f: 
                            db_possible_matches.append("/".join(db_folders[:idx+1]))
                            break
            if db_possible_matches:
                db_possible_matches.sort(key=lambda p: (0 if p.split('/')[-1].lower() == show_folder_name.lower() else 1, len(p.split('/')[-1])))
                best_match_path = db_possible_matches[0]

        if not best_match_path:
            try:
                search_roots = []
                if category_key:
                    b_large, b_sub = CAT_ROUTER[category_key]
                    search_roots.append(get_openlist_path(f"{DIR_CAS_ROOT}/{b_large}/{b_sub}".strip("/").replace("//", "/")))
                else:
                    search_roots = [get_openlist_path(f"{DIR_CAS_ROOT}/动漫/0-动漫"), get_openlist_path(f"{DIR_CAS_ROOT}/电视剧/0-电视剧")]
                
                for root_path in search_roots:
                    ym_items = get_olist_dir_cached(root_path)
                    ym_nodes = [item["name"] for item in ym_items if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                    ym_nodes.sort(reverse=True)
                    
                    all_months_matches = []
                    for ym in ym_nodes:
                        ym_path = f"{root_path}/{ym}"
                        show_items = get_olist_dir_cached(ym_path)
                        for item in show_items:
                            if item["is_dir"] and search_key == get_match_key(item["name"]):
                                all_months_matches.append({"ym": ym, "name": item["name"]})
                                
                    if all_months_matches:
                        all_months_matches.sort(key=lambda x: (0 if x["name"].lower() == show_folder_name.lower() else 1, len(x["name"]), -int(x["ym"])))
                        best_share = all_months_matches[0]
                        best_match_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{best_share['ym']}/{best_share['name']}".replace("//", "/")
                    if best_match_path: break
            except: pass

        if best_match_path:
            target_cloud_path = best_match_path
            if any(k in target_cloud_path for k in ["电视剧", "动漫", "短剧"]): is_tv_show = True
            
        if not target_cloud_path:
            if category_key:
                base_large, base_sub = CAT_ROUTER[category_key]
                target_cloud_path = f"{DIR_CAS_ROOT}/{base_large}/{base_sub}/{current_ym}/{show_folder_name}".replace("//", "/")
                if base_large in ["电视剧", "动漫", "短剧"]: is_tv_show = True
            else:
                if true_season_num > 1 or temp_ep_num > 1: target_cloud_path = f"{DIR_CAS_ROOT}/电视剧/0-电视剧/{current_ym}/{show_folder_name}"; is_tv_show = True
                else: target_cloud_path = f"{DIR_CAS_ROOT}/电影/0-电影/{current_ym}/{show_folder_name}"

        base_notify_path = target_cloud_path 
        if is_tv_show:
            if not re.search(r'(?i)/Season\s*\d+$', target_cloud_path): target_cloud_path = f"{target_cloud_path}/Season {true_season_num}"
        elif true_season_num > 1 or (len(parts) > 1 and "season" in parts[-1].lower()):
            if not re.search(r'(?i)/Season\s*\d+$', target_cloud_path): target_cloud_path = f"{target_cloud_path}/Season {true_season_num}"

        is_no_delete = any(nd in raw_dir for nd in NO_DELETE_DIRS)
        if is_no_delete:
            final_name = filename 
        else:
            final_name = generate_smart_name(filename, target_cloud_path) or filename
            
        final_target_dir = get_openlist_path(target_cloud_path)
        
        group_key = (raw_dir, final_target_dir, base_notify_path)
        if group_key not in batch_groups: batch_groups[group_key] = []
        batch_groups[group_key].append({"orig": filename, "final": final_name})

    for (raw_dir, final_target_dir, base_notify_path), file_items in batch_groups.items():
        if final_target_dir not in created_dirs:
            requests.post(f"{API_5244_URL}/api/fs/mkdir", json={"path": final_target_dir}, headers=headers).close()
            created_dirs.add(final_target_dir)
            logger.info(f"📁 [基建] 新建目录: {final_target_dir} ...")
            time.sleep(1.5)

        is_no_delete = any(nd in raw_dir for nd in NO_DELETE_DIRS)
        api_endpoint = "copy" if is_no_delete else "move"
        action_name = "复制" if is_no_delete else "移动"
        
        names_to_move = []
        for item in file_items:
            orig_n, final_n = item["orig"], item["final"]
            if orig_n != final_n:
                src_path = f"{raw_dir}/{orig_n}".replace("//", "/")
                requests.post(f"{API_5244_URL}/api/fs/rename", json={"name": final_n, "path": src_path}, headers=headers).close()
                names_to_move.append(final_n)
            else: names_to_move.append(orig_n)
                
        time.sleep(5.0)
        requests.post(f"{API_5244_URL}/api/fs/list", json={"path": raw_dir, "refresh": True}, headers=headers).close()

        logger.info(f"🚚 [{action_name}] 正在处理: {final_target_dir} ({len(names_to_move)}件)")
        r_mov = requests.post(f"{API_5244_URL}/api/fs/{api_endpoint}", json={"src_dir": raw_dir, "dst_dir": final_target_dir, "names": names_to_move}, headers=headers)
        mov_res = r_mov.json()
        r_mov.close()
        
        if mov_res.get("code") == 200:
            logger.info(f"✅ [{action_name}] 批量成功!")
            processed_names.extend(names_to_move)

            if is_no_delete:
                for name in names_to_move: add_to_ledger(name)
            else:
                need_sync = False
                for name in names_to_move:
                    if name in strm_ledger:
                        del strm_ledger[name]
                        strm_ledger_changed = True
                    else:
                        need_sync = True
                if need_sync:
                    updated_paths.add(base_notify_path)
                
        else:
            if "exists" not in str(mov_res.get('message', '')).lower():
                logger.error(f"❌ [{action_name}] 批量失败: {mov_res.get('message')}")
                
            for name in names_to_move:
                single_res = requests.post(f"{API_5244_URL}/api/fs/{api_endpoint}", json={"src_dir": raw_dir, "dst_dir": final_target_dir, "names": [name]}, headers=headers).json()
                
                is_exists = single_res.get("code") != 200 and "exists" in str(single_res.get("message", "")).lower()
                
                if single_res.get("code") == 200 or is_exists:
                    if is_exists:
                        logger.debug(f"⏭️ [{action_name}] 目标已存在，已静默修复 db 数据库记忆: {name}")
                    else:
                        logger.info(f"✅ [{action_name}] 单件成功: {name}")
                        
                    processed_names.append(name)
                    
                    if is_no_delete:
                        add_to_ledger(name)
                    else:
                        if name in strm_ledger:
                            del strm_ledger[name]
                            strm_ledger_changed = True
                        else:
                            if not is_exists: updated_paths.add(base_notify_path)

    if strm_ledger_changed:
        save_json(strm_ledger_file, strm_ledger)
        
    conn.close() 

    if updated_paths:
        logger.info("⏳ [引擎] 物理归档结束，等待管家同步...")
        time.sleep(8)
        target_media_roots = set()
        for p in updated_paths:
            parts = p.split('/')
            media_root = ""
            for i, part in enumerate(parts):
                if part.lower().startswith("season"):
                    media_root = '/'.join(parts[:i])
                    break
            if not media_root:
                last_part = parts[-1]
                if '.' in last_part and any(last_part.lower().endswith(ext) for ext in ['.mp4', '.mkv', '.ts', '.iso', '.rmvb', '.avi', '.cas', '.strm']):
                    media_root = os.path.dirname(p)
                else: media_root = p
            if media_root: target_media_roots.add(media_root)
        
        strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
        sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
        real_type = sync_type_map.get(strm_mode_189, "cas")
        strm_dirs_to_refresh = set()
        
        for media_root in target_media_roots:
            olist_p = get_openlist_path(media_root)
            try:
                requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": olist_p, "type": real_type}, timeout=3).close()
                logger.info(f"🔄 [管家] 触发云端目录同步: {media_root} (模式: {strm_mode_189})")
                
                local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
                strm_sub_dir = media_root.split(DIR_CAS_ROOT)[-1].strip("/")
                target_local_strm_path = os.path.join(local_strm_dir, strm_sub_dir).replace("\\", "/")
                strm_dirs_to_refresh.add(target_local_strm_path)
            except Exception as e:
                pass
                
        if strm_dirs_to_refresh:
            bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
            refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
            
            def delayed_harvest_refresh(dirs):
                time.sleep(10.0)
                for d in dirs:
                    subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                    logger.info(f"⚡ [收割聚合刷新] 已精确唤醒 Emby 局部更新: {d}")
                    time.sleep(1.0)
                    
            threading.Thread(target=delayed_harvest_refresh, args=(strm_dirs_to_refresh,)).start()
                
    return processed_names

# ==========================================
# 🌟 查重逻辑与自动订阅拉取
# ==========================================
def check_subscriptions(client_obj, force_target_id=None, is_first_run=False, ignore_time=False):
    subs = load_json(SUBS_FILE)
    history = load_json(HISTORY_FILE)
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    if not subs: return
    
    global_emby_paths = set()
    global_cas_paths = set() 
    
    if not force_target_id:
        logger.info(f"🛸 [雷达] 全频段扫描启动，当前监控池共有 {len(subs)} 个挂载节点")
    
    for target_id, sub_info in list(subs.items()): 
        try:
            if force_target_id and str(target_id) != str(force_target_id): continue
                
            share_url = sub_info if isinstance(sub_info, str) else sub_info.get("url", "")
            keyword = "" if isinstance(sub_info, str) else sub_info.get("keyword", "")
            path = "" if isinstance(sub_info, str) else sub_info.get("path", "")
            freq = "" if isinstance(sub_info, str) else sub_info.get("freq", "")

            if not force_target_id and not is_first_run and not ignore_time:
                if path:
                    now = datetime.now()
                    curr_h, curr_m, curr_w = now.hour, now.minute, now.weekday()

                    if freq == "剧迷":
                        if not ((10 <= curr_h < 12) or (18 <= curr_h < 24)): continue
                    elif freq == "周更" or "周更" in path or "动漫" in path:
                        target_weekday = sub_info.get("update_weekday", 5) if isinstance(sub_info, dict) else 5
                        valid_days = [target_weekday, (target_weekday+1)%7, (target_weekday+2)%7]
                        
                        is_am = (curr_h == 10 and curr_m >= 30) or (curr_h == 11)
                        is_pm = (curr_h >= 18 and curr_m >= 30) or (curr_h >= 19)
                        
                        if not (is_am or is_pm): 
                            continue 
                            
                        curr_week = now.strftime("%Y-%V")
                        curr_day = now.strftime("%Y-%m-%d")
                        last_week = sub_info.get("last_success_week", "")
                        last_day = sub_info.get("last_success_day", "")
                        
                        if curr_w in valid_days:
                            if last_week == curr_week and last_day != curr_day:
                                continue 
                            else:
                                pass
                        else:
                            continue
                    elif freq == "日更" or "日更" in path or "电视剧" in path or "剧" in path:
                        if curr_h < 18: continue
            
            logger.info(f"📡 [侦测] 核对上游动态节点: {path} ...")
            
            info = client_obj.getShareInfo(share_url)
            all_files = get_all_share_files_recursive(info)
            
            cloud_files = client_obj.listPrivateFiles(target_id)
            existing_names = {cf["name"] for cf in cloud_files}

            new_files = []
            for f in all_files:
                if str(f["id"]) in history: continue
                if keyword and not all(k in f["full_path"].lower() for k in keyword.lower().split()): continue
                
                smart_target_name = generate_smart_name(f["name"], path)
                if smart_target_name is None: continue
                
                is_duplicate = False
                if smart_target_name in existing_names or f["name"] in existing_names:
                    is_duplicate = True
                else:
                    core_match = re.search(r'\.S\d+E\d+', smart_target_name)
                    if core_match:
                        core_str = core_match.group(0)
                        for ex_name in existing_names:
                            if core_str in ex_name:
                                is_duplicate = True
                                break

                if is_duplicate:
                    history[str(f["id"])] = {"name": f["name"], "sub_id": str(target_id)}
                    continue
                
                new_files.append(f)

            if new_files:
                logger.info(f"🎯 [搬运] 锁定 {len(new_files)} 个增量更新文件，开始物理下发...")
                
                taskInfos = [{"fileId": f["id"], "fileName": clean_filename(f["name"]), "isFolder": 0} for f in new_files]
                code = info.saveShareFiles(taskInfos, target_id)
                
                time.sleep(8)
                fresh_cloud_files = client_obj.listPrivateFiles(target_id)
                fresh_names = [cf["name"] for cf in fresh_cloud_files]
                
                actually_saved_count = 0
                saved_tasks = []
                failed_tasks = []

                for task in taskInfos:
                    orig_name = task["fileName"]
                    expected_smart_name = generate_smart_name(orig_name, path)
                    
                    if orig_name in fresh_names or (expected_smart_name and expected_smart_name in fresh_names):
                        history[str(task["fileId"])] = {"name": orig_name, "sub_id": str(target_id)}
                        actually_saved_count += 1
                        saved_tasks.append(task)
                    else:
                        failed_tasks.append(task)

                if actually_saved_count > 0:
                    save_json(HISTORY_FILE, history)
                    if isinstance(subs.get(str(target_id)), dict):
                        now_dt = datetime.now()
                        subs[str(target_id)]["last_update"] = int(time.time())
                        if freq == "周更" or "周更" in path or "动漫" in path:
                            subs[str(target_id)]["last_success_week"] = now_dt.strftime("%Y-%V")
                            subs[str(target_id)]["last_success_day"] = now_dt.strftime("%Y-%m-%d")
                        save_json(SUBS_FILE, subs)
                        
                    renamed_files_list = []
                    for cf in fresh_cloud_files:
                        hist_info = history.get(str(cf["id"]))
                        orig_name = hist_info["name"] if hist_info else cf["name"]
                        new_name = generate_smart_name(orig_name, path)
                        
                        if new_name and cf["name"] != new_name:
                            if client_obj.renameFile(cf["id"], new_name):
                                renamed_files_list.append(new_name)
                                time.sleep(0.5) 
                                
                    notifier.send_message(f"✅【追剧落地报告】\n🔗 来源: {share_url}\n📂 成功入库并洗名 {actually_saved_count} 个文件！")
                    if renamed_files_list:
                        if len(renamed_files_list) > 20:
                            rename_msg = "\n".join([f" └ {n}" for n in renamed_files_list[:20]]) + f"\n...等共 {len(renamed_files_list)} 个文件"
                        else:
                            rename_msg = "\n".join([f" └ {n}" for n in renamed_files_list])
                        notifier.send_message(f"✨ 云端洗名规范化完成:\n{rename_msg}")

                if code not in [0, '0', None, False, '']:
                    failed_msg = "\n".join([f" ❌ {t['fileName'][:30]}" for t in failed_tasks[:10]])
                    notifier.send_message(f"⚠️ 触发天翼云特征码拦截 (错误码: {code})！\n拦截/未存上的毒文件有 {len(failed_tasks)} 个:\n{failed_msg}")

                openlist_target_path = get_openlist_path(path)
                
                if path.startswith(DIR_CAS_ROOT) or path.startswith(DIR_CAS_ROOT.strip('/')): 
                    global_cas_paths.add(openlist_target_path) 
                else: 
                    global_emby_paths.add(openlist_target_path)
            else:
                save_json(HISTORY_FILE, history)
                logger.info(f"💤 [安静] {path} 暂无新资源发布。")

            if freq in ["完结", "单次", "电影"]:
                subs_for_update = load_json(SUBS_FILE)
                if str(target_id) in subs_for_update:
                    del subs_for_update[str(target_id)]
                    save_json(SUBS_FILE, subs_for_update)
                    
                    history_data = load_json(HISTORY_FILE)
                    old_len = len(history_data)
                    history_data = {k: v for k, v in history_data.items() if not (isinstance(v, dict) and str(v.get("sub_id")) == str(target_id))}
                    save_json(HISTORY_FILE, history_data)
                    cleaned_count = old_len - len(history_data)
                    
                    logger.info(f"🎉 [归档] 完结撒花：[{path}] 资源已全部归档，清空节点。")
                    notifier.send_message(f"🎉 完结撒花：[{path}] 资源已全部归档！\n✅ 自动解除订阅，并清理了 {cleaned_count} 条关联记忆。")

        except Exception as e:
            error_msg = str(e)
            
            if "SHARE_AUDIT" in error_msg:
                bad_path = path if path else "未知目录"
                logger.warning(f"⚠️ [风控] 遭遇官方拦截: 目录 [{bad_path}] 绑定的链接正在等待审核！引擎已跳过该故障节点。")
                continue

            logger.error(f"❌ [异常] 检查链路异常: {error_msg}")
            
            if "SHARE_DEAD" in error_msg:
                subs_for_update = load_json(SUBS_FILE)
                if str(target_id) in subs_for_update:
                    dead_path = subs_for_update[str(target_id)].get("path", "未知") if isinstance(subs_for_update[str(target_id)], dict) else "未知"
                    del subs_for_update[str(target_id)]
                    save_json(SUBS_FILE, subs_for_update)
                    notifier.send_message(f"❌ 警告：监测到订阅已失效！\n📁 目录: {dead_path}\n🗑️ 已为您清理记忆。")
                    
                history_data = load_json(HISTORY_FILE)
                history_data = {k: v for k, v in history_data.items() if not (isinstance(v, dict) and str(v.get("sub_id")) == str(target_id))}
                save_json(HISTORY_FILE, history_data)
                continue
            elif any(kw in error_msg for kw in ["掉线", "失败", "拦截", "风控", "UNKNOWN_ERROR", "unknown"]): 
                if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                auto_relogin(client_obj, force=True)

    if global_cas_paths:
        s = load_json(SETTINGS_FILE)
        bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
        refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
        local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
        strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
        sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
        real_type = sync_type_map.get(strm_mode_189, "cas")
        strm_dirs_to_refresh = set()
        
        for p in global_cas_paths:
            try:
                requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": p, "type": real_type}, timeout=3) 
                logger.info(f"⚡ [API] 成功向管家后方下发同步指令: {p}")
                notifier.send_message(f"✅ 管家同步指令已下发: {p}")
                
                strm_sub_dir = p.split(DIR_CAS_ROOT)[-1].strip("/")
                target_local_strm_path = os.path.join(local_strm_dir, strm_sub_dir).replace("\\", "/")
                strm_dirs_to_refresh.add(target_local_strm_path)
                
            except Exception as e: 
                logger.error(f"❌ [API] 管家服务无法联通: {e}")
                notifier.send_message(f"❌ 管家同步无响应: {e}")
            time.sleep(1) 
            
        if strm_dirs_to_refresh:
            def delayed_sub_refresh(dirs):
                time.sleep(10.0)
                for d in dirs:
                    subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                    logger.info(f"⚡ [订阅聚合刷新] 已精确唤醒 Emby 局部更新: {d}")
                    time.sleep(1.0)
            threading.Thread(target=delayed_sub_refresh, args=(strm_dirs_to_refresh,)).start()
        
    for p in global_emby_paths:
        try:
            s = load_json(SETTINGS_FILE)
            bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
            refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
            subprocess.Popen([bash_path, refresh_sh, p], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
            logger.info(f"⚡ [脚本] 成功唤醒原生 Emby 刷新: {p}")
            notifier.send_message(f"✅ Emby刷新指令已下发: {p}")
        except: pass
        time.sleep(2)

# ==========================================
# 📦 本地投递箱极速雷达 (全新独立模块，不干扰云端收割)
# ==========================================
def scan_local_dropbox(specific_dir=None):
    s = load_json(SETTINGS_FILE)
    dropbox_dir = s.get("local_dropbox_dir", DEFAULT_LOCAL_DROPBOX)
    local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
    strm_mode = s.get("189_strm_mode", "mode_a_189")
    
    scan_target = specific_dir if specific_dir else dropbox_dir

    db_path = os.path.join(DB_DIR, "monitor_history.db")
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS history (
            file_path TEXT PRIMARY KEY,
            process_time TEXT,
            file_size INTEGER,
            mtime REAL,
            target_strm_dir TEXT,
            strm_name TEXT,
            status TEXT
        )
    ''')

    history_file = os.path.join(DB_DIR, "monitor_history.json")
    if os.path.exists(history_file):
        try:
            old_history = load_json(history_file)
            if old_history:
                for k, v in old_history.items():
                    cursor.execute('''
                        INSERT OR IGNORE INTO history 
                        (file_path, process_time, file_size, mtime, target_strm_dir, strm_name, status)
                        VALUES (?, ?, ?, ?, ?, ?, ?)
                    ''', (k, v.get("process_time"), v.get("file_size"), v.get("mtime"), v.get("target_strm_dir"), v.get("strm_name"), v.get("status")))
                conn.commit()
            os.remove(history_file)
            logger.info("♻️ [系统] 投递箱臃肿的 JSON 记忆库已成功升级为 SQLite 高速数据库！")
        except: pass

    updated_dirs = set()

    cas_files = []
    
    if specific_dir:
        logger.info(f"⏳ [投递箱] 启动智能穿透雷达，寻找 .cas 踪迹...")
        for _ in range(120):
            cas_files = [] 
            if os.path.isfile(scan_target) and scan_target.lower().endswith('.cas'):
                cas_files.append(scan_target.replace("\\", "/"))
            elif os.path.isdir(scan_target):
                for root, dirs, files in os.walk(scan_target):
                    for f in files:
                        if f.lower().endswith('.cas'):
                            cas_files.append(os.path.join(root, f).replace("\\", "/"))
            
            if cas_files:
                time.sleep(3.0) 
                break
            time.sleep(1.0) 
    else:
        if os.path.isfile(scan_target) and scan_target.lower().endswith('.cas'):
            cas_files.append(scan_target.replace("\\", "/"))
        elif os.path.isdir(scan_target):
            for root, dirs, files in os.walk(scan_target):
                for f in files:
                    if f.lower().endswith('.cas'):
                        cas_files.append(os.path.join(root, f).replace("\\", "/"))

    if not cas_files:
        logger.warning(f"⚠️ [投递失败] 扫描完毕，没发现任何 .cas 文件！(传输太慢或路径有误): {scan_target}")
        conn.close()
        return

    for file_path in cas_files:
        try:
            file_stat = os.stat(file_path)
            size = file_stat.st_size
            mtime = file_stat.st_mtime
            filename = file_path.split("/")[-1]

            cursor.execute("SELECT mtime, file_size, target_strm_dir, strm_name FROM history WHERE file_path = ?", (file_path,))
            record = cursor.fetchone()
            
            if record:
                db_mtime, db_size, strm_dir, strm_name = record
                if db_mtime == mtime and db_size == size:
                    if strm_dir and strm_name:
                        strm_full_path = os.path.join(strm_dir, strm_name)
                        if os.path.exists(strm_full_path):
                            continue  
                        else:
                            logger.info(f"♻️ [自动重置] 雷达发现目标 STRM 已被物理删除，无视旧记忆，准备重新生成: {filename}")
                else:
                    continue 

            logger.info(f"📥 [投递箱] 雷达锁定新目标: {file_path}")            

            rel_path = file_path.replace(dropbox_dir, "").strip("/")
            parts = rel_path.split("/")
            filename = parts[-1]

            category_key = parts[0] if len(parts) > 1 else None
            show_folder_name = filename.rsplit('.', 1)[0]

            local_season_num = None
            if len(parts) > 1:
                for part in reversed(parts[:-1]):
                    s_match_dir = re.match(r'(?i)^(?:season\s*|s)(\d+)$', part.strip())
                    if s_match_dir:
                        local_season_num = int(s_match_dir.group(1))
                        continue
                    show_folder_name = part.strip()
                    break

            clean_show_name = re.sub(r'\s*\(\d{4}\)', '', show_folder_name)
            clean_show_name = re.sub(r'(?i)[_\-\s]*(HQ|IQ|DV|4K|1080[pP]|720[pP]|2160[pP]|WEB-DL|HDR|SDR|H265|x265|BluRay|Remux)[_\-\s]*', '', clean_show_name)
            clean_show_name = re.sub(r'[-_\s]+$', '', clean_show_name).strip()

            current_ym = datetime.now().strftime("%Y%m")
            
            if os.path.exists(local_strm_dir):
                for root, dirs, files in os.walk(local_strm_dir):
                    if show_folder_name in dirs:
                        ym_match = re.search(r'/(\d{6})$', root.replace('\\', '/'))
                        if ym_match:
                            current_ym = ym_match.group(1)
                            break

            b_large, b_sub = "未分类", "0-未分类"
            if category_key and category_key in CAT_ROUTER:
                b_large, b_sub = CAT_ROUTER[category_key]
            else:
                for cat_k, (l, s) in CAT_ROUTER.items():
                    if cat_k in rel_path:
                        b_large, b_sub = l, s
                        break
                if b_large == "未分类": b_large, b_sub = "电视剧", "0-电视剧"

            virtual_cloud_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{current_ym}/{show_folder_name}".replace("//", "/")
            
            if local_season_num is not None:
                season_num = local_season_num
            else:
                s_match_file = re.search(r'(?i)(?:^|[^a-z])s0*(\d+)', filename)
                season_num = int(s_match_file.group(1)) if s_match_file else 1

            if b_large in ["电视剧", "动漫", "短剧"]:
                virtual_cloud_path = f"{virtual_cloud_path}/Season {season_num}"
            elif season_num > 1 and b_large not in ["电影", "演唱会"]:
                virtual_cloud_path = f"{virtual_cloud_path}/Season {season_num}"

            final_name = filename
            if final_name.lower().endswith('.cas'):
                strm_name = final_name[:-4] + ".strm"
            else:
                strm_name = final_name + ".strm"

            local_sub_dir = virtual_cloud_path.replace(DIR_CAS_ROOT, "").strip("/")
            target_local_dir = os.path.join(local_strm_dir, local_sub_dir).replace("\\", "/")

            payload = {
                "source_cas_path": file_path,
                "target_local_dir": target_local_dir,
                "strm_name": strm_name,
                "show_name": clean_show_name,
                "mode": strm_mode 
            }

            try:
                res = requests.post(f"{API_5000_URL}/api/make_strm", json=payload, timeout=5)
                if res.status_code == 200:
                    
                    try:
                        final_mtime = os.stat(file_path).st_mtime
                    except:
                        final_mtime = mtime
                        
                    cursor.execute('''
                        INSERT OR REPLACE INTO history 
                        (file_path, process_time, file_size, mtime, target_strm_dir, strm_name, status)
                        VALUES (?, ?, ?, ?, ?, ?, ?)
                    ''', (file_path, datetime.now().strftime("%Y-%m-%d %H:%M:%S"), size, final_mtime, target_local_dir, strm_name, "success"))
                    conn.commit()

                    try:
                        import shutil
                        src_dir_sub = os.path.dirname(file_path)
                        valid_sub_exts = ('.srt', '.ass', '.ssa', '.vtt', '.idx', '.sub')
                        
                        core_name = filename[:-4] if filename.lower().endswith('.cas') else filename
                        for ext in ['.mp4', '.mkv', '.ts', '.iso', '.rmvb', '.avi', '.wmv', '.flv']:
                            if core_name.lower().endswith(ext):
                                core_name = core_name[:-len(ext)]
                                break
                                
                        for sub_f in os.listdir(src_dir_sub):
                            if sub_f.lower().endswith(valid_sub_exts) and sub_f.startswith(core_name):
                                sub_src = os.path.join(src_dir_sub, sub_f)
                                sub_dst = os.path.join(target_local_dir, sub_f)
                                
                                if not os.path.exists(sub_dst) or os.stat(sub_src).st_mtime > os.stat(sub_dst).st_mtime:
                                    shutil.copy2(sub_src, sub_dst) 
                                    logger.info(f"🔤 [字幕] 已无缝同步外挂字幕: {sub_f} -> {target_local_dir}")
                    except Exception as sub_e:
                        logger.error(f"❌ [字幕] 同步外挂字幕失败: {sub_e}")

                    updated_dirs.add(target_local_dir) 
                    
                    ledger_file = os.path.join(DB_DIR, "strm_ledger.json")
                    ledger = load_json(ledger_file)
                    ledger[final_name] = True
                    if len(ledger) > 1000:
                        ledger = dict(list(ledger.items())[-1000:])
                    save_json(ledger_file, ledger)
                else:
                    logger.error(f"❌ [投递箱] API 拒绝: {res.text}")
            except Exception as e:
                logger.error(f"❌ [投递箱] 无法连接到 管家: {e}")

        except Exception as e:
            logger.error(f"❌ [投递箱] 解析异常: {file_path} - {e}")

    conn.close() 

    if updated_dirs:
        logger.info(f"🎬 投递箱批量生成完毕，开始聚合局部刷新 (共 {len(updated_dirs)} 个目录)")
        s = load_json(SETTINGS_FILE)
        bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
        refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
        for d in updated_dirs:
            try:
                subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                logger.info(f"✅ [聚合刷新] 已精准触发单剧目录刷新: {d}")
                time.sleep(0.5)
            except: pass

def main_control_loop(client_obj):
    offset = 0
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    wizard_states = {} 

    logger.info("🚀 [系统] 引擎核心组件自检完毕，执行初次跃迁扫描...")
    check_subscriptions(client_obj, is_first_run=True)
    logger.info("✅ [系统] 预热完毕！引擎正式切入智能静默巡航模式。")

    def scheduled_task():
        settings = load_json(SETTINGS_FILE)
        logger.info("==========================================")
        logger.info("🛸 [系统] 定时唤醒：引擎升空，接管侦测作业...")

        if settings.get("auto_check_subs", True):
            check_subscriptions(client_obj)
        if settings.get("auto_scan_cas", False):
            process_cas_via_olist_api()
            
        if settings.get("auto_scan_local", True):
            scan_local_dropbox()
            scan_139_library()
        
        process_139_pipeline()

        wait_min = random.randint(25, 45)
        logger.info(f"🛌 [系统] 航线巡逻结束。进入节电待机，距下次起飞还有 {wait_min} 分钟...")
        logger.info("==========================================")
        schedule.clear('patrol')
        schedule.every(wait_min).minutes.do(scheduled_task).tag('patrol')

    scheduled_task()
    schedule.every(6).hours.do(auto_relogin, client_obj)

    def daily_force_harvest():
        logger.info("⏰ [定时任务] 触发 23:30 跨月保底强行收割...")
        notifier.send_message("⏰ 触发 23:30 每日保底收割，防止跨月断层...")
        
        p_names = process_cas_via_olist_api()
        
        if p_names:
            msg_str = "\n".join([f" └ {n}" for n in p_names[:20]])
            if len(p_names) > 20: msg_str += f"\n...等共 {len(p_names)} 个文件"
            notifier.send_message(f"✅ 每日保底收割完成:\n{msg_str}")
        else:
            notifier.send_message("✅ 每日保底收割完成，投递箱/云端暂无新文件。")

    schedule.every().day.at("23:30").do(daily_force_harvest)
    
    def cookie_monitor():
        last_check_time = 0
        while True:
            if not os.path.exists(COOKIES_FILE):
                now = time.time()
                if now - last_check_time > 60: 
                    logger.warning("🚨 [探针] 发现 cookies.json 已被删除！立刻触发强制取钥重登...")
                    last_check_time = now
                    if auto_relogin(client_obj, force=True):
                        notifier.send_message("✅ 察觉到外部调用需求，已成功重新获取并刷新 189 账号 Key！")
            time.sleep(0.5)

    threading.Thread(target=cookie_monitor, daemon=True).start()

    while True:
        schedule.run_pending()
        try:
            url = f"https://api.telegram.org/bot{TG_BOT_TOKEN}/getUpdates?offset={offset}&timeout=10"
            res = requests.get(url, timeout=15, proxies=LOCAL_PROXIES).json()
            if res.get('ok'):
                for item in res['result']:
                    offset = item['update_id'] + 1
                    
                    if 'callback_query' in item:
                        cb = item['callback_query']
                        chat_id = cb['message']['chat']['id']
                        msg_id = cb['message']['message_id']
                        data = cb['data']
                        
                        if str(chat_id) != str(TG_ADMIN_USER_ID): continue
                        notifier.answer_callback(cb['id']) 
                        
                        if data == "wiz_cancel":
                            notifier.edit_message(msg_id, "🚫 <b>已取消订阅向导。</b>")
                            wizard_states.pop(chat_id, None)
                            continue
                            
                        if data.startswith("wiz_freq_"):
                            freq = data.split("_")[2]
                            wizard_states[chat_id]["freq"] = freq
                            
                            if freq == "周更":
                                kb = {"inline_keyboard": [
                                    [{"text": "周一", "callback_data": "wiz_day_周一"}, {"text": "周二", "callback_data": "wiz_day_周二"}, {"text": "周三", "callback_data": "wiz_day_周三"}],
                                    [{"text": "周四", "callback_data": "wiz_day_周四"}, {"text": "周五", "callback_data": "wiz_day_周五"}, {"text": "周六", "callback_data": "wiz_day_周六"}],
                                    [{"text": "周日", "callback_data": "wiz_day_周日"}, {"text": "自动/默认", "callback_data": "wiz_day_未知"}],
                                    [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                                ]}
                                notifier.edit_message(msg_id, f"✅ 剧名: {wizard_states[chat_id]['title']}\n✅ 频率: {freq}\n\n<b>📅 请选择该剧的更新时间 (周几):</b>", kb)
                            else:
                                kb = {"inline_keyboard": [
                                    [{"text": "📺 华语剧", "callback_data": "wiz_cat_华语剧"}, {"text": "📺 欧美剧", "callback_data": "wiz_cat_欧美剧"}],
                                    [{"text": "🎬 华语电影", "callback_data": "wiz_cat_华语电影"}, {"text": "🎬 欧美电影", "callback_data": "wiz_cat_欧美电影"}],
                                    [{"text": "🐼 日漫/番剧", "callback_data": "wiz_cat_日漫"}, {"text": "🐼 国漫", "callback_data": "wiz_cat_国漫"}],
                                    [{"text": "📺 日韩剧", "callback_data": "wiz_cat_日韩剧"}, {"text": "🎬 日韩电影", "callback_data": "wiz_cat_日韩电影"}],
                                    [{"text": "🎤 综艺", "callback_data": "wiz_cat_综艺"}, {"text": "🎥 纪录片", "callback_data": "wiz_cat_纪录片"}],
                                    [{"text": "📱 短剧", "callback_data": "wiz_cat_短剧"}, {"text": "🎵 演唱会", "callback_data": "wiz_cat_演唱会"}],
                                    [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                                ]}
                                notifier.edit_message(msg_id, f"✅ 剧名: {wizard_states[chat_id]['title']}\n✅ 频率: {freq}\n\n<b>请选择【精确分类】(匹配路由):</b>", kb)
                            continue

                        elif data.startswith("wiz_day_"):
                            day = data.split("_")[2]
                            if day != "未知":
                                wizard_states[chat_id]["day"] = day
                                
                            kb = {"inline_keyboard": [
                                [{"text": "📺 华语剧", "callback_data": "wiz_cat_华语剧"}, {"text": "📺 欧美剧", "callback_data": "wiz_cat_欧美剧"}],
                                [{"text": "🎬 华语电影", "callback_data": "wiz_cat_华语电影"}, {"text": "🎬 欧美电影", "callback_data": "wiz_cat_欧美电影"}],
                                [{"text": "🐼 日漫/番剧", "callback_data": "wiz_cat_日漫"}, {"text": "🐼 国漫", "callback_data": "wiz_cat_国漫"}],
                                [{"text": "📺 日韩剧", "callback_data": "wiz_cat_日韩剧"}, {"text": "🎬 日韩电影", "callback_data": "wiz_cat_日韩电影"}],
                                [{"text": "🎤 综艺", "callback_data": "wiz_cat_综艺"}, {"text": "🎥 纪录片", "callback_data": "wiz_cat_纪录片"}],
                                [{"text": "📱 短剧", "callback_data": "wiz_cat_短剧"}, {"text": "🎵 演唱会", "callback_data": "wiz_cat_演唱会"}],
                                [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                            ]}
                            day_str = f" ({day})" if day != "未知" else ""
                            notifier.edit_message(msg_id, f"✅ 剧名: {wizard_states[chat_id]['title']}\n✅ 频率: {wizard_states[chat_id]['freq']}{day_str}\n\n<b>请选择【精确分类】(匹配路由):</b>", kb)
                            continue

                        elif data.startswith("wiz_cat_"):
                            wizard_states[chat_id]["cat"] = data.split("_")[2]
                            kb = {"inline_keyboard": [
                                [{"text": "🎥 仅存视频 (MP4/MKV等)", "callback_data": "wiz_type_视频"}],
                                [{"text": "🗂️ 仅存 CAS (秒传文件)", "callback_data": "wiz_type_CAS"}],
                                [{"text": "📦 全盘转存 (不过滤)", "callback_data": "wiz_type_全盘"}],
                                [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                            ]}
                            day_str = f" ({wizard_states[chat_id]['day']})" if "day" in wizard_states[chat_id] else ""
                            notifier.edit_message(msg_id, f"✅ 频率: {wizard_states[chat_id]['freq']}{day_str}\n✅ 分类: {wizard_states[chat_id]['cat']}\n\n<b>请选择【文件过滤规则】:</b>", kb)
                            continue
                            
                        elif data.startswith("wiz_type_"):
                            f_type = data.split("_")[2]
                            state = wizard_states.pop(chat_id)
                            
                            kw_map = {"视频": ".mp4 .mkv .ts", "CAS": ".cas", "全盘": ""}
                            kw = kw_map.get(f_type, "")
                            
                            day_tag = f" #{state['day']}" if "day" in state else ""
                            s_cmd = f"订阅{state['s_num']}" 
                            
                            cmd = f"{s_cmd} {state['title']} {state['url']} #{state['freq']}{day_tag} #{state['cat']} {kw}".strip()
                            
                            notifier.edit_message(msg_id, f"🎉 <b>向导收集完毕！</b>\n正在为您下发指令:\n<code>{cmd}</code>")
                            
                            item['message'] = {'chat': {'id': chat_id}, 'text': cmd}
                        
                        elif data.startswith("wiz_feed_"):
                            url_code = data.replace("wiz_feed_", "")
                            full_url = f"https://cloud.189.cn/t/{url_code}"
                            
                            wizard_states[chat_id] = {"step": 1, "url": full_url}
                            kb = {"inline_keyboard": [[{"text": "❌ 取消", "callback_data": "wiz_cancel"}]]}
                            notifier.edit_message(msg_id, f"🔗 <b>已从内部广场锁定直链！</b>\n\n✏️ 请直接回复本条消息，输入【干净剧名(年份)】\n<i>(如带季数，请直接写: 庆余年 2)</i>", kb)
                            continue

                    msg = item.get('message', {})
                    text = msg.get('text', '')
                    chat_id = msg.get('chat', {}).get('id')

                    logger.info(f"🚨【强力抓包】收到消息 -> 聊天ID: {chat_id} | 纯文本内容: [{text}]")

                    if str(chat_id) == str(TG_ADMIN_USER_ID) or str(chat_id).startswith("-"):
                        text = text.strip()
                        
                        if text.startswith("查字典") or text == "/dict":
                            r_data = load_json(ROUTER_FILE)
                            msg = "📖 <b>当前系统动态路由字典：</b>\n\n"
                            for k, v in r_data.items():
                                sub_str = f" / {v[1]}" if len(v) > 1 and v[1] else ""
                                msg += f"🏷️ <code>{k}</code> ➔ 📁 {v[0]}{sub_str}\n"
                            msg += "\n💡 <b>增加:</b> 加字典 [识别词] [大分类] [小分类(可选)]\n(例: 加字典 漫威 电影 1-电影)\n💡 <b>删除:</b> 删字典 [识别词]"
                            notifier.send_message(msg)
                            continue

                        elif text.startswith("加字典") or text.startswith("/adddict"):
                            parts = text.split()
                            if len(parts) < 3:
                                notifier.send_message("❌ 格式错误！\n示例：加字典 漫威 电影 1-电影\n英文示例：/adddict 纪录片 纪录片")
                                continue
                            kw = parts[1].strip()
                            l_cat = parts[2].strip()
                            s_cat = parts[3].strip() if len(parts) > 3 else ""
                            
                            r_data = load_json(ROUTER_FILE)
                            r_data[kw] = [l_cat, s_cat]
                            save_json(ROUTER_FILE, r_data)
                            reload_globals()
                            notifier.send_message(f"✅ 字典添加成功并已热重载生效！\n🏷️ 识别词: {kw}\n📁 路由至: {l_cat} / {s_cat}")
                            continue

                        elif text.startswith("删字典") or text.startswith("/deldict"):
                            parts = text.split()
                            if len(parts) < 2:
                                notifier.send_message("❌ 格式错误！\n示例：删字典 漫威\n英文示例：/deldict 漫威")
                                continue
                            kw = parts[1].strip()
                            r_data = load_json(ROUTER_FILE)
                            if kw in r_data:
                                del r_data[kw]
                                save_json(ROUTER_FILE, r_data)
                                reload_globals()
                                notifier.send_message(f"✅ 已从动态字典中物理删除并重载内存：{kw}")
                            else:
                                notifier.send_message(f"❌ 字典中不存在该特征词：{kw}")
                            continue

                        elif text == "查配置" or text == "/config":
                            s_data = load_json(SETTINGS_FILE)
                            msg = "⚙️ <b>系统核心配置 (settings.json)</b>\n\n"
                            for k, v in s_data.items():
                                msg += f"▪️ <b>{k}</b> : <code>{v}</code>\n"
                            msg += "\n💡 <b>修改指令:</b> 设配置 [参数名] [新值]\n(例: /setconfig DIR_CAS_ROOT /新目录)"
                            notifier.send_message(msg)
                            continue

                        elif text.startswith("设配置") or text.startswith("/setconfig"):
                            parts = text.split(maxsplit=2)
                            if len(parts) < 3:
                                notifier.send_message("❌ 格式错误！\n示例：设配置 DIR_CAS_ROOT /新目录")
                                continue
                            key = parts[1].strip()
                            val = parts[2].strip()
                            
                            if val.lower() == "true": val = True
                            elif val.lower() == "false": val = False
                            elif val.isdigit(): val = int(val)
                            
                            s_data = load_json(SETTINGS_FILE)
                            s_data[key] = val
                            save_json(SETTINGS_FILE, s_data)
                            reload_globals()
                            notifier.send_message(f"✅ 配置修改成功并已热拔插生效！\n⚙️ {key} = {val}")
                            continue

                        elif text.startswith("加库") or text.startswith("/hsub"):
                            try:
                                if text.startswith("加库"):
                                    clean_text = text[2:].strip()
                                else:
                                    clean_text = text[5:].strip()
                                    
                                parts = clean_text.split()
                                if len(parts) < 2:
                                    notifier.send_message("格式错误！\n常规：加库 剧名(年份) 分类\n指定老月：加库 剧名 分类 202604")
                                    continue
                                
                                if parts[-1].startswith("/"):
                                    cloud_path = parts[-1].strip()
                                    show_name = " ".join(parts[:-1]).strip()
                                elif re.match(r'^\d{6}$', parts[-1]):
                                    target_ym = parts[-1].strip()
                                    category_key = parts[-2].strip()
                                    show_name = " ".join(parts[:-2]).strip()
                                    b_large, b_sub = CAT_ROUTER.get(category_key, ("未分类", "0-未分类"))
                                    cloud_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{target_ym}/{show_name}".replace("//", "/")
                                else:
                                    target_ym = datetime.now().strftime("%Y%m")
                                    category_key = parts[-1].strip()
                                    show_name = " ".join(parts[:-1]).strip()
                                    b_large, b_sub = CAT_ROUTER.get(category_key, ("未分类", "0-未分类"))
                                    cloud_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{target_ym}/{show_name}".replace("//", "/")

                                search_key = get_match_key(show_name)

                                if not search_key:
                                    notifier.send_message("❌ 剧名无法提取有效特征词，建档失败！")
                                    continue
                                    
                                h_data = load_json(HARVEST_SUBS_FILE)
                                h_data[search_key] = cloud_path
                                save_json(HARVEST_SUBS_FILE, h_data)
                                notifier.send_message(f"✅ 收割记录建档成功！\n📺 剧名：{show_name}\n🔍 特征词：{search_key}\n📂 目标路径：{cloud_path}")
                            except Exception as e:
                                notifier.send_message(f"🚨 代码崩溃抓包：\n<code>{str(e)}</code>")
                            continue 
                        
                        elif text.startswith("设置模式") or text.startswith("/mode") and not text.startswith("/mode139"):
                            mode_input = text.replace("设置模式", "").replace("/mode", "").strip().upper()
                            mode_map = {"A": "mode_a_189", "B": "mode_b_189", "C": "both_189"}
                            
                            if mode_input not in mode_map:
                                notifier.send_message("❌ 模式错误！\n可选模式:\nA - 模式A (189 官方秒传直通 -> .strm)\nB - 模式B (189 OpenList中转 -> -189B.strm)\nC - 双轨模式 (同时生成 A 和 B)\n示例: /mode A")
                                continue
                                
                            real_mode = mode_map[mode_input]
                            s = load_json(SETTINGS_FILE)
                            s["189_strm_mode"] = real_mode
                            save_json(SETTINGS_FILE, s)
                            notifier.send_message(f"✅ 成功！以后 189 投递管家将统一使用模式: {mode_input} ({real_mode})")
                            continue

                        elif text.startswith("139设置模式") or text.startswith("/mode139"):
                            mode_input = text.replace("139设置模式", "").replace("/mode139", "").strip().upper()
                            mode_map = {"A": "mode_a_139", "B": "mode_b_139", "C": "both_139"}
                            
                            if mode_input not in mode_map:
                                notifier.send_message(
                                    "❌ 模式错误！\n"
                                    "可选模式:\n"
                                    "A - 模式A (139 官方原生直连 -> -139A.strm)\n"
                                    "B - 模式B (139 老模式中转 -> -139B.strm)\n"
                                    "C - 双轨模式 (同时生成 A 和 B)\n\n"
                                    "💡 无论选哪种，生成的 strm 都会统一放在唯一的 139 媒体库目录中。\n"
                                    "👉 示例: /mode139 B"
                                )
                                continue
                                
                            real_mode = mode_map[mode_input]
                            s = load_json(SETTINGS_FILE)
                            s["139_strm_mode"] = real_mode
                            save_json(SETTINGS_FILE, s)
                            notifier.send_message(f"✅ 设置成功！\n\n当前 139 专属流水线已切换为模式: {mode_input} ({real_mode})\n🎯 以后生成的 strm 内部将采用该模式的链接，并统一放入 139 单一媒体库中！")
                            continue

                        elif text.startswith("删库") or text.startswith("/dsub"):
                            try:
                                kw = text[2:].strip() if text.startswith("删库") else text[5:].strip()
                                if not kw:
                                    notifier.send_message("格式错误！\n示例：删库 师兄啊师兄")
                                    continue
                                    
                                h_file = globals().get('HARVEST_SUBS_FILE', os.path.join(DB_DIR, "harvest_subs.json"))
                                h_data = load_json(h_file)
                                
                                if not h_data:
                                    notifier.send_message("📭 收割记录库为空，没什么可删的。")
                                    continue

                                try:
                                    search_key = get_match_key(kw)
                                except NameError:
                                    c = re.sub(r'[（\(\[\{]?\d{4}[）\)\]\}]?', '', kw)
                                    c = re.sub(r'(?i)\b(4k|1080p|2160p|web-dl|sdr|hdr)\b', '', c)
                                    c = re.sub(r'(完结|连载中|全\d+集|打包|修正)', '', c)
                                    search_key = re.sub(r'[^\w\u4e00-\u9fa5]', '', c).lower()

                                deleted_items = []
                                
                                if search_key and search_key in h_data:
                                    deleted_items.append((search_key, h_data[search_key]))
                                    del h_data[search_key]
                                else:
                                    keys_to_delete = []
                                    for k, v in h_data.items():
                                        if kw.lower() in k or kw.lower() in v.lower():
                                            keys_to_delete.append(k)
                                    for k in keys_to_delete:
                                        deleted_items.append((k, h_data[k]))
                                        del h_data[k]
                                        
                                if deleted_items:
                                    save_json(h_file, h_data)
                                    msg = "✅ 已成功从收割库中删除以下记录：\n"
                                    for k, p in deleted_items:
                                        msg += f" └ {k}"
                                    notifier.send_message(msg)
                                else:
                                    notifier.send_message(f"❌ 没找到与“{kw}”相关的收割记录。")
                                    
                            except Exception as e:
                                notifier.send_message(f"🚨 删库指令报错：\n<code>{str(e)}</code>")
                            continue

                        elif text == "查库" or text == "/lsub":
                            try:
                                h_file = globals().get('HARVEST_SUBS_FILE', os.path.join(DB_DIR, "harvest_subs.json"))
                                h_data = load_json(h_file)
                                if not h_data:
                                    notifier.send_message("📭 当前收割库为空，没有任何手动建档的记录。")
                                    continue
                                
                                msg_lines = ["📋 <b>【专属收割库】当前记录：</b>\n"]
                                for i, (k, p) in enumerate(h_data.items(), 1):
                                    msg_lines.append(f"{i}. <b>{k}</b>\n   └ 📁 {p}")
                                msg_lines.append("\n💡 提示：回复“删库 剧名”即可删除对应记录。")
                                notifier.send_message("\n".join(msg_lines))
                            except Exception as e:
                                notifier.send_message(f"🚨 查库指令报错：\n<code>{str(e)}</code>")
                            continue

                        elif text == "查目录" or text == "/ldir":
                            s = load_json(SETTINGS_FILE)
                            watch_dirs = s.get("watch_dirs", ["/family/177_cas", "/local_cas"])
                            if not watch_dirs:
                                notifier.send_message("📭 当前没有配置任何巡逻目录，收割兵正在集体放假。")
                                continue
                            
                            msg = "📋 <b>当前自动收割的【巡逻路线】：</b>\n\n"
                            for i, wd in enumerate(watch_dirs, 1):
                                msg += f"{i}. 📁 <code>{wd}</code>\n"
                            msg += "\n💡 提示：回复“加目录 路径”或“删目录 序号”进行动态调整。"
                            notifier.send_message(msg)
                            continue

                        elif text.startswith("加目录") or text.startswith("/adir"):
                            new_dir = text[3:].strip() if text.startswith("加目录") else text[5:].strip()
                            if not new_dir:
                                notifier.send_message("格式错误！\n示例：加目录 /177-临时收割")
                                continue
                            
                            s = load_json(SETTINGS_FILE)
                            watch_dirs = s.get("watch_dirs", ["/family/177_cas", "/local_cas"])
                            
                            if new_dir not in watch_dirs:
                                watch_dirs.append(new_dir)
                                s["watch_dirs"] = watch_dirs
                                save_json(SETTINGS_FILE, s)
                                notifier.send_message(f"✅ 成功划定新的巡逻战区：\n📁 {new_dir}\n(下次收割时生效)")
                            else:
                                notifier.send_message(f"⚠️ 该目录已经在巡逻路线中了，无需重复添加：\n📁 {new_dir}")
                            continue

                        elif text.startswith("删目录") or text.startswith("/ddir"):
                            del_dir = text[3:].strip() if text.startswith("删目录") else text[5:].strip()
                            if not del_dir:
                                notifier.send_message("格式错误！\n示例：删目录 1\n(发“查目录”看序号，直接填序号或名字删)")
                                continue
                                
                            s = load_json(SETTINGS_FILE)
                            watch_dirs = s.get("watch_dirs", ["/family/177_cas", "/local_cas"])
                            
                            target_to_del = None
                            if del_dir.isdigit() and 1 <= int(del_dir) <= len(watch_dirs):
                                target_to_del = watch_dirs[int(del_dir) - 1]
                            else:
                                for wd in watch_dirs:
                                    if del_dir.lower() in wd.lower():
                                        target_to_del = wd
                                        break
                                        
                            if target_to_del:
                                watch_dirs.remove(target_to_del)
                                s["watch_dirs"] = watch_dirs
                                save_json(SETTINGS_FILE, s)
                                notifier.send_message(f"✅ 已撤销该战区的巡逻任务：\n🗑️ {target_to_del}")
                            else:
                                notifier.send_message(f"❌ 没找到匹配的目录：{del_dir}")
                            continue
                            
                        elif text == "查个人云" or text == "/listcopy":
                            s = load_json(SETTINGS_FILE)
                            nd_list = s.get("no_delete_dirs", [])
                            if nd_list:
                                msg = "📂 <b>当前免删复制(个人云)线路</b>：\n\n"
                                for i, d in enumerate(nd_list, 1):
                                    msg += f"{i}. 📑 <code>{d}</code>\n"
                                msg += "\n💡 提示：以上目录下的 CAS 文件只执行复制，保留个人云源文件。"
                                notifier.send_message(msg)
                            else:
                                notifier.send_message("📂 当前免删名单为空，所有目录均执行默认的【移动+删除】。")
                            continue

                        elif text.startswith("加个人云") or text.startswith("/addcopy"):
                            new_dir = text[4:].strip() if text.startswith("加个人云") else text[8:].strip()
                            if not new_dir:
                                notifier.send_message("格式错误！\n示例：加个人云 /个人云/下载")
                                continue
                            
                            s = load_json(SETTINGS_FILE)
                            nd_list = s.get("no_delete_dirs", [])
                            if new_dir not in nd_list:
                                nd_list.append(new_dir)
                                s["no_delete_dirs"] = nd_list
                                save_json(SETTINGS_FILE, s)
                                notifier.send_message(f"✅ 成功划定免流复制战区：\n📂 {new_dir}\n(下次收割时生效，只复制不删CAS)")
                            else:
                                notifier.send_message(f"⚠️ 该目录已经在免删名单中了：\n📂 {new_dir}")
                            continue

                        elif text.startswith("删个人云") or text.startswith("/rmcopy"):
                            del_dir = text[4:].strip() if text.startswith("删个人云") else text[7:].strip()
                            if not del_dir:
                                notifier.send_message("格式错误！\n示例：删个人云 /个人云/下载")
                                continue
                                
                            s = load_json(SETTINGS_FILE)
                            nd_list = s.get("no_delete_dirs", [])
                            if del_dir in nd_list:
                                nd_list.remove(del_dir)
                                s["no_delete_dirs"] = nd_list
                                save_json(SETTINGS_FILE, s)
                                notifier.send_message(f"🗑️ 已移出名单！以后该目录恢复为常规【移动并删除】模式：\n📂 {del_dir}")
                            else:
                                notifier.send_message(f"⚠️ 找不到该目录，请检查路径是否正确：\n📂 {del_dir}")
                            continue   

                        elif text.startswith("139同步") or text.startswith("/sync139"):
                            target_path = text.replace("139同步", "").replace("/sync139", "").strip()
                            if not target_path:
                                notifier.send_message("❌ 格式错误！\n示例：139同步 天才\n(也支持直接发送绝对路径)")
                                continue
                            
                            def bg_sync_139():
                                target_paths = []
                                if target_path.startswith("/"):
                                    target_paths.append(target_path)
                                else:
                                    notifier.send_message(f"🔍 启动 139 智能雷达，搜索【{target_path}】...")
                                    try:
                                        r_log = requests.post(f"{API_5255_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                        if r_log.get("code") == 200:
                                            h_139 = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                            
                                            r_cats = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": DIR_139_TARGET}, headers=h_139, timeout=5).json()
                                            cats = [c["name"] for c in (r_cats.get("data") or {}).get("content", []) if c["is_dir"]]
                                            for cat in cats:
                                                cat_path = f"{DIR_139_TARGET}/{cat}"
                                                r_shows = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": cat_path}, headers=h_139, timeout=5).json()
                                                shows = (r_shows.get("data") or {}).get("content", [])
                                                for s_itm in shows:
                                                    if s_itm["is_dir"] and target_path.lower() in s_itm["name"].lower():
                                                        show_path = f"{cat_path}/{s_itm['name']}"
                                                        r_seasons = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": show_path}, headers=h_139, timeout=5).json()
                                                        seasons = [ss["name"] for ss in (r_seasons.get("data") or {}).get("content", []) if ss["is_dir"] and "season" in ss["name"].lower()]
                                                        if seasons:
                                                            for s_name in seasons: target_paths.append(f"{show_path}/{s_name}")
                                                        else: target_paths.append(show_path)
                                        if not target_paths:
                                            notifier.send_message(f"📭 遍历完毕，未找到与【{target_path}】相关的文件。")
                                            return
                                    except Exception as e:
                                        notifier.send_message(f"⚠️ 139 智能寻轨网络异常: {e}")
                                        return
                                        
                                refresh_dirs = set()
                                for tp in target_paths:
                                    try:
                                        t_strm_dir = sync_139_to_smart_strm(tp)
                                        if t_strm_dir: refresh_dirs.add(t_strm_dir)
                                    except Exception as e:
                                        logger.error(f"❌ [139同步] 升维翻译异常 ({tp}): {e}")

                                msg_list = "\n".join([f" └ 📁 <code>{p}</code>" for p in target_paths])
                                notifier.send_message(f"🎯 成功锁定 {len(target_paths)} 个目标并执行规则核对：\n{msg_list}")

                                if refresh_dirs:
                                    s = load_json(SETTINGS_FILE)
                                    bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                    refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                    
                                    def delayed_manual_139_refresh(dirs_to_refresh):
                                        time.sleep(15.0)
                                        for r_dir in dirs_to_refresh:
                                            try:
                                                subprocess.Popen([bash_path, refresh_sh, r_dir], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                            except: pass
                                            time.sleep(1.0)
                                    threading.Thread(target=delayed_manual_139_refresh, args=(refresh_dirs,)).start()
                            threading.Thread(target=bg_sync_139).start()
                            continue

                        elif text in ["139全库", "/scan139"]:
                            notifier.send_message("🚀 启动 139 全域扫荡雷达！\n正在应用 SQLite 高速记忆库执行补漏与升维建档...")
                            def bg_scan139():
                                try:
                                    rescued = scan_139_library()
                                    if rescued:
                                        notifier.send_message(f"✅ 139 全域扫荡完毕！\n🎉 成功处理了 {rescued} 个发生变动的目录！")
                                    else:
                                        notifier.send_message("✅ 139 扫荡完毕，SQLite 记忆库比对一致，无文件变动。")
                                except Exception as e:
                                    notifier.send_message(f"❌ 扫荡异常: {e}")
                            threading.Thread(target=bg_scan139).start()
                            continue

                        elif text in ["开启自动收割", "开启扫描", "/ascan"]:
                            s = load_json(SETTINGS_FILE); s["auto_scan_cas"] = True; save_json(SETTINGS_FILE, s)
                            notifier.send_message("✅ 已【开启】CAS自动收割巡逻。")
                            continue
                        elif text in ["关闭自动收割", "关闭扫描", "/sscan"]:
                            s = load_json(SETTINGS_FILE); s["auto_scan_cas"] = False; save_json(SETTINGS_FILE, s)
                            notifier.send_message("⏸️ 已【关闭】CAS自动收割巡逻。")
                            continue
                        elif text in ["开启自动订阅", "开启订阅检查", "/asub"]:
                            s = load_json(SETTINGS_FILE); s["auto_check_subs"] = True; save_json(SETTINGS_FILE, s)
                            notifier.send_message("✅ 已【开启】定时订阅拉取。")
                            continue
                        elif text in ["关闭自动订阅", "关闭订阅检查", "/ssub"]:
                            s = load_json(SETTINGS_FILE); s["auto_check_subs"] = False; save_json(SETTINGS_FILE, s)
                            notifier.send_message("⏸️ 已【关闭】定时订阅拉取。")
                            continue
                        elif text in ["同步订阅", "立即拉取", "全部拉取", "更新订阅", "/sync"]:
                            notifier.send_message("🚀 正在强行冲破时间门槛，全量拉取订阅中...")
                            check_subscriptions(client_obj, ignore_time=True)
                            notifier.send_message("✅ 同步拉取任务彻底执行完毕。")
                            continue
                        elif text in ["收割", "处理", "添加", "/harvest"]:
                            notifier.send_message("📥 收到【收割】指令：任务已分配到后台，正在为您洗名并入库 CAS 文件...")
                            def bg_harvest():
                                p_names = process_cas_via_olist_api()
                                if p_names:
                                    msg_str = "\n".join([f" └ {n}" for n in p_names[:20]])
                                    if len(p_names) > 20: msg_str += f"\n...等共 {len(p_names)} 个文件"
                                    notifier.send_message(f"✅ CAS 收割入库完成:\n{msg_str}")
                                else:
                                    notifier.send_message("✅ CAS 收割完成，暂无新文件。")
                            threading.Thread(target=bg_harvest).start()
                            continue
                        elif text in ["139收割", "139加工", "/139hv"]:
                            notifier.send_message("🚀 收到指令：任务已分配到后台，正在为您启动 139 专属加工流水线...")
                            def bg_139():
                                try:
                                    process_139_pipeline()
                                    notifier.send_message("✅ 139 加工流水线本次作业执行完毕！")
                                except Exception as e:
                                    notifier.send_message(f"❌ 139 流水线执行报错: {e}")
                            threading.Thread(target=bg_139).start()
                            continue
                        elif text in ["扫箱子", "投递", "本地扫描", "/dropbox"]:
                            notifier.send_message("🚀 收到指令：任务已分配到后台，立刻启动本地投递箱极速雷达...")
                            def bg_dropbox():
                                try:
                                    scan_local_dropbox()
                                    notifier.send_message("✅ 本地投递箱雷达扫描与下发任务已执行完毕！")
                                except Exception as e:
                                    notifier.send_message(f"❌ 扫描本地投递箱时发生异常: {e}")
                            threading.Thread(target=bg_dropbox).start()
                            continue
                        elif text in ["动态", "广场", "上新", "/feed"]:
                            notifier.send_message("📡 正在连接订阅中心，拉取最新情报...")
                            try:
                                res = client_obj.session.get("https://cloud.189.cn/api/open/share/getOwnerSubscribeShare.action?pageNum=1&pageSize=8", timeout=10).json()
                                if res.get("code") == "success":
                                    file_list = res.get("data", {}).get("shareFileList", [])
                                    if not file_list:
                                        notifier.send_message("📭 订阅中心目前没有任何更新。")
                                        continue
                                    
                                    msg_lines = ["📡 <b>【订阅中心】最新动态：</b>\n"]
                                    kb_buttons = []
                                    
                                    for i, item in enumerate(file_list, 1):
                                        raw_name = item.get("name", "未知资源")
                                        name = translate_folder_name(raw_name)
                                        author = item.get("ownerAccount", "未知发布者")
                                        url_code = item.get("accessURL", "")
                                        date_str = item.get("lastOpTime", "")[5:16] 
                                        
                                        if not url_code: continue
                                        msg_lines.append(f"{i}. 📁 <code>{name}</code>\n   └ 👤 {author} | ⏱ {date_str}\n")
                                        btn_text = name.replace("📺 ", "").replace("🎬 ", "")
                                        kb_buttons.append([{"text": f"📥 订阅: {btn_text[:12]}...", "callback_data": f"wiz_feed_{url_code}"}])
                                        
                                    kb_buttons.append([{"text": "❌ 取消", "callback_data": "wiz_cancel"}])
                                    kb = {"inline_keyboard": kb_buttons}
                                    notifier.send_message("\n".join(msg_lines), kb)
                                else:
                                    raise Exception(f"接口掉线/风控拦截: {res}")
                            except Exception as e:
                                notifier.send_message(f"❌ 广场拉取异常: {e}")
                                err_str = str(e).upper()
                                if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                    if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                    auto_relogin(client_obj, force=True)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
                            continue

                        elif text.startswith("查剧 ") or text.startswith("信息 ") or text.startswith("/info "):
                            keyword = text.split(" ", 1)[1].strip()
                            if not keyword: continue
                            
                            notifier.send_message(f"🔍 正在连接 TMDB 全球数据库，检索【{keyword}】的档案...")
                            try:
                                info_msg = fetch_tmdb_rich_info(keyword)
                                kb = {"inline_keyboard": [[{"text": "❌ 关闭面板", "callback_data": "wiz_cancel"}]]}
                                notifier.send_message(info_msg, kb)
                            except Exception as e:
                                notifier.send_message(f"❌ 检索失败: {e}")
                            continue

                        elif text.startswith("搜 ") or text.startswith("搜索 ") or text.startswith("/search "):
                            raw_keyword = text.split(" ", 1)[1].strip()
                            if not raw_keyword: continue
                            
                            notifier.send_message(f"🧠 正在请求 TMDB 反查 【{raw_keyword}】 的全维特征码...")
                            tmdb_info = get_tmdb_info(raw_keyword)
                            
                            keywords = set()
                            keywords.add(raw_keyword.lower())
                            if " " in raw_keyword:
                                for k in raw_keyword.split():
                                    if len(k) > 1: keywords.add(k.lower())
                                    
                            if tmdb_info:
                                if tmdb_info.get('en_name'): keywords.add(tmdb_info['en_name'].lower())
                                if tmdb_info.get('pinyin_full'): keywords.add(tmdb_info['pinyin_full'].lower())
                                if tmdb_info.get('pinyin_initial'): keywords.add(tmdb_info['pinyin_initial'].lower())
                                if tmdb_info.get('id'): 
                                    keywords.add(f"tmdb-{tmdb_info['id']}")
                                    keywords.add(f"tmdb{tmdb_info['id']}")
                                    keywords.add(str(tmdb_info['id']))
                                    
                            keyword_list = list(keywords)
                            display_kw = " | ".join(keyword_list)
                            
                            notifier.send_message(f"🔍 锁定全维特征码: 【{display_kw}】\n启动地毯式穿甲雷达，深入主页挖掘...")
                            try:
                                active_users = {} 
                                for feed_page in range(1, 5):
                                    feed_url = f"https://cloud.189.cn/api/open/share/getOwnerSubscribeShare.action?pageNum={feed_page}&pageSize=100"
                                    res_feed = client_obj.session.get(feed_url, timeout=15).json()
                                    
                                    if res_feed.get("code") == "success":
                                        items = res_feed.get("data", {}).get("shareFileList", [])
                                        if not items: break
                                        for item in items:
                                            uid = item.get("upUserId")
                                            name = item.get("ownerAccount", "未知大佬")
                                            if uid: active_users[uid] = name
                                    else:
                                        raise Exception(f"接口掉线/风控拦截: {res_feed}")
                                
                                if not active_users:
                                    notifier.send_message("❌ 广场空空如也，未获取到任何订阅大佬的信息。")
                                    continue
                                
                                matched_items = []
                                for uid, uname in active_users.items():
                                    for page in range(1, 50):
                                        url = f"https://cloud.189.cn/api/open/share/getUpResourceShare.action?pageNum={page}&pageSize=30&upUserId={uid}"
                                        res_user = client_obj.session.get(url, timeout=10).json()
                                        
                                        if res_user.get("code") == "success":
                                            items = res_user.get("data", {}).get("fileList", []) 
                                            if not items: break 
                                            for item in items:
                                                item_name_lower = item.get("name", "").lower()
                                                if any(kw in item_name_lower for kw in keyword_list):
                                                    item["ownerAccount"] = uname 
                                                    matched_items.append(item)
                                        else:
                                            raise Exception(f"扒主页时掉线/风控拦截: {res_user}")
                                
                                unique_matches = []
                                seen_urls = set()
                                for item in matched_items:
                                    url_code = item.get("accessURL", "")
                                    if url_code and url_code not in seen_urls:
                                        seen_urls.add(url_code)
                                        unique_matches.append(item)

                                if not unique_matches:
                                    notifier.send_message(f"📭 翻遍了 {len(active_users)} 位大佬的个人历史主页，没找到相关的资源。")
                                    continue
                                    
                                msg_lines = [f"🎯 <b>为您精准捞到了 {len(unique_matches)} 个相关资源：</b>\n"]
                                kb_buttons = []
                                
                                for i, item in enumerate(unique_matches[:8], 1):
                                    raw_name = item.get("name", "未知资源")
                                    name = translate_folder_name(raw_name)
                                    author = item.get("ownerAccount", "未知发布者")
                                    url_code = item.get("accessURL", "")
                                    date_str = item.get("lastOpTime", "")[5:16]
                                    
                                    if not url_code: continue
                                    msg_lines.append(f"{i}. 📁 <code>{name}</code>\n   └ 👤 {author} | ⏱ {date_str}\n")
                                    btn_text = name.replace("📺 ", "").replace("🎬 ", "")
                                    kb_buttons.append([{"text": f"📥 订阅: {btn_text[:12]}...", "callback_data": f"wiz_feed_{url_code}"}])
                                    
                                kb_buttons.append([{"text": "❌ 取消", "callback_data": "wiz_cancel"}])
                                kb = {"inline_keyboard": kb_buttons}
                                
                                notifier.send_message("\n".join(msg_lines), kb)
                            except Exception as e:
                                notifier.send_message(f"❌ 搜索拉取异常: {e}")
                                err_str = str(e).upper()
                                if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                    if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                    auto_relogin(client_obj, force=True)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
                            continue

                        elif text.startswith("查人 ") or text.startswith("查作者 ") or text.startswith("/author "):
                            author_kw = text.split(" ", 1)[1].strip()
                            if not author_kw: continue
                            
                            notifier.send_message(f"🕵️‍♂️ 启动多账号联合扫描：正在锁定所有包含【{author_kw}】的大佬...")
                            try:
                                res_feed = client_obj.session.get("https://cloud.189.cn/api/open/share/getOwnerSubscribeShare.action?pageNum=1&pageSize=100", timeout=15).json()
                                suspects = [] 
                                
                                if res_feed.get("code") == "success":
                                    for item in res_feed.get("data", {}).get("shareFileList", []):
                                        owner = item.get("ownerAccount", "")
                                        if author_kw.lower() in owner.lower():
                                            uid = item.get("upUserId")
                                            if uid and (uid, owner) not in suspects:
                                                suspects.append((uid, owner))
                                else:
                                    raise Exception(f"接口掉线/风控拦截: {res_feed}")
                                
                                if not suspects:
                                    notifier.send_message(f"❌ 没找到名字里带【{author_kw}】的大佬。")
                                    continue
                                    
                                notifier.send_message(f"🔍 锁定 {len(suspects)} 个关联账号，正在合力挖掘最新动态...")
                                
                                all_blind_boxes = []
                                for uid, uname in suspects:
                                    for page in range(1, 7):
                                        url = f"https://cloud.189.cn/api/open/share/getUpResourceShare.action?pageNum={page}&pageSize=30&upUserId={uid}"
                                        res_user = client_obj.session.get(url, timeout=10).json()
                                        
                                        if res_user.get("code") == "success":
                                            items = res_user.get("data", {}).get("fileList", [])
                                            if not items: break 
                                            for itm in items:
                                                itm["_from_user"] = uname 
                                                all_blind_boxes.append(itm)
                                        else:
                                            raise Exception(f"扒主页时掉线/风控拦截: {res_user}")

                                if not all_blind_boxes:
                                    notifier.send_message("📭 选中的大佬们最近都没有发过任何东西。")
                                    continue

                                all_blind_boxes.sort(key=lambda x: x.get("lastOpTime", ""), reverse=True)
                                    
                                msg_lines = [f"🕵️‍♂️ <b>多账号联合情报（真实时间线）：</b>\n"]
                                kb_buttons = []
                                
                                for i, item in enumerate(all_blind_boxes[:20], 1):
                                    raw_name = item.get("name", "未知资源")
                                    name = translate_folder_name(raw_name)
                                    author = item.get("_from_user", "未知")
                                    url_code = item.get("accessURL", "")
                                    date_str = item.get("lastOpTime", "")[5:16]
                                    
                                    if not url_code: continue
                                    msg_lines.append(f"{i}. 📁 <code>{name}</code>\n   └ 👤 {author} | ⏱ {date_str}\n")
                                    btn_text = name.replace("📺 ", "").replace("🎬 ", "")
                                    kb_buttons.append([{"text": f"📥 订阅: {btn_text[:12]}...", "callback_data": f"wiz_feed_{url_code}"}])
                                    
                                kb_buttons.append([{"text": "❌ 取消", "callback_data": "wiz_cancel"}])
                                kb = {"inline_keyboard": kb_buttons}
                                
                                notifier.send_message("\n".join(msg_lines), kb)
                            except Exception as e:
                                notifier.send_message(f"❌ 联合查水表异常: {e}")
                                err_str = str(e).upper()
                                if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                    if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                    auto_relogin(client_obj, force=True)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
                            continue

                        elif text.startswith("全库重建") or text.startswith("/reall"):
                            notifier.send_message("🚨 收到全库重建指令！为防 Termux 内存爆炸，引擎已启动【切片下发模式】...")
                            try:
                                r_log = requests.post(f"{API_5244_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                if r_log.get("code") == 200:
                                    o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                    
                                    radar_bases = set()
                                    for l_cat, s_cat in CAT_ROUTER.values():
                                        sub_p = f"{l_cat}/{s_cat}".strip('/') if s_cat else l_cat
                                        radar_bases.add(get_openlist_path(f"{DIR_CAS_ROOT}/{sub_p}".replace("//", "/")))
                                    
                                    matched_paths = []
                                    for base_p in radar_bases:
                                        r_list = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": base_p}, headers=o_headers, timeout=5).json()
                                        if r_list.get("code") == 200:
                                            ym_dirs = [item["name"] for item in (r_list.get("data") or {}).get("content", []) if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                                            for ym in ym_dirs:
                                                ym_path = f"{base_p}/{ym}"
                                                r_shows = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": ym_path}, headers=o_headers, timeout=5).json()
                                                if r_shows.get("code") == 200:
                                                    for s_item in (r_shows.get("data") or {}).get("content", []):
                                                        if s_item["is_dir"]:
                                                            matched_paths.append(f"{ym_path}/{s_item['name']}")
                                    
                                    if matched_paths:
                                        notifier.send_message(f"🎯 扫描完毕！共锁定 {len(matched_paths)} 个独立剧集目录，开始逐一下发...")
                                        
                                        def rebuild_task():
                                            s = load_json(SETTINGS_FILE)
                                            strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
                                            sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
                                            real_type = sync_type_map.get(strm_mode_189, "cas")
                                            for i, mp in enumerate(matched_paths, 1):
                                                requests.post(f"{API_5244_URL}/api/fs/list", json={"path": mp, "refresh": True}, headers=o_headers, timeout=10).close()
                                                time.sleep(1.0)
                                                try: 
                                                    requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": mp, "type": real_type}, timeout=3).close()
                                                    logger.info(f"[{i}/{len(matched_paths)}] ✅ 成功下发: {mp}")
                                                except: pass
                                                time.sleep(2.0) 
                                            notifier.send_message(f"🎉 189 全库 STRM 重建完毕！")
                                        
                                        threading.Thread(target=rebuild_task).start()
                            except Exception as e:
                                notifier.send_message(f"❌ 全库重建异常: {e}")
                            continue

                        elif text == "列表" or text == "清单" or text == "/list":
                            subs = load_json(SUBS_FILE)
                            if not subs:
                                notifier.send_message("📭 当前没有任何活跃的监控任务。")
                                continue
                            msg_lines = ["📋 当前监控清单："]
                            for i, (sid, info) in enumerate(subs.items(), 1):
                                p = info.get("path", "")
                                freq = info.get("freq", "常规")
                                msg_lines.append(f"{i}. [{freq}] {p}")
                            msg_lines.append("\n💡 提示：回复“取消+序号”(如: 取消1) 即可解除监控并清理记忆。")
                            notifier.send_message("\n".join(msg_lines))

                        elif text.startswith("取消") or text.startswith("/cancel"):
                            kw = text.replace("取消", "").strip()
                            if not kw: continue
                            
                            subs = load_json(SUBS_FILE)
                            target_id = None
                            
                            if kw.isdigit():
                                idx = int(kw) - 1
                                if 0 <= idx < len(subs):
                                    target_id = list(subs.keys())[idx]
                            
                            if not target_id:
                                for sid, info in subs.items():
                                    p = info.get("path", "")
                                    if kw.lower() in p.lower():
                                        target_id = sid; break
                            
                            if target_id:
                                path_to_del = subs[target_id].get("path", "未知")
                                del subs[str(target_id)]
                                save_json(SUBS_FILE, subs)
                                
                                history_data = load_json(HISTORY_FILE)
                                old_len = len(history_data)
                                history_data = {k: v for k, v in history_data.items() if not (isinstance(v, dict) and str(v.get("sub_id")) == str(target_id))}
                                save_json(HISTORY_FILE, history_data)
                                
                                notifier.send_message(f"✅ 已解除订阅：{path_to_del}\n🗑️ 同步粉碎了 {old_len - len(history_data)} 条关联记忆。")
                            else:
                                notifier.send_message(f"❌ 没找到与“{kw}”相关的订阅任务。")

                        elif text == "体检":
                            notifier.send_message("🩺 正在为您执行全库深度体检与垃圾回收...")
                            subs = load_json(SUBS_FILE)
                            dead_count, dead_list = 0, []
                            for sid, info in list(subs.items()):
                                try:
                                    res_ch = client_obj.session.get("https://cloud.189.cn/api/open/file/listFiles.action", params={"folderId": sid, "pageNum": 1, "pageSize": 1}, timeout=10).json()
                                    if str(res_ch.get("res_code", "")) == "0":
                                        file_list_ao = res_ch.get("fileListAO", {})
                                        files = file_list_ao.get("fileList", [])
                                        folders = file_list_ao.get("folderList", [])
                                        if not files and not folders:
                                            dead_count += 1
                                            p = info.get("path", "未知路径") if isinstance(info, dict) else info
                                            dead_list.append(p)
                                            del subs[sid]
                                    else:
                                        logger.warning(f"⚠️ 目录 {sid} 接口异常，启动防误杀保护跳过！")
                                except Exception as e:
                                    logger.warning(f"⚠️ 目录 {sid} 检查失败跳过: {e}")
                                        
                            if dead_count > 0: save_json(SUBS_FILE, subs)
                            
                            history_data = load_json(HISTORY_FILE)
                            old_len = len(history_data)
                            history_data = {k: v for k, v in history_data.items() if not (isinstance(v, dict) and str(v.get("sub_id")) not in subs)}
                            save_json(HISTORY_FILE, history_data)
                            ghost_count = old_len - len(history_data)
                            
                            msg_str = ""
                            if dead_count > 0: msg_str += f"🚨 成功拔除 {dead_count} 个失效死目录：\n" + "\n".join(dead_list) + "\n\n"
                            else: msg_str += "✅ 您的所有订阅目录均健康在线。\n\n"
                                
                            if ghost_count > 0: msg_str += f"👻 执行垃圾回收：清除了 {ghost_count} 条残留历史记忆！"
                            else: msg_str += "✨ 历史记录库非常干净，无残留垃圾。"
                            notifier.send_message(f"🩺 体检报告：\n{msg_str}")

                        elif text.startswith("恢复云端") or text.startswith("/recloud"):
                            base_kw = text.replace("恢复云端", "").replace("/recloud", "").strip()
                            if not base_kw: 
                                notifier.send_message("❌ 请输入目标，例如：\n恢复云端 华语剧\n恢复云端 /177/177-秒传/电视剧")
                                continue
                            
                            notifier.send_message(f"🚑 收到恢复云端指令: [{base_kw}]\n🛡️ 【精准对账+30秒生成】已开启：逐个文件严格对比，少一集补一集！")
                            
                            try:
                                r_log = requests.post(f"{API_5244_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                if r_log.get("code") != 200:
                                    notifier.send_message("❌ OpenList 登录失败。")
                                    continue
                                o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                
                                scan_bases = []
                                if base_kw.startswith("/"):
                                    scan_bases.append(get_openlist_path(base_kw))
                                elif base_kw in CAT_ROUTER:
                                    l_cat, s_cat = CAT_ROUTER[base_kw]
                                    scan_bases.append(get_openlist_path(f"{DIR_CAS_ROOT}/{l_cat}/{s_cat}".strip('/').replace("//", "/")))
                                else:
                                    for l_cat, s_cat in CAT_ROUTER.values():
                                        sub_p = f"{l_cat}/{s_cat}".strip('/') if s_cat else l_cat
                                        if base_kw in sub_p:
                                            scan_bases.append(get_openlist_path(f"{DIR_CAS_ROOT}/{sub_p}".replace("//", "/")))
                                    if not scan_bases:
                                        scan_bases.append(get_openlist_path(DIR_CAS_ROOT))

                                scan_bases = list(set(scan_bases))

                                def do_segmented_recovery():
                                    s = load_json(SETTINGS_FILE)
                                    local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
                                    total_rebuild = 0
                                    
                                    def scan_for_shows(current_path):
                                        nonlocal total_rebuild
                                        try:
                                            res = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": current_path, "refresh": True}, headers=o_headers, timeout=10).json()
                                            if res.get("code") != 200: return
                                        except: return
                                        
                                        items = (res.get("data") or {}).get("content", [])
                                        
                                        has_cas = False
                                        for item in items:
                                            if not item["is_dir"] and item["name"].lower().endswith(".cas"):
                                                has_cas = True
                                                break
                                                
                                        if has_cas:
                                            strm_sub_dir = current_path.split(DIR_CAS_ROOT)[-1].strip("/")
                                            target_local_dir = os.path.join(local_strm_dir, strm_sub_dir).replace("\\", "/")
                                            
                                            need_rebuild = False
                                            
                                            if not os.path.exists(target_local_dir):
                                                need_rebuild = True
                                            else:
                                                import re
                                                local_strms = [f.lower() for f in os.listdir(target_local_dir) if f.endswith(".strm")]
                                                
                                                for item in items:
                                                    if not item["is_dir"] and item["name"].lower().endswith(".cas"):
                                                        cloud_raw = item["name"][:-4].lower()
                                                        cloud_keys = set(re.findall(r'[a-z0-9\u4e00-\u9fa5]+', cloud_raw))
                                                        
                                                        is_matched = False
                                                        for loc in local_strms:
                                                            loc_keys = set(re.findall(r'[a-z0-9\u4e00-\u9fa5]+', loc[:-5]))
                                                            if loc_keys and loc_keys.issubset(cloud_keys):
                                                                is_matched = True
                                                                break
                                                        
                                                        if not is_matched:
                                                            need_rebuild = True
                                                            break 
                                            
                                            if need_rebuild:
                                                logger.warning(f"🚑 发现真实缺漏文件，立即交由管家补齐: {current_path}")
                                                try:
                                                    s = load_json(SETTINGS_FILE)
                                                    strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
                                                    sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
                                                    real_type = sync_type_map.get(strm_mode_189, "cas")
                                                    requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": current_path, "type": real_type}, timeout=5)
                                                except Exception as e:
                                                    pass
                                                total_rebuild += 1
                                                
                                                logger.info(f"⏳ 缓冲防爆：等待管家落盘，休眠 15 秒...")
                                                time.sleep(15.0)
                                                
                                            return 
                                            
                                        for item in items:
                                            if item["is_dir"]:
                                                scan_for_shows(f"{current_path}/{item['name']}")

                                    # 💥 关键修复：把调用循环正确缩进到后台线程函数的内部！
                                    for base_p in scan_bases:
                                        notifier.send_message(f"🔎 启动极限穿透扫描: {base_p}")
                                        scan_for_shows(base_p)
                                        
                                    notifier.send_message(f"🎉 扫描结束！本次总计精准抢救并补全了 {total_rebuild} 部存在缺漏的剧集！")

                                threading.Thread(target=do_segmented_recovery).start()
                            except Exception as e:
                                notifier.send_message(f"❌ 恢复指令异常: {e}")
                            continue
                            
                        elif text.startswith("刷新") or text.startswith("入库"):
                            match_refresh = re.match(r'^(刷新|入库)\s+(.*)', text)
                            if match_refresh:
                                keyword_input = match_refresh.group(2).strip()
                                m = re.match(r'^(.*?)\s+(?:[sS第]\s*)?0?(\d{1,2})[季]?$', keyword_input, re.IGNORECASE)
                                if m and m.group(1).strip(): 
                                    base_kw, s_num = m.group(1).strip(), int(m.group(2))
                                else: 
                                    base_kw, s_num = keyword_input, None

                                notifier.send_message(f"🔍 收到入库指令，正在检索: {base_kw}...")
                                
                                subs = load_json(SUBS_FILE)
                                matched_paths = []
                                
                                if base_kw.startswith("/"):
                                    matched_paths.append(base_kw.strip())
                                else:
                                    for t_id, info in subs.items():
                                        path_in_db = info.get("path", "") if isinstance(info, dict) else ""
                                        if base_kw.lower() in path_in_db.lower():
                                            if path_in_db not in matched_paths: matched_paths.append(path_in_db)

                                    if not matched_paths:
                                        notifier.send_message(f"🧠 记忆库未收录老剧，正启动 Openlist 穿透雷达自动检索真实路径: [{base_kw}] ...")
                                        try:
                                            r_log = requests.post(f"{API_5244_URL}/api/auth/login", 
                                                                  json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                            if r_log.get("code") == 200:
                                                o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                                
                                                radar_bases = set()
                                                for l_cat, s_cat in CAT_ROUTER.values():
                                                    sub_p = f"{l_cat}/{s_cat}".strip('/') if s_cat else l_cat
                                                    radar_bases.add(get_openlist_path(f"{DIR_CAS_ROOT}/{sub_p}".replace("//", "/")))
                                                    radar_bases.add(get_openlist_path(f"{DIR_VIDEO_ROOT}/{sub_p}".replace("//", "/")))
                                                
                                                for base_p in radar_bases:
                                                    r_list = requests.post(f"{API_5244_URL}/api/fs/list", 
                                                                           json={"path": base_p}, headers=o_headers, timeout=5).json()
                                                    if r_list.get("code") == 200:
                                                        content = (r_list.get("data") or {}).get("content") or []
                                                        ym_dirs = [item["name"] for item in content if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                                                        ym_dirs.sort(reverse=True) 
                                                        
                                                        for ym in ym_dirs:
                                                            ym_path = f"{base_p}/{ym}"
                                                            r_shows = requests.post(f"{API_5244_URL}/api/fs/list", 
                                                                                    json={"path": ym_path}, headers=o_headers, timeout=5).json()
                                                            if r_shows.get("code") == 200:
                                                                shows = (r_shows.get("data") or {}).get("content") or []
                                                                for s_item in shows:
                                                                    if s_item["is_dir"] and base_kw.lower() in s_item["name"].lower():
                                                                        exact_path = f"{ym_path}/{s_item['name']}"
                                                                        matched_paths.append(exact_path)
                                                                        break
                                                            if matched_paths: break
                                                    if matched_paths: break
                                        except Exception as radar_err:
                                            logger.warning(f"Openlist自动寻轨雷达异常: {radar_err}")
                                        
                                        if matched_paths:
                                            notifier.send_message(f"🎯 雷达寻轨成功！全自动还原出带年份的物理绝对路径:\n📁 {matched_paths[0]}")
                                        else:
                                            notifier.send_message(f"📭 雷达遍历了全部分区，未找到包含【{base_kw}】的实体文件夹，已跳过。")
                                            
                                if matched_paths:
                                    notifier.send_message(f"🎯 共命中 {len(matched_paths)} 个关联目录，执行双轨刷新...")
                                    
                                    o_headers = None
                                    try:
                                        r_log = requests.post(f"{API_5244_URL}/api/auth/login", 
                                                              json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                        if r_log.get("code") == 200:
                                            o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                    except: pass

                                    for mp in matched_paths:
                                        openlist_p = get_openlist_path(mp)
                                        butler_path = re.sub(r'(?i)/Season\s*\d+/?$', '', openlist_p)
                                        
                                        if o_headers:
                                            try:
                                                requests.post(f"{API_5244_URL}/api/fs/list", 
                                                              json={"path": butler_path, "refresh": True}, headers=o_headers, timeout=15).close()
                                                time.sleep(3.0) 
                                            except: pass

                                        if DIR_CAS_ROOT in openlist_p:
                                            try: 
                                                s = load_json(SETTINGS_FILE)
                                                strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
                                                sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
                                                real_type = sync_type_map.get(strm_mode_189, "cas")
                                                
                                                requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": butler_path, "type": real_type}, timeout=3).close()
                                                notifier.send_message(f"✅ 管家同步指令已精准下发: {butler_path}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            try:
                                                s = load_json(SETTINGS_FILE)
                                                bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                                refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                                subprocess.Popen([bash_path, refresh_sh, openlist_p], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                                notifier.send_message(f"✅ 缓存已刷，原生Emby拉取成功: {openlist_p}")
                                            except: pass
                                        
                                    notifier.send_message("🎉 批量单点指令已全部双轨执行完毕！")

                                else:
                                    notifier.send_message("⚠️ 启动大范围全局雷达扫描，正在过滤顶级父目录防止STRM外溢...")
                                    
                                    scan_roots = set()
                                    for tid, info in subs.items():
                                        p = info.get("path", "") if isinstance(info, dict) else ""
                                        if p:
                                            clean_p = get_openlist_path(p)
                                            safe_butler_path = re.sub(r'(?i)/Season\s*\d+/?$', '', clean_p)
                                            scan_roots.add(safe_butler_path)

                                    notifier.send_message(f"⏳ 精确锁定 {len(scan_roots)} 个安全剧集锚点，开始双轨下发...")
                                    
                                    o_headers = None
                                    try:
                                        r_log = requests.post(f"{API_5244_URL}/api/auth/login", 
                                                              json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                        if r_log.get("code") == 200: 
                                            o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                    except: pass
                                    
                                    for rp in scan_roots:
                                        if o_headers:
                                            try:
                                                requests.post(f"{API_5244_URL}/api/fs/list", 
                                                              json={"path": rp, "refresh": True}, headers=o_headers, timeout=10).close()
                                                time.sleep(1.5) 
                                            except: pass

                                        if DIR_CAS_ROOT in rp:
                                            try: 
                                                s = load_json(SETTINGS_FILE)
                                                strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
                                                sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
                                                real_type = sync_type_map.get(strm_mode_189, "cas")
                                                requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": rp, "type": real_type}, timeout=3).close()
                                            except: pass
                                        else:
                                            try:
                                                s = load_json(SETTINGS_FILE)
                                                bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                                refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                                subprocess.Popen([bash_path, refresh_sh, rp], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                            except: pass
                                        
                                    notifier.send_message(f"✅ 全区最高指令已下发！安全触发并双轨刷新了 {len(scan_roots)} 个雷达基点。")

                        elif text.startswith("补档"):
                            match_fill = re.match(r'^补档\s+(.*?)\s+(http[s]?://\S+)', text)
                            if match_fill:
                                keyword_input = match_fill.group(1).strip()
                                share_url = match_fill.group(2).strip()
                                m = re.match(r'^(.*?)\s*[sS第]?0?(\d+)[季]?$', keyword_input)
                                if m and m.group(1).strip(): base_kw, s_num = m.group(1).strip(), int(m.group(2))
                                else: base_kw, s_num = keyword_input, None

                                notifier.send_message(f"🔍 启动补档...\n🎯 解析剧名: {base_kw}\n🔗 链接: {share_url}")

                                subs = load_json(SUBS_FILE)
                                matched_target = None
                                for t_id, info in subs.items():
                                    path_in_db = info.get("path", "") if isinstance(info, dict) else ""
                                    kw_in_db = info.get("keyword", "") if isinstance(info, dict) else ""
                                    if base_kw.lower() in path_in_db.lower():
                                        if s_num is not None:
                                            s_patterns = [f"season {s_num}", f"s{s_num:02d}", f"s{s_num}"]
                                            if any(p in path_in_db.lower() for p in s_patterns) or str(s_num) in path_in_db.split('/')[-1]:
                                                matched_target = (t_id, path_in_db, kw_in_db); break
                                        else:
                                            matched_target = (t_id, path_in_db, kw_in_db); break

                                if not matched_target: continue

                                target_id, target_path, target_keyword = matched_target
                                notifier.send_message(f"🎯 命中目录: {target_path}\n🚀 核对云端文件中...")
                                
                                try: info_s = client_obj.getShareInfo(share_url)
                                except Exception as e:
                                    notifier.send_message(f"❌ 补档失效 ({e})")
                                    continue

                                try:
                                    all_files = get_all_share_files_recursive(info_s)
                                    if target_keyword: all_files = [f for f in all_files if all(k in f["full_path"].lower() for k in target_keyword.lower().split())]

                                    cloud_files = client_obj.listPrivateFiles(target_id)
                                    cloud_file_names = {cf["name"] for cf in cloud_files}

                                    new_files = []
                                    for f in all_files:
                                        expected_smart_name = generate_smart_name(f["name"], target_path)
                                        if expected_smart_name not in cloud_file_names and f["name"] not in cloud_file_names:
                                            new_files.append(f)

                                    openlist_target_path = get_openlist_path(target_path)

                                    if not new_files:
                                        notifier.send_message("⚠️ 核对完毕：无需重复拉取。")
                                        if target_path.startswith(DIR_CAS_ROOT) or target_path.startswith(DIR_CAS_ROOT.strip('/')):
                                            try: 
                                                s = load_json(SETTINGS_FILE)
                                                strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
                                                sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
                                                real_type = sync_type_map.get(strm_mode_189, "cas")
                                                requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": openlist_target_path, "type": real_type}, timeout=3).close()
                                                notifier.send_message(f"✅ 管家同步指令已下发: {openlist_target_path}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            try: 
                                                s = load_json(SETTINGS_FILE)
                                                bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                                refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                                subprocess.Popen([bash_path, refresh_sh, openlist_target_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                                notifier.send_message(f"✅ Emby刷新指令已下发: {openlist_target_path}")
                                            except: pass
                                    else:
                                        taskInfos = [{"fileId": f["id"], "fileName": clean_filename(f["name"]), "isFolder": 0} for f in new_files]
                                        
                                        code = info_s.saveShareFiles(taskInfos, target_id)
                                        
                                        if code in [0, '0', None, False, '']:
                                            time.sleep(8)
                                            fresh_cloud_files = client_obj.listPrivateFiles(target_id)
                                            fresh_names = [cf["name"] for cf in fresh_cloud_files]
                                            
                                            history_data = load_json(HISTORY_FILE)
                                            actually_saved_count = 0
                                            for task in taskInfos:
                                                orig_name = task["fileName"]
                                                expected_smart_name = generate_smart_name(orig_name, target_path)
                                                if orig_name in fresh_names or (expected_smart_name and expected_smart_name in fresh_names):
                                                    history_data[str(task["fileId"])] = {"name": orig_name, "sub_id": str(target_id)}
                                                    actually_saved_count += 1
                                            
                                            if actually_saved_count > 0:
                                                renamed_files_list = []
                                                for task in taskInfos:
                                                    original_name = task["fileName"]
                                                    new_name = generate_smart_name(original_name, target_path)
                                                    if new_name != original_name:
                                                        for cf in fresh_cloud_files:
                                                            if cf["name"] == original_name:
                                                                if client_obj.renameFile(cf["id"], new_name): 
                                                                    renamed_files_list.append(new_name)
                                                                break

                                                save_json(HISTORY_FILE, history_data)
                                                notifier.send_message(f"✅ 补档完美结束！\n📂 成功抓取 {actually_saved_count} 个缺失文件。")
                                                if renamed_files_list:
                                                    if len(renamed_files_list) > 20:
                                                        r_msg = "\n".join([f" └ {n}" for n in renamed_files_list[:20]]) + f"\n...等共 {len(renamed_files_list)} 个文件"
                                                    else:
                                                        r_msg = "\n".join([f" └ {n}" for n in renamed_files_list])
                                                    notifier.send_message(f"✨ 补档云端洗名完成:\n{r_msg}")
                                                time.sleep(6)
                                                
                                                if target_path.startswith(DIR_CAS_ROOT) or target_path.startswith(DIR_CAS_ROOT.strip('/')):
                                                    try:
                                                        s = load_json(SETTINGS_FILE)
                                                        strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
                                                        sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
                                                        real_type = sync_type_map.get(strm_mode_189, "cas")
                                                        requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": openlist_target_path, "type": real_type}, timeout=3).close()
                                                        notifier.send_message(f"✅ 管家同步指令已下发: {openlist_target_path}")
                                                    except Exception as e: 
                                                        notifier.send_message(f"❌ 管家同步无响应: {e}")
                                                else:
                                                    try:
                                                        s = load_json(SETTINGS_FILE)
                                                        bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                                        refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                                        subprocess.Popen([bash_path, refresh_sh, openlist_target_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                                        notifier.send_message(f"✅ Emby刷新指令已下发: {openlist_target_path}")
                                                    except: pass
                                        else:
                                            notifier.send_message(f"❌ 天翼云拒绝转存: {code}")
                                except Exception as e:
                                    err_str = str(e).upper()
                                    if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                        notifier.send_message(f"⚠️ 检测到 IP 漂移！正在自愈...")
                                        if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                        auto_relogin(client_obj, force=True)
                                        notifier.send_message("✅ IP 漂移已修复！请重发指令。")
                                    else:
                                        notifier.send_message(f"❌ 补档异常: {e}")

                        elif text.startswith("订阅") or text.startswith("绑定") or text.startswith("/sub "):
                            match_bind = re.match(r'^(订阅|绑定)(\d)?\s+', text)
                            if match_bind:
                                action = match_bind.group(1)
                                season_num = match_bind.group(2)
                                
                                freq_tag = ""
                                if "#周更" in text: freq_tag, text = "周更", text.replace("#周更", "").strip()
                                elif "#双更" in text: freq_tag, text = "双更", text.replace("#双更", "").strip()
                                elif "#剧迷" in text: freq_tag, text = "剧迷", text.replace("#剧迷", "").strip()
                                elif "#日更" in text: freq_tag, text = "日更", text.replace("#日更", "").strip()
                                elif "#完结" in text: freq_tag, text = "完结", text.replace("#完结", "").strip()
                                elif "#单次" in text: freq_tag, text = "单次", text.replace("#单次", "").strip()
                                elif "#电影" in text: freq_tag, text = "单次", text.replace("#电影", "").strip() 
                                
                                explicit_weekday = False
                                weekday_map = {"周一": 0, "周二": 1, "周三": 2, "周四": 3, "周五": 4, "周六": 5, "周日": 6}
                                t_weekday = 5
                                for d_name, d_code in weekday_map.items():
                                    if f"#{d_name}" in text:
                                        t_weekday = d_code
                                        explicit_weekday = True
                                        text = text.replace(f"#{d_name}", "").strip()
                                        break
                                
                                if explicit_weekday and not freq_tag:
                                    freq_tag = "周更"
                                        
                                custom_tags = []
                                for tag in CAT_ROUTER.keys():
                                    if f"#{tag}" in text:
                                        custom_tags.append(tag)
                                        text = text.replace(f"#{tag}", "").strip()
                                        
                                cat_map = {
                                    "电影": ["电影", "movie", "大电影"],
                                    "动漫": ["动漫", "动画", "新番", "番剧", "anime"],
                                    "综艺": ["综艺", "真人秀", "晚会"],
                                    "演唱会": ["演唱会", "live", "音乐会", "巡演"],
                                    "纪录片": ["纪录片", "documentary", "探索"],
                                    "短剧": ["短剧", "微短剧", "爽剧"]
                                }
                                explicit_cat = ""
                                for cat, aliases in cat_map.items():
                                    if f"#{cat}" in text:
                                        explicit_cat = cat
                                        text = text.replace(f"#{cat}", "").strip()
                                    for alias in aliases:
                                        if f"#{alias}" in text:
                                            if not explicit_cat: explicit_cat = cat
                                            text = text.replace(f"#{alias}", "").strip()
                                        
                                is_bind = (action == "绑定")
                                
                                pwd_match = re.search(r'(?:访问码|提取码|密码)\s*[:：]\s*([a-zA-Z0-9]{4})', text)
                                extracted_pwd = pwd_match.group(1) if pwd_match else None
                                text = re.sub(r'[\(（]?\s*(?:访问码|提取码|密码)\s*[:：]\s*[a-zA-Z0-9]{4}\s*[\)）]?', '', text)
                                
                                parts = text.split()
                                share_url, keyword, target_path = "", "", ""

                                url_index = -1
                                for i, p in enumerate(parts):
                                    if p.startswith("http"):
                                        share_url, url_index = p, i
                                        break

                                if url_index != -1:
                                    if extracted_pwd:
                                        share_url += f"?pwd={extracted_pwd}" if "?" not in share_url else f"&pwd={extracted_pwd}"
                                        
                                    target_path = " ".join(parts[1:url_index])
                                    if url_index < len(parts) - 1: keyword = " ".join(parts[url_index+1:])
                                else:
                                    target_path = " ".join(parts[1:])

                                if season_num:
                                    s_num = int(season_num)
                                    if "season" not in target_path.lower(): target_path = f"{target_path.rstrip('/')}/Season {s_num}"
                                    if not re.search(r'(?i)S\d+', keyword): keyword = f"S{s_num:02d} {keyword}".strip()
                                    else: keyword = keyword.strip()

                                if not share_url:
                                    subs = load_json(SUBS_FILE)
                                    for tid, info_dict in subs.items():
                                        if isinstance(info_dict, dict) and info_dict.get("path") == target_path:
                                            share_url = info_dict.get("url", ""); break
                                            
                                    if not share_url: continue

                                is_absolute_path = target_path.startswith('/') or DIR_MEDIA_PREFIX in target_path or target_path.startswith(DIR_CAS_ROOT)
                                if not is_absolute_path:
                                    if not target_path:
                                        notifier.send_message("❌ 缺少【干净剧名】！\n为防乱套，请不要发纯链接。\n格式：订阅 剧集名称(年份) 链接")
                                        continue
                                        
                                    notifier.send_message("🔍 启动智能路由，探测通道与品类...")
                                    try:
                                        info_s = client_obj.getShareInfo(share_url)
                                        raw_name = info_s.file_name
                                        preview_files = get_all_share_files_recursive(info_s)
                                        is_cas = any(f['name'].lower().endswith('.cas') for f in preview_files)
                                        
                                        base_dir_large = "电视剧"
                                        base_dir_sub = "0-电视剧"
                                        
                                        if custom_tags:
                                            base_dir_large, base_dir_sub = CAT_ROUTER[custom_tags[0]]
                                        elif explicit_cat:
                                            base_dir_large = explicit_cat
                                            if explicit_cat == "电影": base_dir_sub = "0-电影"
                                            elif explicit_cat == "动漫": base_dir_sub = "0-动漫"
                                            else: base_dir_sub = ""
                                        else:
                                            combined_input = f"{target_path} {keyword} {freq_tag} {text}".lower()
                                            found_cat = False
                                            for cat in cat_map.keys():
                                                if f"#{cat}" in combined_input or freq_tag == cat:
                                                    base_dir_large = cat
                                                    if cat == "电影": base_dir_sub = "0-电影"
                                                    elif cat == "动漫": base_dir_sub = "0-动漫"
                                                    else: base_dir_sub = ""
                                                    found_cat = True; break
                                            
                                            if not found_cat:
                                                for cat, kws in cat_map.items():
                                                    if any(kw in raw_name.lower() for kw in kws):
                                                        base_dir_large = cat
                                                        if cat == "电影": base_dir_sub = "0-电影"
                                                        elif cat == "动漫": base_dir_sub = "0-动漫"
                                                        else: base_dir_sub = ""
                                                        break
                                        
                                        clean_user_path = target_path.strip()
                                        current_ym = datetime.now().strftime("%Y%m")
                                        
                                        subs_cache = load_json(SUBS_FILE)
                                        existing_path = None
                                        
                                        def get_pure(text): return text.replace(" ", "").lower()
                                        show_name = clean_user_path.split('/')[0].strip()
                                        pure_show = get_pure(show_name)
                                        
                                        best_match_path = None
                                        
                                        ignore_words = {get_pure(DIR_CAS_ROOT), get_pure(DIR_VIDEO_ROOT), "season", "s"}
                                        for cat_key, (large_cat, sub_cat) in CAT_ROUTER.items():
                                            ignore_words.add(get_pure(cat_key))
                                            ignore_words.add(get_pure(large_cat))
                                            if sub_cat: ignore_words.add(get_pure(sub_cat))
                                        
                                        for sid, info_dict in subs_cache.items():
                                            if isinstance(info_dict, dict):
                                                db_path = info_dict.get("path", "")
                                                db_is_cas = DIR_CAS_ROOT in db_path
                                                if is_cas != db_is_cas: continue
                                                
                                                db_folders = db_path.split('/')
                                                for idx, f_name in enumerate(db_folders):
                                                    pure_f = get_pure(f_name)
                                                    if not pure_f or len(pure_f) < 2: continue
                                                    if re.match(r'^\d{4,6}$', pure_f) or "season" in pure_f or re.match(r'^s\d+$', pure_f): continue
                                                    if pure_f in ignore_words: continue
                                                    
                                                    if pure_show == pure_f:
                                                        root_path = "/".join(db_folders[:idx+1])
                                                        best_match_path = root_path + clean_user_path[len(show_name):] if "/" in clean_user_path else root_path
                                                        break
                                                if best_match_path: break
                                        
                                        if best_match_path:
                                            existing_path = best_match_path
                                            type_msg = f"记忆库精确匹配沿用旧目录"
                                            
                                        if not existing_path:
                                            notifier.send_message("🧠 记忆库未找到记录，启动网盘物理层穿甲扫描，探测历史遗留目录...")
                                            try:
                                                root_for_search = DIR_CAS_ROOT if is_cas else DIR_VIDEO_ROOT
                                                base_search_path = f"{root_for_search}/{base_dir_large}/{base_dir_sub}".strip('/').replace("//", "/") if base_dir_sub else f"{root_for_search}/{base_dir_large}".strip('/')
                                                
                                                curr_id = -11
                                                valid_path = True
                                                for p in base_search_path.split('/'):
                                                    if not p: continue
                                                    nodes = client_obj.getObjectFolderNodes(curr_id)
                                                    matched = next((n for n in nodes if n["name"] == p), None)
                                                    if matched:
                                                        curr_id = matched["id"]
                                                    else:
                                                        valid_path = False; break
                                                
                                                if valid_path:
                                                    ym_nodes = client_obj.getObjectFolderNodes(curr_id)
                                                    ym_nodes.sort(key=lambda x: x["name"], reverse=True)
                                                    
                                                    phy_best_path = None
                                                    
                                                    for ym_node in ym_nodes:
                                                        if re.match(r'^\d{4,6}$', ym_node["name"]):
                                                            show_nodes = client_obj.getObjectFolderNodes(ym_node["id"])
                                                            for show_node in show_nodes:
                                                                pure_f = get_pure(show_node["name"])
                                                                if pure_show == pure_f:
                                                                    found_physical_path = f"/{base_search_path}/{ym_node['name']}/{show_node['name']}"
                                                                    phy_best_path = found_physical_path + clean_user_path[len(show_name):] if "/" in clean_user_path else found_physical_path
                                                                    break
                                                        if phy_best_path: break
                                                
                                                if phy_best_path:
                                                    existing_path = phy_best_path
                                                    type_msg = f"网盘实体精确寻回"
                                            except Exception as e:
                                                logger.warning(f"⚠️ 物理雷达扫描异常 (防风控跳过): {e}")
                                                
                                        if existing_path:
                                            target_path = existing_path
                                        else:
                                            if is_cas:
                                                target_path = f"{DIR_CAS_ROOT}/{base_dir_large}/{base_dir_sub}/{current_ym}/{clean_user_path}".replace("//", "/") if base_dir_sub else f"{DIR_CAS_ROOT}/{base_dir_large}/{current_ym}/{clean_user_path}".replace("//", "/")
                                                type_msg = "全新CAS秒传(建新基建)"
                                            else:
                                                target_path = f"{DIR_VIDEO_ROOT}/{base_dir_large}/{base_dir_sub}/{current_ym}/{clean_user_path}".replace("//", "/") if base_dir_sub else f"{DIR_VIDEO_ROOT}/{base_dir_large}/{current_ym}/{clean_user_path}".replace("//", "/")
                                                type_msg = "全新智能直连(建新基建)"
                                        
                                        if base_dir_large == "电影" and not freq_tag:
                                            freq_tag = "单次"
                                            
                                        notifier.send_message(f"🧠 路由组装完毕 [{type_msg}]！\n📂 建档至: {target_path}")
                                    except Exception as e:
                                        err_str = str(e).upper()
                                        if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                            notifier.send_message(f"⚠️ 探测到 IP 漂移！正在自愈...")
                                            if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                            auto_relogin(client_obj, force=True)
                                            notifier.send_message("✅ 自愈完成，请重发指令。")
                                        else:
                                            notifier.send_message(f"❌ 智能解析失败 ({e})")
                                        continue

                                tag_msg = f" ⏱️ 频率: {freq_tag}" if freq_tag else ""
                                kw_msg = f" 🎯 过滤: {keyword}" if keyword else ""
                                notifier.send_message(f"⏳ 正在处理{action}：\n📁 {target_path}{tag_msg}{kw_msg} ...")
                                
                                try: info_s = client_obj.getShareInfo(share_url)
                                except Exception as e:
                                    notifier.send_message(f"❌ {action}失败：{e}")
                                    continue
                                    
                                try:
                                    target_id = client_obj.mkdirAll(target_path)
                                    subs = load_json(SUBS_FILE)
                                    subs[str(target_id)] = {"url": share_url, "keyword": keyword, "path": target_path, "last_update": 0, "freq": freq_tag, "update_weekday": t_weekday, "next_check_time": 0}
                                    save_json(SUBS_FILE, subs)
                                    
                                    if is_bind:
                                        all_files = get_all_share_files_recursive(info_s)
                                        if keyword: all_files = [f for f in all_files if all(k in f["full_path"].lower() for k in keyword.lower().split())]
                                        history_data = load_json(HISTORY_FILE)
                                        for f in all_files: history_data[str(f["id"])] = {"name": f["name"], "sub_id": str(target_id)}
                                        save_json(HISTORY_FILE, history_data)
                                        notifier.send_message(f"✅ 成功绑定！\n❇️ 标记了 {len(all_files)} 个旧文件。")
                                    else:
                                        notifier.send_message(f"✅ 添加成功！优先拉取资源...")
                                        check_subscriptions(client_obj, force_target_id=target_id) 
                                except Exception as e:
                                    err_str = str(e).upper()
                                    if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:        
                                        notifier.send_message(f"⚠️ 建档探测到 IP 漂移！正在自愈...")
                                        if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                        auto_relogin(client_obj, force=True)
                                        notifier.send_message("✅ 自愈完成，请重发指令。")
                                    else:
                                        notifier.send_message(f"❌ 云端拦截: {e}")
        except Exception: pass 
        time.sleep(2)


trigger_app = Flask(__name__)
log = logging.getLogger('werkzeug')
log.setLevel(logging.ERROR)


# ==========================================
# 📦 139 智能全域扫荡雷达
# ==========================================
def scan_139_library(force=False):
    s = load_json(SETTINGS_FILE)
    strm_mode_139 = s.get("139_strm_mode", "mode_a_139")
    target_dirs = set()
    
    if strm_mode_139 in ["mode_b_139", "both_139"]:
        try:
            r_log = requests.post(f"{API_5255_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
            if r_log.get("code") == 200:
                h_139 = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                r_cats = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": DIR_139_TARGET}, headers=h_139, timeout=5).json()
                cats = [c["name"] for c in (r_cats.get("data") or {}).get("content", []) if c["is_dir"]]
                for cat in cats:
                    cat_path = f"{DIR_139_TARGET}/{cat}"
                    r_shows = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": cat_path}, headers=h_139, timeout=5).json()
                    shows = [sh["name"] for sh in (r_shows.get("data") or {}).get("content", []) if sh["is_dir"]]
                    for show in shows:
                        show_path = f"{cat_path}/{show}"
                        r_seasons = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": show_path}, headers=h_139, timeout=5).json()
                        seasons = [ss["name"] for ss in (r_seasons.get("data") or {}).get("content", []) if ss["is_dir"] and "season" in ss["name"].lower()]
                        if seasons:
                            for s_name in seasons: target_dirs.add(f"{show_path}/{s_name}")
                        else:
                            target_dirs.add(show_path)
        except: pass
    else:
        if os.path.exists(DIR_LOCAL_CAS):
            for root, dirs, files in os.walk(DIR_LOCAL_CAS):
                if any(f.lower().endswith(('.cas', '.iso')) for f in files):
                    target_dirs.add(root.replace("\\", "/"))
                    
    if not target_dirs: return 0
        
    refresh_dirs = set()
    for t_dir in target_dirs:
        try:
            # 💥 核心修正：全域扫描坚决不传 is_webhook=True，启用极速秒扫！
            t_strm_dir = sync_139_to_smart_strm(t_dir, force_rebuild=force, is_webhook=False)
            if t_strm_dir: refresh_dirs.add(t_strm_dir)
        except: pass
            
    if refresh_dirs:
        logger.info(f"✨ [139全域扫荡] 发现增量变动，共更新 {len(refresh_dirs)} 个目录，准备刷新 Emby...")
        bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
        refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
        for d in refresh_dirs:
            try:
                subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                time.sleep(0.5)
            except: pass
                
    return len(refresh_dirs)
    
# ==========================================
# 🟢 189 外部下载脚本入口 (带参数翻译恢复)
# ==========================================
@trigger_app.route('/api/trigger', methods=['GET', 'POST'])
def trigger():
    target_dir = request.args.get('path')
    if not target_dir and request.is_json: 
        target_dir = request.json.get('path')
        
    if target_dir:
        def handle_external():
            logger.info(f"⚡ [外部189投递] 收到闪电信号: {target_dir}")
            
            s = load_json(SETTINGS_FILE)
            strm_mode_189 = s.get("189_strm_mode", "mode_a_189")
            
            sync_type_map = {"mode_a_189": "cas", "mode_b_189": "cas_native", "both_189": "both"}
            if strm_mode_189 in ["cas", "cas_native", "both"]: real_type = strm_mode_189
            else: real_type = sync_type_map.get(strm_mode_189, "cas")
            
            try: 
                requests.get(f"{API_5000_URL}/api/sync", params={"drive": "189", "path": target_dir, "type": real_type}, timeout=15).close()
            except Exception as e: 
                logger.error(f"❌ [外部189投递] 呼叫 5000 异常: {e}")
            
            try:
                time.sleep(15.0)
                strm_sub_dir = target_dir.replace(DEFAULT_LOCAL_DROPBOX, "").strip("/") 
                real_strm_path = os.path.join(s.get("local_strm_dir", DEFAULT_LOCAL_STRM), strm_sub_dir).replace("\\", "/")
                subprocess.Popen([s.get("bash_path", DEFAULT_BASH_PATH), s.get("refresh_sh_path", DEFAULT_REFRESH_SH), real_strm_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                logger.info(f"⚡ [外部189投递] 已精准下发 Emby 刷新指令: {real_strm_path}")
            except: pass
                
        threading.Thread(target=handle_external).start()
        return "✅ 189 外部投递信号接管成功", 200
        
    return "❌ 缺少 path 参数", 400

# ==========================================
# 🌟 139 专属智能翻译引擎 (189 级雷达复刻版)
# ==========================================
def sync_139_to_smart_strm(target_dir, force_rebuild=False, is_webhook=False):
    try:
        s = load_json(SETTINGS_FILE)
        strm_mode_139 = s.get("139_strm_mode", "mode_a_139")
        local_139_strm_dir = s.get("local_139_strm_dir", DEFAULT_139_LOCAL_STRM)
        
        db_path = os.path.join(os.path.dirname(SETTINGS_FILE), "139_monitor_history.db")
        conn = sqlite3.connect(db_path)
        cursor = conn.cursor()
        cursor.execute('''CREATE TABLE IF NOT EXISTS history (file_path TEXT PRIMARY KEY, mtime REAL, file_size INTEGER)''')
        
        if DIR_139_TARGET in target_dir:
            cloud_dir = target_dir
            local_dir = target_dir.replace(DIR_139_TARGET, DIR_LOCAL_CAS)
        elif DIR_LOCAL_CAS in target_dir:
            local_dir = target_dir
            cloud_dir = target_dir.replace(DIR_LOCAL_CAS, DIR_139_TARGET)
        else:
            conn.close()
            return None
            
        rel_path = cloud_dir.replace(DIR_139_TARGET, "").strip("/")
        parts = rel_path.split("/")
        category_key = parts[0] if len(parts) > 0 else None
        
        # 💥 致命 Bug 修复点：移除了 parts[:-1]，直接从最底层目录往上推
        # 完美对齐 189 逻辑，剥离 Season 后，精准命中真正的剧名/电影名
        local_season_num = 1
        show_folder_name = "未命名"
        
        for part in reversed(parts):
            s_match_dir = re.match(r'(?i)^(?:season\s*|s)(\d+)$', part.strip())
            if s_match_dir:
                local_season_num = int(s_match_dir.group(1))
                continue
            show_folder_name = part.strip()
            break
                
        current_ym = datetime.now().strftime("%Y%m")
        if os.path.exists(local_139_strm_dir):
            for root, dirs, files in os.walk(local_139_strm_dir):
                if show_folder_name in dirs:
                    ym_match = re.search(r'/(\d{6})$', root.replace('\\', '/'))
                    if ym_match: current_ym = ym_match.group(1); break
                        
        b_large, b_sub = "未分类", "0-未分类"
        if category_key and category_key in CAT_ROUTER:
            b_large, b_sub = CAT_ROUTER[category_key]
        else:
            for cat_k, (l, sub) in CAT_ROUTER.items():
                if cat_k in rel_path: b_large, b_sub = l, sub; break
            if b_large == "未分类": b_large, b_sub = "电视剧", "0-电视剧"
            
        virtual_cloud_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{current_ym}/{show_folder_name}".replace("//", "/")
        if b_large in ["电视剧", "动漫", "短剧"]: virtual_cloud_path = f"{virtual_cloud_path}/Season {local_season_num}"
        elif local_season_num > 1 and b_large not in ["电影", "演唱会"]: virtual_cloud_path = f"{virtual_cloud_path}/Season {local_season_num}"
            
        strm_sub_dir = virtual_cloud_path.replace(DIR_CAS_ROOT, "").strip("/")
        t_strm_dir = os.path.join(local_139_strm_dir, strm_sub_dir).replace("\\", "/")
        os.makedirs(t_strm_dir, exist_ok=True)
        
        files_to_process = []
        
        if strm_mode_139 in ["mode_b_139", "both_139"]:
            try:
                r_log = requests.post(f"{API_5255_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                if r_log.get("code") == 200:
                    h_139 = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                    
                    if is_webhook:
                        for _ in range(60):
                            has_new = False
                            fs_resp = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": cloud_dir, "refresh": True}, headers=h_139, timeout=10).json()
                            if fs_resp.get("code") == 200:
                                content = (fs_resp.get("data") or {}).get("content", [])
                                for item in content:
                                    if not item["is_dir"] and any(item["name"].lower().endswith(ext) for ext in ['.cas', '.iso']):
                                        c_f_path = f"{cloud_dir}/{item['name']}"
                                        c_size = item.get("size", 0)
                                        
                                        base_clean_name = item["name"][:-4] if item["name"].lower().endswith('.cas') else item["name"]
                                        strm_n = base_clean_name + ".strm" if strm_mode_139 == "both_139" else (base_clean_name + "-139B.strm" if strm_mode_139 == "mode_b_139" else base_clean_name + "-139A.strm")
                                        target_strm_path = os.path.join(t_strm_dir, strm_n).replace("\\", "/")
                                        
                                        cursor.execute("SELECT file_size FROM history WHERE file_path = ?", (c_f_path,))
                                        record = cursor.fetchone()
                                        
                                        if force_rebuild or not record or record[0] != c_size or not os.path.exists(target_strm_path):
                                            if record and record[0] == c_size and not os.path.exists(target_strm_path):
                                                logger.info(f"♻️ [自动重置] 雷达发现目标 STRM 已被物理删除，无视旧记忆，准备重新生成: {item['name']}")
                                            files_to_process.append(item["name"])
                                            cursor.execute("INSERT OR REPLACE INTO history (file_path, mtime, file_size) VALUES (?, ?, ?)", (c_f_path, 0, c_size))
                                            has_new = True
                            if has_new: 
                                time.sleep(3.0)
                                break
                            time.sleep(2.0)
                    else:
                        fs_resp = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": cloud_dir, "refresh": True}, headers=h_139, timeout=10).json()
                        if fs_resp.get("code") == 200:
                            content = (fs_resp.get("data") or {}).get("content", [])
                            for item in content:
                                if not item["is_dir"] and any(item["name"].lower().endswith(ext) for ext in ['.cas', '.iso']):
                                    c_f_path = f"{cloud_dir}/{item['name']}"
                                    c_size = item.get("size", 0)
                                    
                                    base_clean_name = item["name"][:-4] if item["name"].lower().endswith('.cas') else item["name"]
                                    strm_n = base_clean_name + ".strm" if strm_mode_139 == "both_139" else (base_clean_name + "-139B.strm" if strm_mode_139 == "mode_b_139" else base_clean_name + "-139A.strm")
                                    target_strm_path = os.path.join(t_strm_dir, strm_n).replace("\\", "/")
                                    
                                    cursor.execute("SELECT file_size FROM history WHERE file_path = ?", (c_f_path,))
                                    record = cursor.fetchone()
                                    
                                    if force_rebuild or not record or record[0] != c_size or not os.path.exists(target_strm_path):
                                        if record and record[0] == c_size and not os.path.exists(target_strm_path):
                                            logger.info(f"♻️ [自动重置] 雷达发现目标 STRM 已被物理删除，无视旧记忆，准备重新生成: {item['name']}")
                                        files_to_process.append(item["name"])
                                        cursor.execute("INSERT OR REPLACE INTO history (file_path, mtime, file_size) VALUES (?, ?, ?)", (c_f_path, 0, c_size))
            except Exception: pass
                
        else:
            if is_webhook:
                for _ in range(120):
                    has_new = False
                    if os.path.exists(local_dir) and os.path.isdir(local_dir):
                        for f in os.listdir(local_dir):
                            if not any(f.lower().endswith(ext) for ext in ['.cas', '.iso']): continue
                            loc_f_path = os.path.join(local_dir, f).replace("\\", "/")
                            l_stat = os.stat(loc_f_path)
                            
                            base_clean_name = f[:-4] if f.lower().endswith('.cas') else f
                            strm_n = base_clean_name + ".strm" if strm_mode_139 == "both_139" else (base_clean_name + "-139B.strm" if strm_mode_139 == "mode_b_139" else base_clean_name + "-139A.strm")
                            target_strm_path = os.path.join(t_strm_dir, strm_n).replace("\\", "/")
                            
                            cursor.execute("SELECT mtime, file_size FROM history WHERE file_path = ?", (loc_f_path,))
                            record = cursor.fetchone()
                            
                            if force_rebuild or not record or record[0] != l_stat.st_mtime or record[1] != l_stat.st_size or not os.path.exists(target_strm_path):
                                if record and record[0] == l_stat.st_mtime and record[1] == l_stat.st_size and not os.path.exists(target_strm_path):
                                    logger.info(f"♻️ [自动重置] 雷达发现目标 STRM 已被物理删除，无视旧记忆，准备重新生成: {f}")
                                files_to_process.append(f)
                                cursor.execute("INSERT OR REPLACE INTO history (file_path, mtime, file_size) VALUES (?, ?, ?)", (loc_f_path, l_stat.st_mtime, l_stat.st_size))
                                has_new = True
                    if has_new:
                        time.sleep(3.0)
                        break
                    time.sleep(1.0)
            else:
                if os.path.exists(local_dir) and os.path.isdir(local_dir):
                    for f in os.listdir(local_dir):
                        if not any(f.lower().endswith(ext) for ext in ['.cas', '.iso']): continue
                        loc_f_path = os.path.join(local_dir, f).replace("\\", "/")
                        l_stat = os.stat(loc_f_path)
                        
                        base_clean_name = f[:-4] if f.lower().endswith('.cas') else f
                        strm_n = base_clean_name + ".strm" if strm_mode_139 == "both_139" else (base_clean_name + "-139B.strm" if strm_mode_139 == "mode_b_139" else base_clean_name + "-139A.strm")
                        target_strm_path = os.path.join(t_strm_dir, strm_n).replace("\\", "/")
                        
                        cursor.execute("SELECT mtime, file_size FROM history WHERE file_path = ?", (loc_f_path,))
                        record = cursor.fetchone()
                        
                        if force_rebuild or not record or record[0] != l_stat.st_mtime or record[1] != l_stat.st_size or not os.path.exists(target_strm_path):
                            if record and record[0] == l_stat.st_mtime and record[1] == l_stat.st_size and not os.path.exists(target_strm_path):
                                logger.info(f"♻️ [自动重置] 雷达发现目标 STRM 已被物理删除，无视旧记忆，准备重新生成: {f}")
                            files_to_process.append(f)
                            cursor.execute("INSERT OR REPLACE INTO history (file_path, mtime, file_size) VALUES (?, ?, ?)", (loc_f_path, l_stat.st_mtime, l_stat.st_size))
                
        conn.commit()
        conn.close()
        
        if not files_to_process:
            if is_webhook:
                logger.warning(f"⚠️ [139投递失败] 扫描完毕，没发现任何新 .cas 文件！(传输太慢或路径有误): {target_dir}")
            return None

        valid_files = 0
        for f in files_to_process:
            base_clean_name = f[:-4] if f.lower().endswith('.cas') else f
            if strm_mode_139 == "both_139": strm_n = base_clean_name + ".strm"
            elif strm_mode_139 == "mode_b_139": strm_n = base_clean_name + "-139B.strm"
            else: strm_n = base_clean_name + "-139A.strm"
            
            target_strm_path = os.path.join(t_strm_dir, strm_n).replace("\\", "/")
            if not force_rebuild and os.path.exists(target_strm_path): continue
            
            payload = {
                "source_cas_path": os.path.join(local_dir, f).replace("\\", "/"), 
                "cloud_cas_path": f"{cloud_dir}/{f}".replace("//", "/"), 
                "target_local_dir": t_strm_dir,
                "strm_name": strm_n,
                "show_name": show_folder_name,
                "mode": strm_mode_139
            }
            try:
                res = requests.post(f"{API_5000_URL}/api/make_strm", json=payload, timeout=5)
                if res.status_code == 200: valid_files += 1
            except: pass
            
        try:
            valid_exts = ('.srt', '.ass', '.ssa', '.vtt', '.idx', '.sub')
            if os.path.exists(local_dir) and os.path.isdir(local_dir):
                for sub_f in os.listdir(local_dir):
                    if sub_f.lower().endswith(valid_exts):
                        sub_src = os.path.join(local_dir, sub_f)
                        sub_dst = os.path.join(t_strm_dir, sub_f)
                        if not os.path.exists(sub_dst) or os.stat(sub_src).st_mtime > os.stat(sub_dst).st_mtime:
                            shutil.copy2(sub_src, sub_dst)
        except: pass
        
        return t_strm_dir if valid_files > 0 else None
    except Exception as e:
        logger.error(f"❌ [139引擎] 崩溃: {e}")
        return None
# ==========================================
# 🌟 139 外部下载脚本专属入口 (防线程崩溃装甲版)
# ==========================================
@trigger_app.route('/api/trigger_139', methods=['GET', 'POST'])
def trigger_139():
    target_dir = request.args.get('path')
    if not target_dir and request.is_json:
        target_dir = request.json.get('path')
        
    if target_dir:
        def handle_139_external():
            # 💥 关键修复：加入强力防爆网，任何错误都会打到控制台，绝不死得不明不白！
            try:
                logger.info(f"⚡ [外部139投递] 收到闪电信号: {target_dir}")
                
                t_strm_dir = sync_139_to_smart_strm(target_dir, is_webhook=True)
                
                if t_strm_dir:
                    s = load_json(SETTINGS_FILE)
                    bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                    refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                    
                    time.sleep(5.0)
                    subprocess.Popen([bash_path, refresh_sh, t_strm_dir], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                    logger.info(f"⚡ [外部139投递] 任务完成，已精准下发 Emby 刷新指令: {t_strm_dir}")
                    
            except Exception as e:
                logger.error(f"❌ [外部139投递] 后台线程发生致命崩溃: {e}")
                
        threading.Thread(target=handle_139_external).start()
        return "✅ 139 外部投递信号已接入前台缓冲队列", 200
        
    return "❌ 缺少 path 参数", 400
# ==========================================
# 🌟 5555 端口：全域双轨智能路由 (支持本地与云端混动)
# ==========================================
@trigger_app.route('/force_harvest', methods=['GET', 'POST'])
def force_harvest():
    target_dir = request.args.get('path')
    if not target_dir and request.is_json:
        target_dir = request.json.get('path')
        
    if target_dir:
        s = load_json(SETTINGS_FILE)
        local_base = s.get("local_dropbox_dir", DEFAULT_LOCAL_DROPBOX)
        local_root = s.get("local_storage_root", DEFAULT_LOCAL_ROOT)
        
        if target_dir.startswith(local_base) or target_dir.startswith(local_root):
            logger.info(f"⚡ 收到【本地投递】闪电信号！目标: {target_dir}")
            threading.Thread(target=scan_local_dropbox, args=(target_dir,)).start()
        else:
            logger.info(f"⚡ 收到【云端收割】精准信号！目标: {target_dir}")
            threading.Thread(target=process_cas_via_olist_api, args=(target_dir,)).start()
    else:
        logger.info("⚡ 收到全量收割信号！开始双轨扫描所有目录...")
        threading.Thread(target=scan_local_dropbox).start()
        threading.Thread(target=process_cas_via_olist_api).start()
        
    return "✅ 收割指令已交由后台接管", 200

if __name__ == '__main__':
    os.makedirs(DB_DIR, exist_ok=True)
    
    threading.Thread(target=lambda: trigger_app.run(host='0.0.0.0', port=TRIGGER_PORT, debug=False, use_reloader=False), daemon=True).start()
    logger.info(f"🎧 本地监听端口 {TRIGGER_PORT} 已启动，随时等待下载脚本呼叫...")

    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    notifier.send_message(f"🤖 追剧转存引擎 (V8.8 智能路由版) 启动，获取凭证...")
    
    try:
        logger.info("✅ [系统] 189底层接口握手成功，正在挂载凭证...")
        client = Cloud189()
        client.login(ENV_189_CLIENT_ID, ENV_189_CLIENT_SECRET)
        last_login_time = time.time()
        notifier.send_message("✅ 网盘登录成功！全天候监控已就位。")
    except Exception as e:
        logger.error(f"❌ [系统] 登录失败: {e}")
        notifier.send_message(f"❌ 首次登录失败: {e}\n(脚本已进入休眠模式防止被封)")
        time.sleep(600)
        sys.exit(-1)
        
    main_control_loop(client)
```