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
139 - ☁️ [139收割 处理139 扫139] 触发5255端口openlist生成cas
sync139 - ☁️ [同步139] 触发139移动云盘专属STRM生成
recloud - ☁️ [恢复云端] 触发云端增量生成STRM文件
reall - ☁️ [全库重建] 重建媒体库
cancel - 🚫 [取消] 解除监控任务并清理关联记忆
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
DEFAULT_LOCAL_ROOT = "/storage/"  # 用来智能判定是否为本地物理路径的通用前缀

# --- 139专属加工厂配置 ---
API_5255_URL = "http://127.0.0.1:5255"
DIR_139_SOURCE = "/141/141source"
DIR_139_TARGET = "/139/139cas"
DIR_LOCAL_CAS = "/storage/emulated/0/Download/139cas" 
DEFAULT_139_LOCAL_STRM = "/storage/emulated/0/Download/139_strm"
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
    
    # 1. 尝试生成拼音特征 (兼容没装 pypinyin 的情况)
    try:
        from pypinyin import pinyin, Style
        info["pinyin_full"] = "".join([p[0] for p in pinyin(keyword, style=Style.NORMAL)])
        info["pinyin_initial"] = "".join([p[0][0] for p in pinyin(keyword, style=Style.FIRST_LETTER)])
    except ImportError:
        pass # 如果没装库就不强求拼音

    # 2. 第一次请求：用中文去查，拿到正确的 ID
    url_cn = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={parse.quote(keyword)}&page=1"
    try:
        res_cn = requests.get(url_cn, timeout=5, proxies=LOCAL_PROXIES).json()
        if res_cn.get("results"):
            top = res_cn["results"][0]
            media_type = top.get("media_type", "tv") # 判断是电影还是剧集
            tmdb_id = str(top.get("id"))
            info["id"] = tmdb_id
            info["cn_name"] = top.get("name") or top.get("title", "")
            
            # 3. 第二次请求：用拿到的 ID，伪装成英文环境再去问一次，榨出官方英文名！
            url_en = f"https://api.themoviedb.org/3/{media_type}/{tmdb_id}?api_key={TMDB_API_KEY}&language=en-US"
            res_en = requests.get(url_en, timeout=5).json()
            # 提取英文环境下的剧名
            info["en_name"] = res_en.get("name") or res_en.get("title") or top.get("original_name") or ""
            
            return info
    except Exception as e:
        logger.error(f"TMDB检索与反查异常: {e}")
        
    # 如果 TMDB 彻底没查到，把拼音特征传回去保底
    return info if (info["pinyin_full"] or info["pinyin_initial"]) else None

def translate_folder_name(folder_name):
    """提取文件夹名中的TMDB ID，翻译成人话(中文剧名)"""
    if not TMDB_API_KEY: return folder_name
    match = re.search(r'(?i)tmdb[-_]?(\d+)', folder_name)
    if not match: return folder_name
    
    tmdb_id = match.group(1)
    try:
        # 先按剧集查
        res_tv = requests.get(f"https://api.themoviedb.org/3/tv/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN", timeout=3, proxies=LOCAL_PROXIES).json()
        if "name" in res_tv:
            cn_name = res_tv["name"]
            s_match = re.search(r'(?i)(S\d+|Season\s*\d+)', folder_name)
            s_tag = f" {s_match.group(1)}" if s_match else ""
            return f"📺 {cn_name}{s_tag} (TMDB-{tmdb_id})"
            
        # 查不到再按电影查
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
        # 1. 基础搜索，先锁定是最贴合的那部戏
        url_search = f"https://api.themoviedb.org/3/search/multi?api_key={TMDB_API_KEY}&language=zh-CN&query={parse.quote(keyword)}&page=1"
        res = requests.get(url_search, timeout=5, proxies=LOCAL_PROXIES).json()
        if not res.get("results"):
            return f"📭 TMDB 数据库中未找到关于【{keyword}】的信息。"
            
        top = res["results"][0]
        media_type = top.get("media_type", "tv")
        tmdb_id = top.get("id")
        
        # 2. 根据 ID 拿取极其详尽的完整档案
        detail_url = f"https://api.themoviedb.org/3/{media_type}/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN"
        detail_res = requests.get(detail_url, timeout=5, proxies=LOCAL_PROXIES).json()
        
        # 3. 基础共用信息清洗
        title = detail_res.get("name") or detail_res.get("title", "未知")
        original_title = detail_res.get("original_name") or detail_res.get("original_title", "")
        overview = detail_res.get("overview", "暂无简介")
        if len(overview) > 200: overview = overview[:197] + "..." # 防止简介太长刷屏
        vote = round(detail_res.get("vote_average", 0), 1)
        genres = ", ".join([g["name"] for g in detail_res.get("genres", [])])
        country = ", ".join(detail_res.get("origin_country", []))
        
        # 状态汉化字典
        status_trans = {
            "Returning Series": "📺 连载中", "Ended": "✅ 已完结", 
            "Canceled": "❌ 已砍掉", "Released": "✅ 已上映", 
            "Post Production": "🛠 后期制作中", "In Production": "🎥 拍摄中"
        }
        raw_status = detail_res.get("status", "未知")
        status = status_trans.get(raw_status, raw_status)

        # 4. 根据类型分发组装 Telegram 卡片
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
# 📁 核心目录与挂载配置
# ==========================================
DIR_CAS_ROOT = "/177-秒传"
DIR_VIDEO_ROOT = "/177-视频"    # 🌟 新增：新版普通视频的专属智能路由根目录      
DIR_MEDIA_PREFIX = "/177-"      
OPENLIST_MOUNT_POINT = "177"    

SUBS_FILE = os.path.join(DB_DIR, "subscriptions.json")
HARVEST_SUBS_FILE = os.path.join(DB_DIR, "harvest_subs.json") # 🌟 新增收割专属库
HISTORY_FILE = os.path.join(DB_DIR, "history.json")
COOKIES_FILE = os.path.join(DB_DIR, "cookies.json")
SETTINGS_FILE = os.path.join(DB_DIR, "settings.json") # 🌟 新增配置保存

last_login_time = 0

# 🌟 分类字典 (保留你需要的路由分类)
CAT_ROUTER = {
    "华语剧": ("电视剧", "0-电视剧"), "大陆剧": ("电视剧", "0-电视剧"), "港剧": ("电视剧", "0-电视剧"), "台剧": ("电视剧", "0-电视剧"),
    "华语剧2601": ("电视剧", "0-电视剧"), 
    "华语剧2602": ("电视剧", "0-电视剧"),
    "华语剧2701": ("电视剧", "0-电视剧"),
    "欧美剧": ("电视剧", "1-电视剧"), "美剧": ("电视剧", "1-电视剧"), "英剧": ("电视剧", "1-电视剧"),
    "欧美剧2601": ("电视剧", "1-电视剧"),
    "欧美剧2701": ("电视剧", "1-电视剧"),
    "日韩剧": ("电视剧", "2-电视剧"), "韩剧": ("电视剧", "2-电视剧"), "日剧": ("电视剧", "2-电视剧"),
    "日韩剧2601": ("电视剧", "2-电视剧"),
    "日韩剧2701": ("电视剧", "2-电视剧"),
    "华语电影": ("电影", "0-电影"), "国语电影": ("电影", "0-电影"),
    "华语电影2601": ("电影", "0-电影"),
    "华语电影2701": ("电影", "0-电影"),
    "欧美电影": ("电影", "1-电影"), "大片": ("电影", "1-电影"),
    "欧美电影2601": ("电影", "1-电影"),
    "欧美电影2701": ("电影", "1-电影"),
    "日韩电影": ("电影", "2-电影"),
    "国漫": ("动漫", "0-动漫"), 
    "国漫2601": ("动漫", "0-动漫"), 
    "国漫2602": ("动漫", "0-动漫"),
    "国漫2701": ("动漫", "0-动漫"),
    "日漫": ("动漫", "1-动漫"), "番剧": ("动漫", "1-动漫"),
    "综艺": ("综艺", ""), "纪录片": ("纪录片", ""), "演唱会": ("演唱会", ""), "短剧": ("短剧", "")
}

def load_json(filepath):
    if os.path.exists(filepath):
        try:
            with open(filepath, 'r', encoding='utf-8') as f:
                content = f.read().strip()
                if not content: 
                    return {}  # 🌟 防御 1：如果是0字节空文件，直接返回空字典，绝不报错
                return json.loads(content)
        except Exception as e:
            # 🌟 防御 2：如果 JSON 格式彻底烂了，拦截报错并输出日志，保证系统继续运行
            logger.error(f"🚨 [致命警告] 数据库文件损坏，已自动重置为空库: {os.path.basename(filepath)} - 错误: {e}")
            return {}
    return {}

def save_json(filepath, data):
    """🛡️ 原子级写入：先写临时文件再瞬间替换，绝对防止断电或高并发导致的文件损坏"""
    tmp_path = f"{filepath}.tmp"
    try:
        # 1. 先安全地把数据写到临时文件里
        with open(tmp_path, 'w', encoding='utf-8') as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        # 2. 瞬间重命名覆盖原文件 (在 Linux/Android 底层这是原子操作，绝对不会冲突)
        os.replace(tmp_path, filepath)
    except Exception as e:
        logger.error(f"🚨 [写盘错误] 无法保存数据到 {filepath}: {e}")

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

# 🌟 将洗名函数提到全局，让收割库也能公用
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
    # --- 🌟 核心一：侦测并保留 .mp4.cas 这种双后缀 ---
    valid_media_exts = ['.mp4', '.mkv', '.ts', '.avi', '.rmvb', '.flv', '.wmv', '.srt', '.ass']
    lower_name = original_filename.lower()
    final_ext = ""
    
    for me in valid_media_exts:
        if lower_name.endswith(f"{me}.cas"):
            final_ext = f"{me}.cas"  # 锁定双后缀
            break
            
    if not final_ext:
        _, ext = os.path.splitext(original_filename)
        if ext.lower() in valid_media_exts or ext.lower() == '.cas':
            final_ext = ext.lower()
            
    # 🚑 抢救瞎子文件：避开图片垃圾，剩下的强行套 .cas
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
    
    # --- 🌟 核心修改：无情绞肉机 (以年份为界，一刀切断) ---
    clean_show_name = folder_name
    # 1. 只要碰到 (四位年份)，把年份和它屁股后面的“所有任意标签”一刀剁掉
    clean_show_name = re.sub(r'\s*\(\d{4}\).*$', '', clean_show_name)
    # 2. 无年份兜底：如果没写年份，用常规黑名单兜底，顺便清理 tmdb 尾巴 (连带外面的花括号/方括号一起杀)
    clean_show_name = re.sub(r'(?i)[_\-\s]*(HQ|IQ|DV|4K|1080[pP]|720[pP]|2160[pP]|WEB-DL|HDR|SDR|HD|H\.?26[45]|x\.?26[45]|BluRay|Remux)[_\-\s]*', '', clean_show_name)
    clean_show_name = re.sub(r'(?i)[\[{\(]?tmdb[-_=]?\w+[\]}\)]?', '', clean_show_name)
    # 3. 修剪首尾垃圾字符，并用点号替换空格
    clean_show_name = re.sub(r'[-_\s]+$', '', clean_show_name).strip()
    clean_show_name = clean_show_name.replace(' ', '.') 
    
    # --- 🌟 核心二：精准提取你要的组合标签 (通杀带点号的 H.265/x.264) ---
    tags_match = re.findall(r'(?i)\b(1080p|2160p|4K|DV|HQ|HDR|SDR|IQ|H\.?26[45]|x\.?26[45])\b', original_filename)
    tags = []
    for t in tags_match:
        # 第一步：全部转大写，并且无情抹除中间的点号 (把 H.265 变成 H265)
        t_upper = t.upper().replace('.', '')
        
        # 第二步：照顾命名强迫症，统一规范大小写
        if t_upper == '1080P': t_upper = '1080p'
        elif t_upper == '2160P': t_upper = '2160p'
        elif t_upper == 'X264': t_upper = 'H264'
        elif t_upper == 'X265': t_upper = 'H265'
        
        if t_upper not in tags:
            tags.append(t_upper)
            
    tag_str = "." + ".".join(tags) if tags else ""

    # 🎬 电影/单次任务组装区 (这里自动接上了 tag_str)
    if any(k in sub_path for k in ["电影", "movie", "演唱会", "纪录片"]):
        part_match = re.search(r'(?i)(part\d+|cd\d+)', original_filename)
        part_str = f".{part_match.group(1).lower()}" if part_match else ""
        year_part = f".{year_str}" if year_str else ""
        return f"{clean_show_name}{year_part}{part_str}{tag_str}{final_ext}".replace('..', '.')

    # 📺 剧集/动漫组装区
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
    
    # 🌟 只有在非强制唤醒时，才受 30 分钟冷却锁限制
    if not force and (current_time - last_login_time < 1800):
        logger.warning("⏳ [系统] 检测到接口报错，防风控冷却锁生效，跳过登录！")
        return False
        
    logger.info("🔄 [系统] 触发保活机制：正在彻底重洗内存与协议握手...")
    try:
        # 💥 核心修复：彻底粉碎内存中残留的旧 Session 幽灵！
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
# 🌟 139 专属独立全自动流水线 (挂载在 5255 端口)
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
    
    # 🌟 核心：准备一个篮子，循环里只记录不刷新，整剧全部执行完只留 1 个去重目录
    strm_dirs_to_refresh = set()
    
    for f in source_files:
        orig_name = f["name"]
        src_dir = f["dir"]
        
        # 过滤非视频文件
        if not any(orig_name.lower().endswith(ext) for ext in ['.mp4', '.mkv', '.ts']): continue
            
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
                # 💡 修正1：在正则的分隔符白名单里加上波浪号 ~
                m = re.match(r'^(?:S\d{1,2}E|EP|E|第)?(\d{1,4})(?:集|话)?[\s\._\-~]*(.*)', orig_name, re.IGNORECASE)
                if m:
                    ep_num = m.group(1).zfill(2)
                    # 💡 修正2：剥离后缀时，把两端可能残留的杂质再强力剥刮一次
                    remainder = m.group(2).strip(" ~._-")
                    if remainder: clean_name = f"{show_name}.{season_str}E{ep_num}.{remainder}"
                    else: clean_name = f"{show_name}.{season_str}E{ep_num}{orig_name[orig_name.rfind('.'):]}"
                    
                    # 💡 修正3：二次净化，防止出现 ".~" 这种怪异拼接
                    clean_name = clean_name.replace(" ", ".").replace("..", ".").replace(".~", ".").replace("~.", ".")
                else:
                    try: clean_name = generate_smart_name(orig_name, src_dir) or orig_name
                    except: pass
        
        # 建立目标目录，非电影类若无 Season/季 自动补全 /Season 1
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
            wait_start_time = time.time() # 用于防空包等待计时
            
            while True:
                time.sleep(5.0)
                task_in_undone = False
                current_state_val = -1
                
                # 第一步：先看一眼运行队列
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
                    logger.warning(f"⚠️ [拉取状态异常] {e}")
                
                # 路线A（跨盘龟速）：任务在运行，启动防卡死测速
                if task_in_undone:
                    task_ever_seen = True
                    wait_start_time = time.time() # 只要还在走，就不算等空包
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
                            logger.warning(f"🔴 [龟速/卡死警告] 实测速度低于 1MB/s ({current_speed_bytes/1024/1024:.2f} MB/s)！警告积累 ({stuck_count}/6)")
                        else:
                            stuck_count = 0
                            
                        # 连续 6 次 (约 30 秒) 龟速或卡死，执行 URL 强杀
                        if stuck_count >= 6:
                            logger.error(f"🔪 [触发斩首] 连续30秒低速或卡死，执行强杀...")
                            if target_task_id:
                                try:
                                    requests.post(f"{API_5255_URL}/api/task/copy/cancel?tid={target_task_id}", headers=headers_139, timeout=5)
                                    time.sleep(1.0)
                                    requests.post(f"{API_5255_URL}/api/task/copy/delete?tid={target_task_id}", headers=headers_139, timeout=5)
                                    time.sleep(3.0) 
                                except Exception as e:
                                    logger.error(f"❌ [强杀请求网络报错] {e}")
                            task_failed = True
                            break
                            
                    last_loaded_bytes = current_loaded
                    last_check_time = now
                    continue
                
                # 路线B（同盘秒传）：不在队列里，直接突击查岗目标文件夹！
                try:
                    fs_resp = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": target_dir, "refresh": True}, headers=headers_139, timeout=10)
                    fs_data = fs_resp.json()
                    if fs_data.get("code") == 200:
                        contents = [item["name"] for item in (fs_data.get("data", {}).get("content", []) or [])]
                        
                        # 只要扫到了，管它是原名还是cas，一律判定大圆满成功！
                        if orig_name in contents:
                            actual_cas_name = orig_name
                            task_success = True
                            logger.info(f"✅ [结果查岗] 秒传或传输完毕，目标区发现原视频: {actual_cas_name}")
                            break
                        elif f"{orig_name}.cas" in contents:
                            actual_cas_name = f"{orig_name}.cas"
                            task_success = True
                            logger.info(f"✅ [结果查岗] 目标区已发现CAS: {actual_cas_name}")
                            break
                        elif f"{orig_name.rsplit('.', 1)[0]}.cas" in contents:
                            actual_cas_name = f"{orig_name.rsplit('.', 1)[0]}.cas"
                            task_success = True
                            logger.info(f"✅ [结果查岗] 目标区已发现短名CAS: {actual_cas_name}")
                            break
                except Exception as e:
                    logger.warning(f"❌ [139加工] 目标查岗网络异常: {e}")
                
                # 异常判定1：以前在队列里出现过，现在没了，且目标区也没查到，说明失败
                if task_ever_seen:
                    logger.warning(f"❌ [异常断线] 任务消失，且目标区无文件，判定失败准备重试...")
                    task_failed = True
                    break
                
                # 异常判定2：等了15秒，没在队列出现，目标区也没有，说明下发死链了
                if time.time() - wait_start_time > 15.0:
                    logger.warning(f"⚠️ [下发超时] 超过15秒毫无动静，判定死链，准备重试...")
                    task_failed = True
                    break
                
                # 没超时就继续转下一圈等
                
            if task_success:
                break
            elif task_failed:
                continue
                
        if not task_success:
            logger.error(f"❌ [139加工] 连续重试均失败，彻底放弃: {orig_name}")
            continue

        # ==========================================
        # 4. 任务成功后：重命名、下载真实CAS镜像、销毁源、触发入库
        # ==========================================
        target_cas_name = f"{clean_name}.cas"
        if actual_cas_name and actual_cas_name != target_cas_name:
            requests.post(f"{API_5255_URL}/api/fs/rename", json={"name": target_cas_name, "path": f"{target_dir}/{actual_cas_name}"}, headers=headers_139).close()
            time.sleep(1.0)
            
        # 🌟 真实下载：把云端的 .cas 文件下载到本地备份目录
        try:
            local_target_dir = target_dir.replace(DIR_139_TARGET, DIR_LOCAL_CAS)
            os.makedirs(local_target_dir, exist_ok=True)
            local_cas_path = os.path.join(local_target_dir, target_cas_name)
            
            cloud_cas_path = f"{target_dir}/{target_cas_name}"
            
            # 1. 请求云端文件直链
            r_get = requests.post(f"{API_5255_URL}/api/fs/get", json={"path": cloud_cas_path}, headers=headers_139, timeout=10)
            if r_get.status_code == 200 and r_get.json().get("code") == 200:
                raw_url = r_get.json().get("data", {}).get("raw_url")
                if raw_url:
                    # 2. 流式下载真实内容到本地硬盘
                    r_download = requests.get(raw_url, stream=True, timeout=60)
                    if r_download.status_code == 200:
                        with open(local_cas_path, 'wb') as local_f:
                            for chunk in r_download.iter_content(chunk_size=8192):
                                if chunk:
                                    local_f.write(chunk)
                        logger.info(f"✨ [139加工] 本地真实 CAS 镜像下载成功: {local_cas_path}")
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
        
        # 触发管家 5000 生成 STRM
        try: requests.get(f"{API_5000_URL}/api/sync?drive=139&path={parse.quote(target_dir)}", timeout=3).close()
        except: pass

        # 🌟 算好真实的本地 strm 目录，扔进篮子（绝对不在此处开枪）
        try:
            s = load_json(SETTINGS_FILE)
            local_139_strm_dir = s.get("local_139_strm_dir", DEFAULT_139_LOCAL_STRM)
            strm_sub_dir = target_dir.replace(DIR_139_TARGET, "").strip("/")
            real_strm_path = os.path.join(local_139_strm_dir, strm_sub_dir).replace("\\", "/")
            strm_dirs_to_refresh.add(real_strm_path)
        except: pass

    # ==========================================
    # 5. 全部清理已完成任务
    # ==========================================
    try: requests.post(f"{API_5255_URL}/api/task/copy/clear_done", headers={"Authorization": headers_139["Authorization"]}, timeout=5).close()
    except: pass
    
    # 🌟 终极闭环：循环处理完后，对去重后的目录统一发一次精准刷新指令！
    if strm_dirs_to_refresh:
        try:
            s = load_json(SETTINGS_FILE)
            bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
            refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
            
            def delayed_139_refresh(dirs):
                time.sleep(5.0) # 给管家 5 秒钟写盘时间
                for d in dirs:
                    subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                    logger.info(f"⚡ [139聚合刷新] 已精准下发 Emby 刷新指令: {d}")
                    time.sleep(1.0) # 错峰保护
                    
            threading.Thread(target=delayed_139_refresh, args=(strm_dirs_to_refresh,)).start()
        except Exception as e:
            logger.error(f"❌ [139加工] 唤醒 Emby 刷新异常: {e}")

# ==========================================
# 🌟 独立新增的 CAS 收割模块 (终极暴力认亲 + 整季合并极速批次处理版)
# ==========================================
def process_cas_via_olist_api(specific_dir=None):
    s = load_json(SETTINGS_FILE)
    WATCH_DIRS = s.get("watch_dirs", ["/family/177_cas", "/local_cas"])
    
    # 🌟 如果有外部指定的精准目录，雷达就只扫这一个！
    if specific_dir:
        WATCH_DIRS = [specific_dir]
    NO_DELETE_DIRS = s.get("no_delete_dirs", [])
    
    # 🌟 新增：把个人云目录自动硬塞进巡逻大军里，去重防止扫两遍
    WATCH_DIRS = list(set(WATCH_DIRS + NO_DELETE_DIRS))
    processed_names = []
    if not WATCH_DIRS: return processed_names 

    # --- 1. 载入【收割专属账本】(防个人云二次复制) ---
    harvest_ledger_file = os.path.join(DB_DIR, "harvest_ledger.json")
    harvest_ledger = load_json(harvest_ledger_file)
    harvest_ledger_changed = False

    # --- 2. 载入【STRM静默账本】(防 5000 端口二次强刷) ---
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
                        # 【核心拦截】：如果文件已在收割账本，直接跳过扫描
                        if item["name"] in harvest_ledger:
                            logger.debug(f"⏭️ [跳过] 已记录于收割账本: {item['name']}")
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
                    r = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": root_path, "password": "", "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=20)
                    if r.json().get("code") == 200:
                        ym_nodes = [item["name"] for item in (r.json().get("data") or {}).get("content", []) if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                        ym_nodes.sort(reverse=True)
                        all_months_matches = []
                        for ym in ym_nodes:
                            ym_path = f"{root_path}/{ym}"
                            r2 = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": ym_path, "password": "", "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=20)
                            if r2.json().get("code") == 200:
                                for item in (r2.json().get("data") or {}).get("content", []):
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

        # ==========================================
        # 🌟 核心保护：个人云原盘直通，绝不洗名
        # ==========================================
        is_no_delete = any(nd in raw_dir for nd in NO_DELETE_DIRS)
        if is_no_delete:
            final_name = filename  # 个人云：原封不动，保护多版本特征
            logger.debug(f"🛡️ [保护] 识别到个人云专属文件，已跳过洗名: {filename}")
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

        # 【核心分支】：判断免删除目录
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

            # 🌟 修复：严密隔离的双轨通知机制
            if is_no_delete:
                for name in names_to_move: harvest_ledger[name] = True
                harvest_ledger_changed = True
                logger.info(f"🛑 [个人云] 纯备份完成，绝不投递管家制造垃圾 STRM")
            else:
                need_sync = False
                for name in names_to_move:
                    if name in strm_ledger:
                        logger.info(f"🤫 [静默防重] 本地已有 STRM，拦截管家二次强刷: {name}")
                        del strm_ledger[name]
                        strm_ledger_changed = True
                    else:
                        need_sync = True
                        
                if need_sync:
                    updated_paths.add(base_notify_path)
                
        else:
            logger.error(f"❌ [{action_name}] 批量失败: {mov_res.get('message')}")
            # 单件降级处理，同样继承全部逻辑并隔离
            for name in names_to_move:
                single_res = requests.post(f"{API_5244_URL}/api/fs/{api_endpoint}", json={"src_dir": raw_dir, "dst_dir": final_target_dir, "names": [name]}, headers=headers).json()
                if single_res.get("code") == 200:
                    logger.info(f"✅ [{action_name}] 单件成功: {name}")
                    processed_names.append(name)
                    
                    if is_no_delete:
                        harvest_ledger[name] = True
                        harvest_ledger_changed = True
                        logger.info(f"🛑 [个人云单件] 纯备份完成，跳过 STRM 投递: {name}")
                    else:
                        if name in strm_ledger:
                            logger.info(f"🤫 [静默防重单件] 拦截管家二次强刷: {name}")
                            del strm_ledger[name]
                            strm_ledger_changed = True
                        else:
                            updated_paths.add(base_notify_path)

    # --- 循环结束，分别保存核销好的双账本 ---
    if harvest_ledger_changed:
        save_json(harvest_ledger_file, harvest_ledger)
    if strm_ledger_changed:
        save_json(strm_ledger_file, strm_ledger)

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
        
        # 从设置里读取你想要的模式，默认是老版的 "cas"
        strm_mode = s.get("189_strm_mode", "cas") 
        
        # 👇 新增：准备去重篮子
        strm_dirs_to_refresh = set()
        
        for media_root in target_media_roots:
            olist_p = get_openlist_path(media_root)
            try:
                # 1. 派发造物指令给 5000 管家
                requests.get(f"{API_5000_URL}/api/sync", params={"path": olist_p, "mode": strm_mode}, timeout=3).close()
                logger.info(f"🔄 [管家] 触发云端目录同步: {media_root} (模式: {strm_mode})")
                
                # 2. 算路：将云端路径转换为本地 STRM 物理路径并装入篮子
                local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
                # 巧妙利用 split 切掉云端根目录前缀，得出相对子目录
                strm_sub_dir = media_root.split(DIR_CAS_ROOT)[-1].strip("/")
                target_local_strm_path = os.path.join(local_strm_dir, strm_sub_dir).replace("\\", "/")
                strm_dirs_to_refresh.add(target_local_strm_path)
                
            except Exception as e:
                logger.debug(f"⚠️ 通知目录强刷闪断: {e}")
                
        # 👇 新增：异步延迟，统一精准刷新 Emby
        if strm_dirs_to_refresh:
            bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
            refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
            
            def delayed_harvest_refresh(dirs):
                time.sleep(10.0) # 给管家 10 秒钟将整剧 STRM 写盘
                for d in dirs:
                    subprocess.Popen([bash_path, refresh_sh, d], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                    logger.info(f"⚡ [收割聚合刷新] 已精确唤醒 Emby 局部更新: {d}")
                    time.sleep(1.0) # 错峰保护
                    
            threading.Thread(target=delayed_harvest_refresh, args=(strm_dirs_to_refresh,)).start()
                
    return processed_names

# 🌟 查重逻辑与自动订阅拉取
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

            # 🌟 加入 ignore_time 参数
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
                
                # ====================================================
                # 🎯 原版的进阶集数去重逻辑
                # ====================================================
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
                # ====================================================
                
                new_files.append(f)

            if new_files:
                logger.info(f"🎯 [搬运] 锁定 {len(new_files)} 个增量更新文件，开始物理下发...")
                
                taskInfos = [{"fileId": f["id"], "fileName": clean_filename(f["name"]), "isFolder": 0} for f in new_files]
                
                # ====================================================
                # 🚀 终极防毒隔离装甲：批次下发 + 幸存者强行洗名抢救
                # ====================================================
                code = info.saveShareFiles(taskInfos, target_id)
                
                # 1. 无论返回成功还是特征码报错，都强行给云端 8 秒缓冲时间
                time.sleep(8)
                fresh_cloud_files = client_obj.listPrivateFiles(target_id)
                fresh_names = [cf["name"] for cf in fresh_cloud_files]
                
                actually_saved_count = 0
                saved_tasks = []
                failed_tasks = []

                # 2. 逐一核对实体盘，把真正成功存进去的“幸存者”捞出来
                for task in taskInfos:
                    orig_name = task["fileName"]
                    expected_smart_name = generate_smart_name(orig_name, path)
                    
                    if orig_name in fresh_names or (expected_smart_name and expected_smart_name in fresh_names):
                        history[str(task["fileId"])] = {"name": orig_name, "sub_id": str(target_id)}
                        actually_saved_count += 1
                        saved_tasks.append(task)
                    else:
                        failed_tasks.append(task)

                # 3. 只要有幸存者落地，立刻存入记忆并执行智能洗名！绝不放生！
                if actually_saved_count > 0:
                    save_json(HISTORY_FILE, history)
                    if isinstance(subs.get(str(target_id)), dict):
                        now_dt = datetime.now()
                        subs[str(target_id)]["last_update"] = int(time.time())
                        if freq == "周更" or "周更" in path or "动漫" in path:
                            subs[str(target_id)]["last_success_week"] = now_dt.strftime("%Y-%V")
                            subs[str(target_id)]["last_success_day"] = now_dt.strftime("%Y-%m-%d")
                        save_json(SUBS_FILE, subs)
                        
                    # ✨ 幸存者洗名重命名
                    renamed_files_list = []
                    for cf in fresh_cloud_files:
                        hist_info = history.get(str(cf["id"]))
                        orig_name = hist_info["name"] if hist_info else cf["name"]
                        new_name = generate_smart_name(orig_name, path)
                        
                        if new_name and cf["name"] != new_name:
                            if client_obj.renameFile(cf["id"], new_name):
                                renamed_files_list.append(new_name)
                                time.sleep(0.5) 
                                
                    # 播报落地战果
                    notifier.send_message(f"✅【追剧落地报告】\n🔗 来源: {share_url}\n📂 成功入库并洗名 {actually_saved_count} 个文件！")
                    if renamed_files_list:
                        if len(renamed_files_list) > 20:
                            rename_msg = "\n".join([f" └ {n}" for n in renamed_files_list[:20]]) + f"\n...等共 {len(renamed_files_list)} 个文件"
                        else:
                            rename_msg = "\n".join([f" └ {n}" for n in renamed_files_list])
                        notifier.send_message(f"✨ 云端洗名规范化完成:\n{rename_msg}")

                # 4. 如果遇到特征码拦截，且有文件没存上，触发精准报警
                if code not in [0, '0', None, False, '']:
                    has_transfer_error = True  # 锁死上一步跟你说的归档自杀行为
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
                # 🌟 如果是未知的底层风控，先删 Cookie 保证重登能拿到新 Token
                if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                auto_relogin(client_obj, force=True)

    if global_cas_paths:
        s = load_json(SETTINGS_FILE)
        # 👇 新增：读取环境路径并准备篮子
        bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
        refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
        local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
        strm_dirs_to_refresh = set()
        
        for p in global_cas_paths:
            try:
                # 1. 派发造物指令给 5000 管家
                requests.get(f"{API_5000_URL}/api/sync", params={"path": p}, timeout=3) 
                logger.info(f"⚡ [API] 成功向管家后方下发同步指令: {p}")
                notifier.send_message(f"✅ 管家同步指令已下发: {p}")
                
                # 2. 算路：提取出相对子目录并转成本地绝对路径，扔进篮子
                strm_sub_dir = p.split(DIR_CAS_ROOT)[-1].strip("/")
                target_local_strm_path = os.path.join(local_strm_dir, strm_sub_dir).replace("\\", "/")
                strm_dirs_to_refresh.add(target_local_strm_path)
                
            except Exception as e: 
                logger.error(f"❌ [API] 管家服务无法联通: {e}")
                notifier.send_message(f"❌ 管家同步无响应: {e}")
            time.sleep(1) 
            
        # 👇 新增：统一异步延迟精准刷新 Emby
        if strm_dirs_to_refresh:
            def delayed_sub_refresh(dirs):
                time.sleep(10.0) # 等 5000 端口写盘完毕
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
    # 🌟 坚决使用变量，绝不写死路径！
    dropbox_dir = s.get("local_dropbox_dir", DEFAULT_LOCAL_DROPBOX)
    local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
    strm_mode = s.get("189_strm_mode", "cas")
    
    scan_target = specific_dir if specific_dir else dropbox_dir

    history_file = os.path.join(DB_DIR, "monitor_history.json")
    monitor_history = load_json(history_file)
    changed = False
    updated_dirs = set()

    # 🌟 100% 恢复你原汁原味的清理逻辑，一行不差！
    keys_to_delete = [path for path in monitor_history.keys() if not os.path.exists(path)]
    for k in keys_to_delete:
        del monitor_history[k]
        changed = True

    # ==========================================
    # 🌟 核心防空包穿透雷达 (后台安全运行，最多等 15 秒)
    # ==========================================
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
            
            # 只要抓到哪怕一个 .cas 文件，说明文件落地了！
            if cas_files:
                time.sleep(3.0) # 给局域网 1 秒时间把内容写完，防空包
                break
            time.sleep(1.0) # 没找到就耐心等 3 秒再看一眼
    else:
        # 定时全量扫描
        if os.path.isfile(scan_target) and scan_target.lower().endswith('.cas'):
            cas_files.append(scan_target.replace("\\", "/"))
        elif os.path.isdir(scan_target):
            for root, dirs, files in os.walk(scan_target):
                for f in files:
                    if f.lower().endswith('.cas'):
                        cas_files.append(os.path.join(root, f).replace("\\", "/"))

    if not cas_files:
        logger.warning(f"⚠️ [投递失败] 扫描完毕，没发现任何 .cas 文件！(传输太慢或路径有误): {scan_target}")
        if changed: save_json(history_file, monitor_history)
        return

    # 🧠 开始智能算命
    for file_path in cas_files:
        try:
            file_stat = os.stat(file_path)
            size = file_stat.st_size
            mtime = file_stat.st_mtime
            filename = file_path.split("/")[-1]

            # 🛡️ 恢复严苛防重 + 🌟 新增：终极物理自动探针！
            if file_path in monitor_history:
                record = monitor_history[file_path]
                if record.get('mtime') == mtime and record.get('file_size') == size:
                    # 🌟 真正的自动化：亲自去硬盘查岗！
                    strm_dir = record.get('target_strm_dir', '')
                    strm_name = record.get('strm_name', '')
                    if strm_dir and strm_name:
                        strm_full_path = os.path.join(strm_dir, strm_name)
                        # 如果你没删，文件还在，就正常防重复跳过
                        if os.path.exists(strm_full_path):
                            continue  
                        else:
                            # 如果你把那个 strm 删了，它立马察觉，直接无视历史记忆！
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

            # 这里的 clean_show_name 只用来传给管家作剧名记录，不影响最后生成的文件名
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
                s_match_file = re.search(r'(?i)S0*(\d+)', filename)
                season_num = int(s_match_file.group(1)) if s_match_file else 1
                
            if b_large in ["电视剧", "动漫", "短剧"] or season_num > 1:
                virtual_cloud_path = f"{virtual_cloud_path}/Season {season_num}"

            # ==========================================
            # 🌟 修复 3：完全物理阉割洗名！绝对保留所有原版压制参数
            # ==========================================
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
                "mode": strm_mode  # 🌟 修复 4：把模式参数完美传给 5000 端口！
            }

            try:
                res = requests.post(f"{API_5000_URL}/api/make_strm", json=payload, timeout=5)
                if res.status_code == 200:
                    
                    # 🌟 关键补丁：在局域网同步工具最终把 mtime 倒拨回去之后，重新抓取一次最真实的物理 mtime！
                    try:
                        final_mtime = os.stat(file_path).st_mtime
                    except:
                        final_mtime = mtime
                        
                    monitor_history[file_path] = {
                        "process_time": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                        "file_size": size,
                        "mtime": final_mtime, # 记录真正的最终同步时间
                        "target_strm_dir": target_local_dir,
                        "strm_name": strm_name,
                        "status": "success"
                    }
                    changed = True
                    updated_dirs.add(target_local_dir) 
                    
                    ledger_file = os.path.join(DB_DIR, "strm_ledger.json")
                    ledger = load_json(ledger_file)
                    ledger[final_name] = True
                    save_json(ledger_file, ledger)
                else:
                    logger.error(f"❌ [投递箱] API 拒绝: {res.text}")
            except Exception as e:
                logger.error(f"❌ [投递箱] 无法连接到 管家: {e}")

        except Exception as e:
            logger.error(f"❌ [投递箱] 解析异常: {file_path} - {e}")

    if changed:
        save_json(history_file, monitor_history)

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
    wizard_states = {} # 🧠 新增：记忆向导状态的“大脑”

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
           
        # 👇 新增：本地投递箱极速雷达
        if settings.get("auto_scan_local", True):
            scan_local_dropbox()
        
        # 🌟 新增：挂载 139 专属流水线
        process_139_pipeline()

        wait_min = random.randint(25, 45)
        logger.info(f"🛌 [系统] 航线巡逻结束。进入节电待机，距下次起飞还有 {wait_min} 分钟...")
        logger.info("==========================================")
        schedule.clear('patrol')
        schedule.every(wait_min).minutes.do(scheduled_task).tag('patrol')

    scheduled_task()
    schedule.every(6).hours.do(auto_relogin, client_obj)

    # ==========================================
    # 🌟 新增：跨月防断层，每日 23:30 强制保底收割
    # ==========================================
    def daily_force_harvest():
        logger.info("⏰ [定时任务] 触发 23:30 跨月保底强行收割...")
        notifier.send_message("⏰ 触发 23:30 每日保底收割，防止跨月断层...")
        
        # 无视设置开关，直接硬调用收割核心函数
        p_names = process_cas_via_olist_api()
        
        if p_names:
            msg_str = "\n".join([f" └ {n}" for n in p_names[:20]])
            if len(p_names) > 20: msg_str += f"\n...等共 {len(p_names)} 个文件"
            notifier.send_message(f"✅ 每日保底收割完成:\n{msg_str}")
        else:
            notifier.send_message("✅ 每日保底收割完成，投递箱/云端暂无新文件。")

    # 设定时间点
    schedule.every().day.at("23:30").do(daily_force_harvest)
    
    # ==========================================
    # 🌟 唯一新增：完全不干扰主循环的独立探针线程
    # ==========================================
    def cookie_monitor():
        last_check_time = 0
        while True:
            if not os.path.exists(COOKIES_FILE):
                now = time.time()
                # 防爆锁：防止一分钟内被无限拉起
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
                    
                    # ==========================================
                    # 🔘 核心交互菜单处理 (回调)
                    # ==========================================
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
                            
                        # 第二步：选择完频率后，动态判断是否需要选【周几】！
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

                        # 🌟 新增中间层：记录你选的周几，并接着跳到分类选项
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

                        # 第三步：选择完分类后，展示过滤规则
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
                            
                        # 第四步（魔法拼接）：组装给 V4.8 解析引擎
                        elif data.startswith("wiz_type_"):
                            f_type = data.split("_")[2]
                            state = wizard_states.pop(chat_id)
                            
                            kw_map = {"视频": ".mp4 .mkv .ts", "CAS": ".cas", "全盘": ""}
                            kw = kw_map.get(f_type, "")
                            
                            # 🌟 核心魔法：把刚才选的周几拼接到指令里！
                            day_tag = f" #{state['day']}" if "day" in state else ""
                            s_cmd = f"订阅{state['s_num']}" 
                            
                            # 最终组装的命令形态：订阅 绝命毒师 https... #周更 #周三 #美剧 mp4
                            cmd = f"{s_cmd} {state['title']} {state['url']} #{state['freq']}{day_tag} #{state['cat']} {kw}".strip()
                            
                            notifier.edit_message(msg_id, f"🎉 <b>向导收集完毕！</b>\n正在为您下发指令:\n<code>{cmd}</code>")
                            
                            # 把指令原样注入消息队列，喂给底下的原始逻辑
                            item['message'] = {'chat': {'id': chat_id}, 'text': cmd}
                        
                        # 🌟 拦截从“广场”点过来的订阅按钮
                        elif data.startswith("wiz_feed_"):
                            url_code = data.replace("wiz_feed_", "")
                            full_url = f"https://cloud.189.cn/t/{url_code}" # 拼装成原生的官方分享链接
                            
                            # 强行激活现有的第一步向导，把组装好的链接塞进去
                            wizard_states[chat_id] = {"step": 1, "url": full_url}
                            kb = {"inline_keyboard": [[{"text": "❌ 取消", "callback_data": "wiz_cancel"}]]}
                            notifier.edit_message(msg_id, f"🔗 <b>已从内部广场锁定直链！</b>\n\n✏️ 请直接回复本条消息，输入【干净剧名(年份)】\n<i>(如带季数，请直接写: 庆余年 2)</i>", kb)
                    # ==========================================
                    # 📝 恢复原有的文字处理逻辑
                    # ==========================================
                    msg = item.get('message', {})
                    text = msg.get('text', '')
                    chat_id = msg.get('chat', {}).get('id')

                    if str(chat_id) == str(TG_ADMIN_USER_ID):
                        text = text.strip()
                        
                        # ====== 🥇 新增：纯收割专用建库指令（纯净版，绝不污染全局） ======
                        if text.startswith("加库") or text.startswith("/hsub"):
                            try:
                                if text.startswith("加库"):
                                    clean_text = text[2:].strip()
                                else:
                                    clean_text = text[5:].strip()
                                    
                                parts = clean_text.split()
                                if len(parts) < 2:
                                    notifier.send_message("格式错误！\n常规：加库 剧名(年份) 分类\n指定老月：加库 剧名 分类 202604")
                                    continue
                                
                                # 直接使用顶部引入的全局 re 和 datetime，绝不在此局部 import
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

                                # 直接使用顶部定义的全局函数和全局变量
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
                            continue # 核心拦截器：干完活直接掐断
                        
                        # ==========================================
                        # 🌟 189 管家 STRM 生成模式切换指令 (A/B/C 极简版)
                        # ==========================================
                        elif text.startswith("设置模式") or text.startswith("/mode"):
                            mode_input = text.replace("设置模式", "").replace("/mode", "").strip().upper()
                            mode_map = {"A": "cas", "B": "cas_native", "C": "both"}
                            
                            if mode_input not in mode_map:
                                notifier.send_message("❌ 模式错误！\n可选模式:\nA - 老版 (cas)\nB - 原生直连 (cas_native)\nC - 双开 (both)\n示例: /mode A")
                                continue
                                
                            real_mode = mode_map[mode_input]
                            s = load_json(SETTINGS_FILE)
                            s["189_strm_mode"] = real_mode
                            save_json(SETTINGS_FILE, s)
                            notifier.send_message(f"✅ 成功！以后 189 投递管家将使用模式: {mode_input} ({real_mode})")
                            continue

                        # ====== 🗑️ 新增：删除收割库记录指令 ======
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
                                
                                # 1. 先尝试精确匹配
                                if search_key and search_key in h_data:
                                    deleted_items.append((search_key, h_data[search_key]))
                                    del h_data[search_key]
                                else:
                                    # 2. 如果没精确对上，启动模糊匹配兜底（只要名字里包含就干掉）
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

                        # ====== 📋 新增：查看收割库清单指令 ======
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

                        # ====== 🕵️‍♂️ 新增：动态巡逻目录管理指令 ======
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
                            # 智能匹配：如果你发的是纯数字，按序号删；如果发的是文字，模糊匹配删
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
                            
                        # ==========================================
                        # 🌟 新增：动态免删复制(个人云)目录管理指令
                        # ==========================================
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

                        # ==========================================
                        # 🌟 139移动云盘 专属 STRM 生成指令 (全版本寻轨 + Emby精准延时刷新)
                        # ==========================================
                        elif text.startswith("同步139") or text.startswith("/sync139"):
                            keyword_input = text.replace("同步139", "").replace("/sync139", "").strip()
                            if not keyword_input:
                                notifier.send_message("❌ 格式错误！\n示例：同步139 天才\n(也支持直接发送绝对路径)")
                                continue
                            
                            target_paths = []
                            
                            # 🧠 1. 判断是直接输入的绝对路径还是模糊关键词
                            if keyword_input.startswith("/"):
                                target_paths.append(keyword_input)
                            else:
                                notifier.send_message(f"🔍 启动 139 全域雷达，正在搜寻所有【{keyword_input}】版本...")
                                try:
                                    r_log = requests.post(f"{API_5255_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                    if r_log.get("code") == 200:
                                        h_139 = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                        
                                        # 读取 /139/139cas 下的所有大类目录
                                        r_cats = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": DIR_139_TARGET}, headers=h_139, timeout=5).json()
                                        cats = [c["name"] for c in (r_cats.get("data") or {}).get("content", []) if c["is_dir"]]
                                        
                                        for cat in cats:
                                            cat_path = f"{DIR_139_TARGET}/{cat}"
                                            r_shows = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": cat_path}, headers=h_139, timeout=5).json()
                                            shows = (r_shows.get("data") or {}).get("content", [])
                                            
                                            for s in shows:
                                                # 🌟 去除 break：只要包含关键词，全部录入（通吃不同压制版本、杜比视界版等）
                                                if s["is_dir"] and keyword_input.lower() in s["name"].lower():
                                                    show_path = f"{cat_path}/{s['name']}"
                                                    
                                                    # 探测是否存在 Season 目录
                                                    r_seasons = requests.post(f"{API_5255_URL}/api/fs/list", json={"path": show_path}, headers=h_139, timeout=5).json()
                                                    seasons = [ss["name"] for ss in (r_seasons.get("data") or {}).get("content", []) if ss["is_dir"] and "season" in ss["name"].lower()]
                                                    
                                                    if seasons:
                                                        for s_name in seasons:
                                                            target_paths.append(f"{show_path}/{s_name}")
                                                    else:
                                                        target_paths.append(show_path)
                                                        
                                        if not target_paths:
                                            notifier.send_message(f"📭 雷达遍历完毕，未找到与【{keyword_input}】相关的文件夹。")
                                            continue
                                except Exception as e:
                                    notifier.send_message(f"⚠️ 139 智能寻轨网络异常: {e}")
                                    continue

                            # 🌟 2. 批量派发给 5000 管家造物，并收集对应的本地 STRM 物理路径
                            s = load_json(SETTINGS_FILE)
                            local_139_strm_dir = s.get("local_139_strm_dir", DEFAULT_139_LOCAL_STRM)
                            refresh_dirs = set()
                            
                            for tp in target_paths:
                                try:
                                    # 唤醒管家生成 STRM
                                    requests.get(f"{API_5000_URL}/api/sync?drive=139&path={parse.quote(tp)}", timeout=5).close()
                                    
                                    # 推算对应的本地 STRM 目录
                                    strm_sub_dir = tp.replace(DIR_139_TARGET, "").strip("/")
                                    real_strm_path = os.path.join(local_139_strm_dir, strm_sub_dir).replace("\\", "/")
                                    refresh_dirs.add(real_strm_path)
                                except Exception as e:
                                    logger.error(f"❌ [139手动同步] 下发异常 ({tp}): {e}")

                            # 汇报命中的所有版本路径
                            msg_list = "\n".join([f" └ 📁 <code>{p}</code>" for p in target_paths])
                            notifier.send_message(f"🎯 成功锁定 {len(target_paths)} 个版本目标，已下发生成：\n{msg_list}")

                            # 🌟 3. 异步延时精准唤醒 Emby 刷新 (给 5000 端口 5 秒写盘时间)
                            if refresh_dirs:
                                bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                
                                def delayed_manual_139_refresh(dirs_to_refresh):
                                    time.sleep(5.0)
                                    for r_dir in dirs_to_refresh:
                                        try:
                                            subprocess.Popen([bash_path, refresh_sh, r_dir], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                            logger.info(f"⚡ [139手动同步] 已精准唤醒 Emby 局部刷新: {r_dir}")
                                        except Exception as err:
                                            logger.error(f"❌ [139手动同步] 唤醒刷新失败: {err}")
                                        time.sleep(1.0)
                                        
                                threading.Thread(target=delayed_manual_139_refresh, args=(refresh_dirs,)).start()
                                
                            continue  

                        # 🌟 新增的动态开关与拉取指令
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
                            notifier.send_message("📥 收到【收割】指令：正在为您洗名并入库 CAS 文件...")
                            p_names = process_cas_via_olist_api()
                            if p_names:
                                msg_str = "\n".join([f" └ {n}" for n in p_names[:20]])
                                if len(p_names) > 20: msg_str += f"\n...等共 {len(p_names)} 个文件"
                                notifier.send_message(f"✅ CAS 收割入库完成:\n{msg_str}")
                            else:
                                notifier.send_message("✅ CAS 收割完成，暂无新文件。")
                            continue
                        # ====== 🌟 新增：139 专属手动触发测试指令 ======
                        elif text in ["139收割", "处理139", "扫139", "/139"]:
                            notifier.send_message("🚀 收到指令：正在为您启动 139 专属加工流水线...")
                            try:
                                process_139_pipeline()
                                notifier.send_message("✅ 139 加工流水线本次作业执行完毕！")
                            except Exception as e:
                                notifier.send_message(f"❌ 139 流水线执行报错: {e}")
                            continue
                        elif text in ["扫箱子", "投递", "本地扫描", "/dropbox"]:
                            notifier.send_message("🚀 收到指令：立刻启动本地投递箱极速雷达...")
                            try:
                                scan_local_dropbox()
                                notifier.send_message("✅ 本地投递箱雷达扫描与下发任务已执行完毕！")
                            except Exception as e:
                                notifier.send_message(f"❌ 扫描本地投递箱时发生异常: {e}")
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
                                    # 🚨 掉线报警，抛出异常让底层接住
                                    raise Exception(f"接口掉线/风控拦截: {res}")
                            except Exception as e:
                                notifier.send_message(f"❌ 广场拉取异常: {e}")
                                err_str = str(e).upper()
                                # 👇 底层接住异常，启动自愈打针
                                if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                    if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                    auto_relogin(client_obj, force=True)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
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
                                        # 🚨 掉线报警 1
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
                                            # 🚨 掉线报警 2
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
                                    
                                    msg_lines.append(f"{i}. 📁 <code>{name}</code>\n   └ 👤 {author} | ⏱ {date_str}\n")
                                    btn_text = name.replace("📺 ", "").replace("🎬 ", "")
                                    kb_buttons.append([{"text": f"📥 订阅: {btn_text[:12]}...", "callback_data": f"wiz_feed_{url_code}"}])
                                    
                                kb_buttons.append([{"text": "❌ 取消", "callback_data": "wiz_cancel"}])
                                kb = {"inline_keyboard": kb_buttons}
                                
                                notifier.send_message("\n".join(msg_lines), kb)
                            except Exception as e:
                                notifier.send_message(f"❌ 搜索拉取异常: {e}")
                                # 👇 自愈打针
                                err_str = str(e).upper()
                                if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                    if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                    auto_relogin(client_obj, force=True)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
                            continue
                        
                        # ====== 📖 新增：TMDB 影视资料百科指令 ======
                        elif text.startswith("查剧 ") or text.startswith("信息 ") or text.startswith("/info "):
                            keyword = text.split(" ", 1)[1].strip()
                            if not keyword: continue
                            
                            notifier.send_message(f"🔍 正在连接 TMDB 全球数据库，检索【{keyword}】的档案...")
                            try:
                                info_msg = fetch_tmdb_rich_info(keyword)
                                # 配合一个快速订阅的建议按钮
                                kb = {"inline_keyboard": [[{"text": "❌ 关闭面板", "callback_data": "wiz_cancel"}]]}
                                notifier.send_message(info_msg, kb)
                            except Exception as e:
                                notifier.send_message(f"❌ 检索失败: {e}")
                            continue

                        elif text.startswith("查 ") or text.startswith("查看 ") or text.startswith("/check "):
                            raw_keyword = text.split(" ", 1)[1].strip()
                            if not raw_keyword: continue
                            
                            keyword_list = [k.strip().lower() for k in raw_keyword.split() if k.strip()]
                            display_kw = " | ".join(keyword_list)
                            
                            notifier.send_message(f"🔍 启动地毯式穿甲雷达，正在遍历所有大佬个人主页挖掘: 【{display_kw}】...")
                            try:
                                # 🌟 第一步：获取活跃大佬的名单
                                res_feed = client_obj.session.get("https://cloud.189.cn/api/open/share/getOwnerSubscribeShare.action?pageNum=1&pageSize=100", timeout=15).json()
                                active_users = {} 
                                
                                if res_feed.get("code") == "success":
                                    for item in res_feed.get("data", {}).get("shareFileList", []):
                                        uid = item.get("upUserId")
                                        name = item.get("ownerAccount", "未知大佬")
                                        if uid: active_users[uid] = name
                                
                                if not active_users:
                                    notifier.send_message("❌ 广场空空如也，未获取到任何订阅大佬的信息。")
                                    continue
                                
                                # 🌟 第二步：踹门深挖
                                matched_items = []
                                for uid, uname in active_users.items():
                                    for page in range(1, 4):
                                        url = f"https://cloud.189.cn/api/open/share/getUpResourceShare.action?pageNum={page}&pageSize=30&upUserId={uid}"
                                        res_user = client_obj.session.get(url, timeout=10).json()
                                        
                                        if res_user.get("code") == "success":
                                            # 🎯 破案核心：这里是 fileList！不是 shareFileList！
                                            items = res_user.get("data", {}).get("fileList", []) 
                                            if not items: break 
                                            
                                            for item in items:
                                                item_name_lower = item.get("name", "").lower()
                                                if any(kw in item_name_lower for kw in keyword_list):
                                                    item["ownerAccount"] = uname 
                                                    matched_items.append(item)
                                        else:
                                            break
                                            
                                # 🌟 第三步：去重展示
                                unique_matches = []
                                seen_urls = set()
                                for item in matched_items:
                                    url_code = item.get("accessURL", "")
                                    if url_code and url_code not in seen_urls:
                                        seen_urls.add(url_code)
                                        unique_matches.append(item)

                                if not unique_matches:
                                    notifier.send_message(f"📭 翻遍了 {len(active_users)} 位大佬的个人历史主页，没找到与【{display_kw}】相关的资源。")
                                    continue
                                    
                                msg_lines = [f"🎯 <b>为您精准捞到了 {len(unique_matches)} 个相关资源：</b>\n"]
                                kb_buttons = []
                                
                                for i, item in enumerate(unique_matches[:8], 1):
                                    name = item.get("name", "未知资源")
                                    author = item.get("ownerAccount", "未知发布者")
                                    url_code = item.get("accessURL", "")
                                    date_str = item.get("lastOpTime", "")[5:16]
                                    
                                    msg_lines.append(f"{i}. 📁 <code>{name}</code>\n   └ 👤 {author} | ⏱ {date_str}\n")
                                    kb_buttons.append([{"text": f"📥 订阅: {name[:15]}...", "callback_data": f"wiz_feed_{url_code}"}])
                                    
                                kb_buttons.append([{"text": "❌ 取消", "callback_data": "wiz_cancel"}])
                                kb = {"inline_keyboard": kb_buttons}
                                
                                notifier.send_message("\n".join(msg_lines), kb)
                            except Exception as e:
                                notifier.send_message(f"❌ 搜索拉取异常: {e}")
                            continue

                        # 🌟 第二处：查人/查作者
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
                                    # 🚨 掉线报警 1
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
                                            # 🚨 掉线报警 2
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
                                # 👇 自愈打针
                                err_str = str(e).upper()
                                if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                    if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                    auto_relogin(client_obj, force=True)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
                            continue

                        # ==========================================
                        # 🧙‍♂️ 智能嗅探向导拦截器
                        # ==========================================
                        if re.match(r'^http[s]?://(cloud\.189\.cn|t\.189\.cn)/', text):
                            wizard_states[chat_id] = {"step": 1, "url": text}
                            kb = {"inline_keyboard": [[{"text": "❌ 取消", "callback_data": "wiz_cancel"}]]}
                            notifier.send_message(f"🔗 <b>嗅探到天翼云链接！</b>\n\n✏️ 请直接回复本条消息，输入【剧名(年份)】\n<i>(如带季数，请直接写: 庆余年 2)</i>", kb)
                            continue
                            
                        if chat_id in wizard_states and wizard_states[chat_id].get("step") == 1 and not text.startswith("/"):
                            m = re.match(r'^(.*?)\s+[sS第]?0?(\d{1,2})[季]?$', text)
                            if m:
                                wizard_states[chat_id]["title"] = m.group(1).strip()
                                wizard_states[chat_id]["s_num"] = m.group(2)
                            else:
                                wizard_states[chat_id]["title"] = text.strip()
                                wizard_states[chat_id]["s_num"] = ""

                            wizard_states[chat_id]["step"] = 2
                            kb = {"inline_keyboard": [
                                [{"text": "🌞 日更 (国产剧/短剧等)", "callback_data": "wiz_freq_日更"}],
                                [{"text": "📅 周更/连载 (美剧/动漫等)", "callback_data": "wiz_freq_周更"}],
                                [{"text": "✅ 已完结 (全集一波流)", "callback_data": "wiz_freq_完结"}],
                                [{"text": "🎬 单次任务 (电影/演唱会等)", "callback_data": "wiz_freq_单次"}], # 🌟 改为单次
                                [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                            ]}
                            
                            s_tip = f" (第 {wizard_states[chat_id]['s_num']} 季)" if wizard_states[chat_id]["s_num"] else ""
                            notifier.send_message(f"✅ 已记录剧名: {wizard_states[chat_id]['title']}{s_tip}\n\n<b>🏷️ 请选择【更新频率】:</b>", kb)
                            continue
                            
                        # === 原版指令识别开始 ===
                        logger.info(f"🛠️ [指令] 接收到远程终端最高权限指令: {text}")
                        # 🌟 净水器：一刀切掉 TG 菜单自带的尾巴
                        if text and "@xushangjun_bot" in text:
                            text = text.replace("@xushangjun_bot", "")

                        if text == "列表" or text == "清单" or text == "/list":
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
                        
                        # ====== 🚀 终极抢救：全自动分块重建 (防 Termux 崩溃版) ======
                        elif text.startswith("全库重建") or text.startswith("/reall"):
                            notifier.send_message("🚨 收到全库重建指令！为防 Termux 内存爆炸，引擎已启动【切片下发模式】...")
                            try:
                                r_log = requests.post(f"{API_5244_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                if r_log.get("code") == 200:
                                    o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                    
                                    # 提取所有分类基点
                                    radar_bases = set()
                                    for l_cat, s_cat in CAT_ROUTER.values():
                                        sub_p = f"{l_cat}/{s_cat}".strip('/') if s_cat else l_cat
                                        radar_bases.add(get_openlist_path(f"{DIR_CAS_ROOT}/{sub_p}".replace("//", "/")))
                                    
                                    matched_paths = []
                                    # 钻透目录，细化到单部剧的层级
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
                                        
                                        # 扔进后台线程跑，防止阻塞 TG 机器人主循环
                                        def rebuild_task():
                                            for i, mp in enumerate(matched_paths, 1):
                                                # 穿透云端缓存
                                                requests.post(f"{API_5244_URL}/api/fs/list", json={"path": mp, "refresh": True}, headers=o_headers, timeout=10).close()
                                                time.sleep(1.0)
                                                try: 
                                                    requests.get(f"{API_5000_URL}/api/sync", params={"path": mp}, timeout=3).close()
                                                    logger.info(f"[{i}/{len(matched_paths)}] ✅ 成功下发: {mp}")
                                                except: pass
                                                # ⏳ 核心：给 5000 端口 2 秒缓冲时间，写完盘释放内存！
                                                time.sleep(2.0) 
                                            notifier.send_message(f"🎉 189 全库 STRM 重建完毕！")
                                            
                                        threading.Thread(target=rebuild_task).start()
                            except Exception as e:
                                notifier.send_message(f"❌ 全库重建异常: {e}")
                            continue

                        elif text.startswith("刷新") or text.startswith("入库"):
                            match_refresh = re.match(r'^(刷新|入库)\s+(.*)', text)
                            if match_refresh:
                                keyword_input = match_refresh.group(2).strip()
                                # 🌟 修复正则：限制季数为1-2位，并要求前方必须有空格或 S/第 等标识，保护带年份的单剧
                                m = re.match(r'^(.*?)\s+(?:[sS第]\s*)?0?(\d{1,2})[季]?$', keyword_input, re.IGNORECASE)
                                if m and m.group(1).strip(): 
                                    base_kw, s_num = m.group(1).strip(), int(m.group(2))
                                else: 
                                    base_kw, s_num = keyword_input, None

                                notifier.send_message(f"🔍 收到入库指令，正在检索: {base_kw}...")
                                
                                subs = load_json(SUBS_FILE)
                                matched_paths = []
                                
                                # 👇 新增：如果输入的是带斜杠的绝对路径，直接无脑锁定，不查库！
                                # =================================================================
                                # 🌟 终极智能寻轨雷达替换开始：彻底解放人脑，全自动找回带年份的绝对路径
                                # =================================================================
                                if base_kw.startswith("/"):
                                    matched_paths.append(base_kw.strip())
                                else:
                                    # 1. 优先极速比对本地活跃记忆库
                                    for t_id, info in subs.items():
                                        path_in_db = info.get("path", "") if isinstance(info, dict) else ""
                                        if base_kw.lower() in path_in_db.lower():
                                            if path_in_db not in matched_paths: matched_paths.append(path_in_db)

                                    # 2. 🌟 终极雷达：如果大脑没记住这剧，立刻启动底层 Openlist 接口全自动搜寻物理路径！
                                    if not matched_paths:
                                        notifier.send_message(f"🧠 记忆库未收录老剧，正启动 Openlist 穿透雷达自动检索真实路径: [{base_kw}] ...")
                                        try:
                                            r_log = requests.post(f"{API_5244_URL}/api/auth/login", 
                                                                  json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                            if r_log.get("code") == 200:
                                                o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                                
                                                # 提取所有分类的总分区路径作为雷达扫描基点
                                                radar_bases = set()
                                                for l_cat, s_cat in CAT_ROUTER.values():
                                                    sub_p = f"{l_cat}/{s_cat}".strip('/') if s_cat else l_cat
                                                    radar_bases.add(get_openlist_path(f"{DIR_CAS_ROOT}/{sub_p}".replace("//", "/")))
                                                    radar_bases.add(get_openlist_path(f"{DIR_VIDEO_ROOT}/{sub_p}".replace("//", "/")))
                                                
                                                for base_p in radar_bases:
                                                    # 极速读取 Openlist 年月目录列表 (利用接口轻量级穿透)
                                                    r_list = requests.post(f"{API_5244_URL}/api/fs/list", 
                                                                           json={"path": base_p}, headers=o_headers, timeout=5).json()
                                                    if r_list.get("code") == 200:
                                                        content = (r_list.get("data") or {}).get("content") or []
                                                        ym_dirs = [item["name"] for item in content if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                                                        ym_dirs.sort(reverse=True) # 优先从最新的月份往老月份找
                                                        
                                                        for ym in ym_dirs:
                                                            ym_path = f"{base_p}/{ym}"
                                                            r_shows = requests.post(f"{API_5244_URL}/api/fs/list", 
                                                                                    json={"path": ym_path}, headers=o_headers, timeout=5).json()
                                                            if r_shows.get("code") == 200:
                                                                shows = (r_shows.get("data") or {}).get("content") or []
                                                                for s_item in shows:
                                                                    # 只要文件夹名字包含输入的关键词，当场锁定！
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
                                # =================================================================
                                # 🌟 终极智能寻轨雷达替换结束
                                # =================================================================
                                            
                                # =================================================================
                                # 🌟 终极双轨驱动装甲替换开始 (涵盖单点刷新与全局扫描)
                                # =================================================================
                                if matched_paths:
                                    notifier.send_message(f"🎯 共命中 {len(matched_paths)} 个关联目录，执行双轨刷新...")
                                    
                                    # 优化：提到循环外登录 Openlist 一次，拿到通用 Token，大幅提升效率
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
                                        
                                        # 💥 第一步：无论 CAS 还是普通视频，必须先强行穿透 Openlist 缓存！
                                        # 只有 Openlist 物理层重载了，它自身才能扫描到新文件并映射出普通视频的 STRM！
                                        if o_headers:
                                            notifier.send_message(f"⚡ 强制穿透 Openlist 缓存: {butler_path} ...")
                                            try:
                                                requests.post(f"{API_5244_URL}/api/fs/list", 
                                                              json={"path": butler_path, "refresh": True}, headers=o_headers, timeout=15).close()
                                                time.sleep(3.0) # 给底层留出落盘和 Openlist 自建 strm 的缓冲时间
                                            except: pass

                                        # 🔀 第二步：严格的双轨路由分流
                                        if DIR_CAS_ROOT in openlist_p:
                                            # 177-秒传 -> 交给 管家 收割
                                            try: 
                                                requests.get(f"{API_5000_URL}/api/sync", params={"path": butler_path}, timeout=3).close()
                                                notifier.send_message(f"✅ 管家同步指令已精准下发: {butler_path}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            # 177-视频与原老目录 -> Openlist 自身已吐出 STRM，直接唤醒 Emby 原生拉取
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
                                    # 核心修正：只从具体订阅库中精确组装有效的“剧集父目录”层级
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
                                        # 💥 无论 CAS 还是普通视频，全域强制穿透 Openlist 缓存
                                        if o_headers:
                                            try:
                                                requests.post(f"{API_5244_URL}/api/fs/list", 
                                                              json={"path": rp, "refresh": True}, headers=o_headers, timeout=10).close()
                                                time.sleep(1.5) # 批量扫描稍微给点延时错峰即可
                                            except: pass

                                        # 🔀 严格的双轨路由分流
                                        if DIR_CAS_ROOT in rp:
                                            try: requests.get(f"{API_5000_URL}/api/sync", params={"path": rp}, timeout=3).close()
                                            except: pass
                                        else:
                                            try:
                                                s = load_json(SETTINGS_FILE)
                                                bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                                                refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                                                subprocess.Popen([bash_path, refresh_sh, rp], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                            except: pass
                                        
                                    notifier.send_message(f"✅ 全区最高指令已下发！安全触发并双轨刷新了 {len(scan_roots)} 个雷达基点。")
                                # =================================================================
                                # 🌟 终极双轨驱动装甲替换结束
                                # =================================================================
                        # ==========================================
                        # 🚑 独立专属指令：【恢复云端】 (严格对比、缺一补一、死等管家完成)
                        # ==========================================
                        elif text.startswith("恢复云端") or text.startswith("/recloud"):
                            base_kw = text.replace("恢复云端", "").strip()
                            if not base_kw:
                                notifier.send_message("❌ 请输入目标，例如：\n恢复云端 华语剧\n恢复云端 /177/177-秒传/电视剧")
                                continue

                            notifier.send_message(f"🚑 收到指令！目标: [{base_kw}]\n🛡️ 【精准对账+30秒生成】已开启：逐个文件严格对比，少一集补一集，管家彻30秒生成完毕再进入下一部！")
                            
                            try:
                                r_log = requests.post(f"{API_5244_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=5).json()
                                if r_log.get("code") != 200:
                                    notifier.send_message("❌ OpenList 登录失败。")
                                    continue
                                o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                
                                scan_bases = []
                                # 1. 支持绝对路径 (哪怕输错层级也能救)
                                if base_kw.startswith("/"):
                                    scan_bases.append(get_openlist_path(base_kw))
                                # 2. 支持字典里的精确小分类，如 华语剧
                                elif base_kw in CAT_ROUTER:
                                    l_cat, s_cat = CAT_ROUTER[base_kw]
                                    scan_bases.append(get_openlist_path(f"{DIR_CAS_ROOT}/{l_cat}/{s_cat}".strip('/').replace("//", "/")))
                                # 3. 如果是大词(如"电视剧")，把底下所有分类目录全包进去
                                else:
                                    for l_cat, s_cat in CAT_ROUTER.values():
                                        sub_p = f"{l_cat}/{s_cat}".strip('/') if s_cat else l_cat
                                        if base_kw in sub_p:
                                            scan_bases.append(get_openlist_path(f"{DIR_CAS_ROOT}/{sub_p}".replace("//", "/")))
                                    # 如果什么都没匹配上，强制扫描整个大库
                                    if not scan_bases:
                                        scan_bases.append(get_openlist_path(DIR_CAS_ROOT))

                                # 去重
                                scan_bases = list(set(scan_bases))

                                def do_segmented_recovery():
                                    s = load_json(SETTINGS_FILE)
                                    local_strm_dir = s.get("local_strm_dir", DEFAULT_LOCAL_STRM)
                                    total_rebuild = 0
                                    
                                    # 🌟 终极杀招：无限递归扫描！不管你在哪一层，一律钻到底找 .cas！
                                    def scan_for_shows(current_path):
                                        nonlocal total_rebuild
                                        try:
                                            res = requests.post(f"{API_5244_URL}/api/fs/list", json={"path": current_path, "refresh": True}, headers=o_headers, timeout=10).json()
                                            if res.get("code") != 200: return
                                        except: return
                                        
                                        items = (res.get("data") or {}).get("content", [])
                                        
                                        # 检查当前目录下有没有 .cas 文件
                                        has_cas = False
                                        for item in items:
                                            if not item["is_dir"] and item["name"].lower().endswith(".cas"):
                                                has_cas = True
                                                break
                                                
                                        if has_cas:
                                            # 这证明已经到达最底层的剧集目录！查岗本地 STRM！
                                            strm_sub_dir = current_path.split(DIR_CAS_ROOT)[-1].strip("/")
                                            target_local_dir = os.path.join(local_strm_dir, strm_sub_dir).replace("\\", "/")
                                            
                                            need_rebuild = False
                                            
                                            # 场景1：如果本地连这个文件夹都没有，直接判定为缺漏
                                            if not os.path.exists(target_local_dir):
                                                need_rebuild = True
                                            else:
                                                import re
                                                local_strms = [f.lower() for f in os.listdir(target_local_dir) if f.endswith(".strm")]
                                                
                                                for item in items:
                                                    if not item["is_dir"] and item["name"].lower().endswith(".cas"):
                                                        cloud_raw = item["name"][:-4].lower()
                                                        # 将云端名字拆解成纯词块和数字（如 's01e01', '1080p', '神雕侠侣'）
                                                        cloud_keys = set(re.findall(r'[a-z0-9\u4e00-\u9fa5]+', cloud_raw))
                                                        
                                                        is_matched = False
                                                        for loc in local_strms:
                                                            loc_keys = set(re.findall(r'[a-z0-9\u4e00-\u9fa5]+', loc[:-5]))
                                                            # 🎯 核心绝杀：只要本地名字里包含的核心词块（含集数），能100%在云端名字里找到，说明就是它！
                                                            # 这完美兼容了你过滤冗余标签的洗名逻辑，绝不误判！
                                                            if loc_keys and loc_keys.issubset(cloud_keys):
                                                                is_matched = True
                                                                break
                                                                
                                                        if not is_matched:
                                                            need_rebuild = True
                                                            break # 只要发现哪怕一集在本地匹配不上，立刻叫管家！
                                            
                                            # 如果发现空洞，喂给 5000 端口
                                            if need_rebuild:
                                                logger.warning(f"🚑 发现真实缺漏文件，立即交由管家补齐: {current_path}")
                                                try:
                                                    requests.get(f"{API_5000_URL}/api/sync", params={"path": current_path}, timeout=5)
                                                except Exception as e:
                                                    pass
                                                total_rebuild += 1
                                                
                                                # 保留你聪明修改的 15 秒缓冲，极其稳定！
                                                logger.info(f"⏳ 缓冲防爆：等待管家落盘，休眠 15 秒...")
                                                time.sleep(15.0)
                                                
                                            return # 搜到 .cas 就证明到底了，不用再往这层往下搜了
                                            
                                        # 如果这层没有 .cas，说明还没到底，继续往子文件夹里钻！
                                        for item in items:
                                            if item["is_dir"]:
                                                scan_for_shows(f"{current_path}/{item['name']}")

                                    for base_p in scan_bases:
                                        notifier.send_message(f"🔎 启动极限穿透扫描: {base_p}")
                                        scan_for_shows(base_p)
                                        
                                    notifier.send_message(f"🎉 扫描结束！本次总计精准抢救并补全了 {total_rebuild} 部存在缺漏的剧集！")

                                threading.Thread(target=do_segmented_recovery).start()
                            except Exception as e:
                                notifier.send_message(f"❌ 恢复指令异常: {e}")
                            continue

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
                                                requests.get(f"{API_5000_URL}/api/sync", params={"path": openlist_target_path}, timeout=3).close()
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
                                                        requests.get(f"{API_5000_URL}/api/sync", params={"path": openlist_target_path}, timeout=3).close()
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

                        # 🌟 修复：修正了 or 的语法错误，完美识别订阅指令
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
                                elif "#电影" in text: freq_tag, text = "单次", text.replace("#电影", "").strip() # 🌟 加入电影快捷标签映射
                                
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
                                
                                # 🚨 核心修复：智能抓取中文提取码，防止变成瞎子
                                pwd_match = re.search(r'(?:访问码|提取码|密码)\s*[:：]\s*([a-zA-Z0-9]{4})', text)
                                extracted_pwd = pwd_match.group(1) if pwd_match else None
                                # 把密码从文本里剔除，防止污染过滤关键词
                                text = re.sub(r'[\(（]?\s*(?:访问码|提取码|密码)\s*[:：]\s*[a-zA-Z0-9]{4}\s*[\)）]?', '', text)
                                
                                parts = text.split()
                                share_url, keyword, target_path = "", "", ""

                                url_index = -1
                                for i, p in enumerate(parts):
                                    if p.startswith("http"):
                                        share_url, url_index = p, i
                                        break

                                if url_index != -1:
                                    # 把抓到的密码强行焊死在网址屁股后面
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
                                        
                                        # ==========================================
                                        # 🛡️ V7 真·究极形态：记忆体(DB) -> 物理雷达(云端) -> 新建(保底)
                                        # ==========================================
                                        subs_cache = load_json(SUBS_FILE)
                                        existing_path = None
                                        
                                        # 🌟 修改：废弃激进正则，保留版本号/4K等特征
                                        def get_pure(text): return text.replace(" ", "").lower()
                                        show_name = clean_user_path.split('/')[0].strip()
                                        pure_show = get_pure(show_name)
                                        
                                        best_match_path = None
                                        
                                        ignore_words = {get_pure(DIR_CAS_ROOT), get_pure(DIR_VIDEO_ROOT), "season", "s"}
                                        for cat_key, (large_cat, sub_cat) in CAT_ROUTER.items():
                                            ignore_words.add(get_pure(cat_key))
                                            ignore_words.add(get_pure(large_cat))
                                            if sub_cat: ignore_words.add(get_pure(sub_cat))
                                        
                                        # 【第一阶段】：扫描大脑记忆库 (subscriptions.json)
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
                                                    
                                                    # 🌟 摒弃打分！严格要求版本特征一致
                                                    if pure_show == pure_f:
                                                        root_path = "/".join(db_folders[:idx+1])
                                                        best_match_path = root_path + clean_user_path[len(show_name):] if "/" in clean_user_path else root_path
                                                        break
                                                if best_match_path: break
                                        
                                        if best_match_path:
                                            existing_path = best_match_path
                                            type_msg = f"记忆库精确匹配沿用旧目录"
                                            
                                        # 【第二阶段】：如果大脑失忆了，启动天翼云物理雷达，扫描历史实体目录！
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
                                                                # 🌟 摒弃打分！
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
                                                
                                        # 【第三阶段】：彻底判定为全新资源，按当前系统年月打基建
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
# 🌟 139 外部下载脚本专属入口 (统筹生成与刷新)
# ==========================================
@trigger_app.route('/api/trigger_139', methods=['GET', 'POST'])
def trigger_139():
    target_dir = request.args.get('path')
    if not target_dir and request.is_json:
        target_dir = request.json.get('path')
        
    if target_dir:
        def handle_139_external():
            logger.info(f"⚡ [外部139投递] 收到闪电信号，目标: {target_dir}")
            
            # 1. 替下载脚本把原请求转发给 5000 端口管家去造物
            try: requests.get(f"{API_5000_URL}/api/sync?drive=139&path={parse.quote(target_dir)}", timeout=3).close()
            except: pass
            
            # 2. 算路并延迟唤醒 Emby (完美解耦)
            try:
                s = load_json(SETTINGS_FILE)
                bash_path = s.get("bash_path", DEFAULT_BASH_PATH)
                refresh_sh = s.get("refresh_sh_path", DEFAULT_REFRESH_SH)
                
                # 🌟 极简调用：从 settings.json 读取，读不到就直接用顶部的全局变量，绝不硬编码！
                local_139_strm_dir = s.get("local_139_strm_dir", DEFAULT_139_LOCAL_STRM)
                
                # 推算：切除云端根目录 /139/139cas，拼接本地绝对路径
                strm_sub_dir = target_dir.replace(DIR_139_TARGET, "").strip("/")
                real_strm_path = os.path.join(local_139_strm_dir, strm_sub_dir).replace("\\", "/")
                
                time.sleep(5.0) # 等 5000 端口写盘结束
                subprocess.Popen([bash_path, refresh_sh, real_strm_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                logger.info(f"⚡ [外部139投递] 已精准唤醒 Emby 刷新: {real_strm_path}")
            except Exception as e:
                logger.error(f"❌ [外部139投递] 唤醒 Emby 失败: {e}")
                
        threading.Thread(target=handle_139_external).start()
        return "✅ 139 外部投递信号已由总控接管", 200
        
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
        # 🌟 坚决使用顶部的变量，彻底消灭 500 报错！
        local_base = s.get("local_dropbox_dir", DEFAULT_LOCAL_DROPBOX)
        local_root = s.get("local_storage_root", DEFAULT_LOCAL_ROOT)
        
        if target_dir.startswith(local_base) or target_dir.startswith(local_root):
            logger.info(f"⚡ 收到【本地投递】闪电信号！目标: {target_dir}")
            # 🌟 扔进后台线程！瞬间回复外部脚本 200，绝不让外部脚本因为等待而报错超时！
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
    
    # 启动后台守护线程监听 5555
    threading.Thread(target=lambda: trigger_app.run(host='0.0.0.0', port=TRIGGER_PORT, debug=False, use_reloader=False), daemon=True).start()
    logger.info(f"🎧 本地监听端口 {TRIGGER_PORT} 已启动，随时等待下载脚本呼叫...")

    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    notifier.send_message(f"🤖 追剧转存引擎 (V7.8 智能路由版) 启动，获取凭证...")
    
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