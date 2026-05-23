---

title: "Termux之auto189.py、casplay.py"

author: "xxsky"

type: "posts"

date: 2026-04-22T19:57:30+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

安装自动转存与秒传播放脚本

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

3.auto189.py脚本

3.1在电脑中新建文件auto189.py内容如下：
```
import os
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
from pypinyin import pinyin, Style

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
            requests.post("http://127.0.0.1:5000/api/remote_log", 
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
        res_cn = requests.get(url_cn, timeout=5).json()
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
        res_tv = requests.get(f"https://api.themoviedb.org/3/tv/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN", timeout=3).json()
        if "name" in res_tv:
            cn_name = res_tv["name"]
            s_match = re.search(r'(?i)(S\d+|Season\s*\d+)', folder_name)
            s_tag = f" {s_match.group(1)}" if s_match else ""
            return f"📺 {cn_name}{s_tag} (TMDB-{tmdb_id})"
            
        # 查不到再按电影查
        res_movie = requests.get(f"https://api.themoviedb.org/3/movie/{tmdb_id}?api_key={TMDB_API_KEY}&language=zh-CN", timeout=3).json()
        if "title" in res_movie:
            return f"🎬 {res_movie['title']} (TMDB-{tmdb_id})"
    except: pass
    return folder_name

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
    "欧美剧": ("电视剧", "1-电视剧"), "美剧": ("电视剧", "1-电视剧"), "英剧": ("电视剧", "1-电视剧"),
    "日韩剧": ("电视剧", "2-电视剧"), "韩剧": ("电视剧", "2-电视剧"), "日剧": ("电视剧", "2-电视剧"),
    "华语电影": ("电影", "0-电影"), "国语电影": ("电影", "0-电影"),
    "欧美电影": ("电影", "1-电影"), "大片": ("电影", "1-电影"),
    "日韩电影": ("电影", "2-电影"),
    "国漫": ("动漫", "0-动漫"), 
    "日漫": ("动漫", "1-动漫"), "番剧": ("动漫", "1-动漫"),
    "综艺": ("综艺", ""), "纪录片": ("纪录片", ""), "演唱会": ("演唱会", ""), "短剧": ("短剧", "")
}

def load_json(filepath):
    if os.path.exists(filepath):
        with open(filepath, 'r', encoding='utf-8') as f:
            return json.load(f)
    return {}

def save_json(filepath, data):
    with open(filepath, 'w', encoding='utf-8') as f:
        json.dump(data, f, ensure_ascii=False, indent=2)

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
    valid_exts = ('.mp4', '.mkv', '.ts', '.avi', '.rmvb', '.flv', '.wmv', '.cas', '.srt', '.ass')
    _, ext = os.path.splitext(original_filename)
    if ext.lower() not in valid_exts: return None
    
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
    clean_show_name = re.sub(r'\(\d{4}\)', '', clean_show_name)
    clean_show_name = re.sub(r'(?i)\b(DV|4K|1080p|720p|2160p|WEB-DL|HDR|SDR|H265|x265|BluRay|Remux)\b', '', clean_show_name)
    clean_show_name = re.sub(r'[-_\s]+$', '', clean_show_name).strip()
    
    _, ext = os.path.splitext(original_filename)
    if not ext or len(ext) > 5: ext = ".mp4"

    tags = []
    if re.search(r'(?i)\bSDR\b', original_filename): tags.append("SDR")
    if re.search(r'(?i)\bHDR\b', original_filename): tags.append("HDR")
    if re.search(r'(?i)\bHQ\b', original_filename): tags.append("HQ")
    if re.search(r'(?i)\bDV\b', original_filename): tags.append("DV")
    tag_str = "." + ".".join(tags) if tags else ""

    # 🌟 修复电影/演唱会等加 S01E01 的逻辑
    if any(k in sub_path for k in ["电影", "movie", "演唱会", "纪录片"]):
        part_match = re.search(r'(?i)(part\d+|cd\d+)', original_filename)
        part_str = f".{part_match.group(1).lower()}" if part_match else ""
        year_part = f".{year_str}" if year_str else ""
        return f"{clean_show_name.replace(' ', '.')}{year_part}{part_str}{tag_str}{ext}".replace('..', '.')

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
    return f"{clean_show_name}.S{season_num:02d}E{ep_num:02d}{year_part}{tag_str}{ext}"

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
            res = requests.post(f"{self.base_url}sendMessage", json=payload, timeout=10).json()
            return res.get("result", {}).get("message_id")
        except: return None

    def edit_message(self, message_id, text, reply_markup=None):
        if not self.bot_token: return
        payload = {"chat_id": self.user_id, "message_id": message_id, "text": text, "parse_mode": "HTML"}
        if reply_markup: payload["reply_markup"] = json.dumps(reply_markup)
        try: requests.post(f"{self.base_url}editMessageText", json=payload, timeout=10)
        except: pass

    def answer_callback(self, callback_query_id, text=""):
        if not self.bot_token: return
        try: requests.post(f"{self.base_url}answerCallbackQuery", json={"callback_query_id": callback_query_id, "text": text}, timeout=5)
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

def auto_relogin(client_obj):
    global last_login_time
    current_time = time.time()
    
    if current_time - last_login_time < 1800:
        logger.warning("⏳ [系统] 检测到接口报错，防风控冷却锁生效，跳过登录！")
        return False
        
    logger.info("🔄 [系统] 触发保活机制：正在重新进行协议握手...")
    try:
        client_obj.login(ENV_189_CLIENT_ID, ENV_189_CLIENT_SECRET)
        last_login_time = time.time()
        logger.info("✅ [系统] 重新登录成功！安全冷却锁已重置。")
        return True
    except Exception as e:
        logger.error(f"❌ [系统] 重新登录失败: {e}")
        return False

# 🌟 独立新增的 CAS 收割模块 (终极暴力认亲 + 整季合并极速批次处理版)
def process_cas_via_olist_api():
    OLIST_URL = "http://127.0.0.1:5244"  
    OLIST_USER = "admin"      
    OLIST_PASS = "xxsky1127"   
    
    # 🌟 动态读取巡逻目录，如果没有配置，默认保留原来的两个
    s = load_json(SETTINGS_FILE)
    WATCH_DIRS = s.get("watch_dirs", ["/family/177_cas", "/local_cas"])
    
    processed_names = []
    if not WATCH_DIRS: return processed_names # 🛡️ 防呆：如果你把目录全删光了，它直接下班，防止报错

    try:
        r_log = requests.post(f"{OLIST_URL}/api/auth/login", json={"username": OLIST_USER, "password": OLIST_PASS}, timeout=10)
        login_res = r_log.json()
        r_log.close()
        if login_res.get("code") != 200: return processed_names
        token = login_res["data"]["token"]
        headers = {"Authorization": token, "Content-Type": "application/json"}
    except Exception: return processed_names

    def scan_olist_dir(path):
        files = []
        try:
            r = requests.post(f"{OLIST_URL}/api/fs/list", json={"path": path, "password": "", "page": 1, "per_page": 3000, "refresh": True}, headers=headers, timeout=45)
            res = r.json()
            r.close()
            if res.get("code") != 200: return files
            
            content = (res.get("data") or {}).get("content") or []
            for item in content:
                item_path = f"{path}/{item['name']}".replace("//", "/")
                if item["is_dir"]: 
                    files.extend(scan_olist_dir(item_path))
                else:
                    if item["name"].lower().endswith(".cas"):
                        files.append({"name": item["name"], "dir": path, "full_path": item_path})
        except Exception as e:
            logger.warning(f"⚠️ [警告] 扫描目录 {path} 时 OpenList 接口超时无响应: {e}")
        return files

    cas_files = []
    for watch_dir in WATCH_DIRS:
        logger.info(f"🌾 [收割] 正在巡逻云端目录: {watch_dir}")
        cas_files.extend(scan_olist_dir(watch_dir))

    if not cas_files: return processed_names

    subs = load_json(SUBS_FILE)
    current_ym = datetime.now().strftime("%Y%m")
    updated_paths = set()
    created_dirs = set()
    
    # =================================================================
    # 🚀 效率跃迁步骤 1：先不急着改名移动，把属于同一个家(目录)的文件合并编组
    # =================================================================
    batch_groups = {}  # 结构: { (原目录, 目标目录): [文件信息字典1, 文件信息字典2] }

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

        # 季数双向决策逻辑 (保留上一轮修复成果)
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

        # ====== 🥇 第一优先：查“专属收割记录库” ======
        try:
            h_data = load_json(HARVEST_SUBS_FILE)
            if search_key in h_data:
                best_match_path = h_data[search_key]
        except Exception:
            pass

        # ====== 🥈 第二优先：查“订阅库” ======
        if not best_match_path:
            # 查本地库 (升级版：防止乱认亲戚截胡)
            db_possible_matches = []
            for sid, info_dict in subs.items():
                if isinstance(info_dict, dict):
                    db_path = info_dict.get("path", "")
                    if DIR_CAS_ROOT not in db_path: continue 
                    db_folders = db_path.split('/')
                    for idx, f_name in enumerate(db_folders):
                        pure_f = get_match_key(f_name)
                        if not pure_f or len(pure_f) < 2: continue
                        if re.match(r'^\d{4,6}$', pure_f) or pure_f in ignore_words: continue
                        if search_key == pure_f: 
                            db_possible_matches.append("/".join(db_folders[:idx+1]))
                            break
                            
            if db_possible_matches:
                # 同样祭出立体打分，防止记忆库瞎认亲
                db_possible_matches.sort(key=lambda p: (
                    0 if p.split('/')[-1].lower() == show_folder_name.lower() else 1, # 1. 名字完全一样绝对优先
                    len(p.split('/')[-1])                                             # 2. 名字越短越干净越优先
                ))
                best_match_path = db_possible_matches[0]

        # ====== 🥉 第三优先：查雷达（跨月份全局扫描精准优选 + HDR原配优先装甲） ======
        if not best_match_path:
            try:
                search_roots = []
                if category_key:
                    b_large, b_sub = CAT_ROUTER[category_key]
                    search_roots.append(get_openlist_path(f"{DIR_CAS_ROOT}/{b_large}/{b_sub}".strip("/").replace("//", "/")))
                else:
                    search_roots = [get_openlist_path(f"{DIR_CAS_ROOT}/动漫/0-动漫"), get_openlist_path(f"{DIR_CAS_ROOT}/电视剧/0-电视剧")]
                
                for root_path in search_roots:
                    r = requests.post(f"{OLIST_URL}/api/fs/list", json={"path": root_path, "password": "", "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=20)
                    if r.json().get("code") == 200:
                        ym_nodes = [item["name"] for item in (r.json().get("data") or {}).get("content", []) if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                        ym_nodes.sort(reverse=True)
                        
                        # 🌟 把备选池提到外面，准备收集所有月份的匹配项
                        all_months_matches = []
                        
                        for ym in ym_nodes:
                            ym_path = f"{root_path}/{ym}"
                            r2 = requests.post(f"{OLIST_URL}/api/fs/list", json={"path": ym_path, "password": "", "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=20)
                            if r2.json().get("code") == 200:
                                for item in (r2.json().get("data") or {}).get("content", []):
                                    if item["is_dir"] and search_key == get_match_key(item["name"]):
                                        # 记录下它在哪个月份叫什么名字
                                        all_months_matches.append({"ym": ym, "name": item["name"]})
                        
                        # 🌟 全盘月份扫描结束，开始三维立体全局优选
                        if all_months_matches:
                            # show_folder_name 就是你传进来的源文件夹原名（比如：搜神记 (2026) HDR）
                            all_months_matches.sort(key=lambda x: (
                                0 if x["name"].lower() == show_folder_name.lower() else 1,  # 第一维：名字完全一样，优先级绝对最高（门当户对）
                                len(x["name"]),                                             # 第二维：名字越短越干净越优先（过滤杂牌标签）
                                -int(x["ym"])                                               # 第三维：年份月份越新越优先
                            ))
                            
                            best_share = all_months_matches[0]
                            best_match_path = f"{DIR_CAS_ROOT}/{b_large}/{b_sub}/{best_share['ym']}/{best_share['name']}".replace("//", "/")
                            
                    if best_match_path:
                        break
            except:
                pass

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

        final_name = generate_smart_name(filename, target_cloud_path) or filename
        final_target_dir = get_openlist_path(target_cloud_path)
        
        # 归入编组
        group_key = (raw_dir, final_target_dir, base_notify_path)
        if group_key not in batch_groups: batch_groups[group_key] = []
        batch_groups[group_key].append({"orig": filename, "final": final_name})

    # =================================================================
    # 🚀 效率跃迁步骤 2：以整部剧/整季为单位，执行极速合并洗名与一波流搬运
    # =================================================================
    for (raw_dir, final_target_dir, base_notify_path), file_items in batch_groups.items():
        # 打基建
        if final_target_dir not in created_dirs:
            requests.post(f"{OLIST_URL}/api/fs/mkdir", json={"path": final_target_dir}, headers=headers).close()
            created_dirs.add(final_target_dir)
            logger.info(f"📁 [基建] 新建目录: {final_target_dir} ...")
            time.sleep(1.5)

        logger.info(f"✨ [洗名] 正在极速挨个下发整组重命名指令 (共 {len(file_items)} 个文件) ...")
        names_to_move = []
        
        # 1. 紧凑循环极速改名（无间歇连发，极其丝滑）
        for item in file_items:
            orig_n, final_n = item["orig"], item["final"]
            if orig_n != final_n:
                src_path = f"{raw_dir}/{orig_n}".replace("//", "/")
                requests.post(f"{OLIST_URL}/api/fs/rename", json={"name": final_n, "path": src_path}, headers=headers).close()
                names_to_move.append(final_n)
            else:
                names_to_move.append(orig_n)
                
        # 2. 极其精明的集体防空缓冲：整组文件只等一次落盘，不用等20次！
        logger.info("⏳ [缓冲] 整组重命名下发完毕，集中等待底层统一刷新落盘 (5秒)...")
        time.sleep(5.0)
        requests.post(f"{OLIST_URL}/api/fs/list", json={"path": raw_dir, "refresh": True}, headers=headers).close()

        # 3. 💥 终极收割：一波流批量 Move！直接传一整串数组！
        logger.info(f"🚚 [搬运] 正在打包移入新家: {final_target_dir} (批量 {len(names_to_move)} 件一波流)")
        r_mov = requests.post(f"{OLIST_URL}/api/fs/move", json={"src_dir": raw_dir, "dst_dir": final_target_dir, "names": names_to_move}, headers=headers)
        mov_res = r_mov.json()
        r_mov.close()
        
        if mov_res.get("code") == 200:
            logger.info(f"✅ [搬运] 批量整组移动成功！")
            processed_names.extend(names_to_move)
            updated_paths.add(base_notify_path)
        else:
            err_msg = mov_res.get('message', str(mov_res))
            logger.warning(f"⚠️ [搬运] 批量移动受阻: {err_msg}。启动单件排查与强制清理模式...")
            
            # 💡 降级为逐个处理：精准解决 Openlist 缓存导致的源文件残留问题
            for name in names_to_move:
                r_single = requests.post(f"{OLIST_URL}/api/fs/move", json={"src_dir": raw_dir, "dst_dir": final_target_dir, "names": [name]}, headers=headers)
                single_res = r_single.json()
                r_single.close()

                if single_res.get("code") == 200:
                    logger.info(f"✅ [搬运] 单件扫尾移动成功: {name}")
                    processed_names.append(name)
                    updated_paths.add(base_notify_path)
                else:
                    single_err = single_res.get("message", "").lower()
                    # 如果提示文件已存在，说明上次已经挪过去了，目前只是个没删掉的缓存幽灵
                    if "exist" in single_err:
                        logger.info(f"🗑️ [清理] 目标端已安全存在 [{name}]，正在强行斩杀残留的源文件...")
                        requests.post(f"{OLIST_URL}/api/fs/remove", json={"dir": raw_dir, "names": [name]}, headers=headers).close()
                        # 文件既然已经在那边了，正常将其加入已处理列表，确保后续能照常通知 5000 管家强刷
                        processed_names.append(name)
                        updated_paths.add(base_notify_path)
                    else:
                        logger.error(f"❌ [搬运] 单件移动彻底失败: {name} - {single_res.get('message')}")

    # =================================================================
    # 🚨 通知管家收尾 (极致精准 - 影剧双修通吃版)
    # =================================================================
    if updated_paths:
        logger.info("⏳ [引擎] 物理归档结束，等待云盘底层统一落盘缓冲 (8秒)...")
        time.sleep(8)
        
        # 🌟 核心极致调优：全栖兼容分季剧集与独立电影文件夹
        target_media_roots = set()
        for p in updated_paths:
            parts = p.split('/')
            media_root = ""
            
            # 轨一：探测是否含有 Season 分季 (针对剧集/动漫)
            for i, part in enumerate(parts):
                if part.lower().startswith("season"):
                    media_root = '/'.join(parts[:i])
                    break
            
            # 轨二：无 Season 分季兜底 (针对独立电影)
            if not media_root:
                # 检查最后一项是否包含扩展名点号 (判定为文件还是目录)
                # 常见后缀过滤，避免把带有标点符号的独立目录误判为文件
                last_part = parts[-1]
                if '.' in last_part and any(last_part.lower().endswith(ext) for ext in ['.mp4', '.mkv', '.ts', '.iso', '.rmvb', '.avi', '.cas', '.strm']):
                    # 传入的是具体电影文件，取其直接所在的电影文件夹
                    media_root = os.path.dirname(p)
                else:
                    # 传入的本身就是目标电影目录，原封不动直接锁定
                    media_root = p
                    
            if media_root:
                target_media_roots.add(media_root)
        
        logger.info(f"📦 [引擎] 聚合完毕，共锁定 {len(target_media_roots)} 个独立影剧根目录待强刷。")
        
        # 仅对精准提取出的影剧总目录发起轻量级同步
        for media_root in target_media_roots:
            olist_p = get_openlist_path(media_root)
            try:
                requests.get("http://127.0.0.1:5000/api/sync", params={"path": olist_p}, timeout=5).close()
                time.sleep(2)  # 黄金间隔保护上游搜刮接口
            except Exception as e:
                logger.debug(f"⚠️ 通知目录强刷闪断 (可静默忽略): {e}")
                pass
                
        logger.info(f"🔔 [收割] 任务彻底完成，已精准强刷 {len(target_media_roots)} 个专属影视。")
        
    # 🌟 终极完美交棒：直接交出系统早已全自动收集好的精准单集文件名列表！
    return processed_names

# 🌟 你的原版查重逻辑，补入 ignore_time 和 实体文件查验防假历史
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
                # 🎯 原版的进阶集数去重逻辑 (一字未改)
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
                auto_relogin(client_obj)

    if global_cas_paths:
        for p in global_cas_paths:
            try:
                requests.get("http://127.0.0.1:5000/api/sync", params={"path": p}, timeout=3) 
                logger.info(f"⚡ [API] 成功向管家后方下发同步指令: {p}")
                notifier.send_message(f"✅ 管家同步指令已下发: {p}")
            except Exception as e: 
                logger.error(f"❌ [API] 管家服务无法联通: {e}")
                notifier.send_message(f"❌ 管家同步无响应: {e}")
            time.sleep(1) 
        
    for p in global_emby_paths:
        try:
            subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", p], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
            logger.info(f"⚡ [脚本] 成功唤醒原生 Emby 刷新: {p}")
            notifier.send_message(f"✅ Emby刷新指令已下发: {p}")
        except: pass
        time.sleep(2)

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
        wait_min = random.randint(25, 45)
        logger.info(f"🛌 [系统] 航线巡逻结束。进入节电待机，距下次起飞还有 {wait_min} 分钟...")
        logger.info("==========================================")
        schedule.clear('patrol')
        schedule.every(wait_min).minutes.do(scheduled_task).tag('patrol')

    scheduled_task()
    schedule.every(6).hours.do(auto_relogin, client_obj)

    while True:
        schedule.run_pending()
        try:
            url = f"https://api.telegram.org/bot{TG_BOT_TOKEN}/getUpdates?offset={offset}&timeout=10"
            res = requests.get(url, timeout=15).json()
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
                                # 🌟 动态插入：如果选了周更，立刻弹出周一到周日的选项
                                kb = {"inline_keyboard": [
                                    [{"text": "周一", "callback_data": "wiz_day_周一"}, {"text": "周二", "callback_data": "wiz_day_周二"}, {"text": "周三", "callback_data": "wiz_day_周三"}],
                                    [{"text": "周四", "callback_data": "wiz_day_周四"}, {"text": "周五", "callback_data": "wiz_day_周五"}, {"text": "周六", "callback_data": "wiz_day_周六"}],
                                    [{"text": "周日", "callback_data": "wiz_day_周日"}, {"text": "自动/默认", "callback_data": "wiz_day_未知"}],
                                    [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                                ]}
                                notifier.edit_message(msg_id, f"✅ 剧名: {wizard_states[chat_id]['title']}\n✅ 频率: {freq}\n\n<b>📅 请选择该剧的更新时间 (周几):</b>", kb)
                            else:
                                # 非周更，直接跳到分类选项 (🌟 加入了 演唱会)
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

                        # 🌟 新增的动态开关与拉取指令
                        if text in ["开启自动收割", "开启扫描", "/ascan"]:
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
                                    auto_relogin(client_obj)
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
                                    auto_relogin(client_obj)
                                    notifier.send_message("✅ 引擎已重新握手自愈，请重发指令！")
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
                                    auto_relogin(client_obj)
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
                                            r_log = requests.post("http://127.0.0.1:5244/api/auth/login", 
                                                                  json={"username": "admin", "password": "xxsky1127"}, timeout=5).json()
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
                                                    r_list = requests.post("http://127.0.0.1:5244/api/fs/list", 
                                                                           json={"path": base_p}, headers=o_headers, timeout=5).json()
                                                    if r_list.get("code") == 200:
                                                        content = (r_list.get("data") or {}).get("content") or []
                                                        ym_dirs = [item["name"] for item in content if item["is_dir"] and re.match(r'^\d{4,6}$', item["name"])]
                                                        ym_dirs.sort(reverse=True) # 优先从最新的月份往老月份找
                                                        
                                                        for ym in ym_dirs:
                                                            ym_path = f"{base_p}/{ym}"
                                                            r_shows = requests.post("http://127.0.0.1:5244/api/fs/list", 
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
                                        r_log = requests.post("http://127.0.0.1:5244/api/auth/login", 
                                                              json={"username": "admin", "password": "xxsky1127"}, timeout=5).json()
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
                                                requests.post("http://127.0.0.1:5244/api/fs/list", 
                                                              json={"path": butler_path, "refresh": True}, headers=o_headers, timeout=15).close()
                                                time.sleep(3.0) # 给底层留出落盘和 Openlist 自建 strm 的缓冲时间
                                            except: pass

                                        # 🔀 第二步：严格的双轨路由分流
                                        if DIR_CAS_ROOT in openlist_p:
                                            # 177-秒传 -> 交给 5000 端口管家收割
                                            try: 
                                                requests.get("http://127.0.0.1:5000/api/sync", params={"path": butler_path}, timeout=3).close()
                                                notifier.send_message(f"✅ 管家同步指令已精准下发: {butler_path}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            # 177-视频与原老目录 -> Openlist 自身已吐出 STRM，直接唤醒 Emby 原生拉取
                                            try:
                                                subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_p], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
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
                                        r_log = requests.post("http://127.0.0.1:5244/api/auth/login", 
                                                              json={"username": "admin", "password": "xxsky1127"}, timeout=5).json()
                                        if r_log.get("code") == 200: 
                                            o_headers = {"Authorization": r_log["data"]["token"], "Content-Type": "application/json"}
                                    except: pass
                                    
                                    for rp in scan_roots:
                                        # 💥 无论 CAS 还是普通视频，全域强制穿透 Openlist 缓存
                                        if o_headers:
                                            try:
                                                requests.post("http://127.0.0.1:5244/api/fs/list", 
                                                              json={"path": rp, "refresh": True}, headers=o_headers, timeout=10).close()
                                                time.sleep(1.5) # 批量扫描稍微给点延时错峰即可
                                            except: pass

                                        # 🔀 严格的双轨路由分流
                                        if DIR_CAS_ROOT in rp:
                                            try: requests.get("http://127.0.0.1:5000/api/sync", params={"path": rp}, timeout=3).close()
                                            except: pass
                                        else:
                                            try: subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", rp], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                            except: pass
                                        
                                    notifier.send_message(f"✅ 全区最高指令已下发！安全触发并双轨刷新了 {len(scan_roots)} 个雷达基点。")
                                # =================================================================
                                # 🌟 终极双轨驱动装甲替换结束
                                # =================================================================

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
                                                requests.get("http://127.0.0.1:5000/api/sync", params={"path": openlist_target_path}, timeout=3).close()
                                                notifier.send_message(f"✅ 管家同步指令已下发: {openlist_target_path}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            try: 
                                                subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
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
                                                        requests.get("http://127.0.0.1:5000/api/sync", params={"path": openlist_target_path}, timeout=3).close()
                                                        notifier.send_message(f"✅ 管家同步指令已下发: {openlist_target_path}")
                                                    except Exception as e: 
                                                        notifier.send_message(f"❌ 管家同步无响应: {e}")
                                                else:
                                                    try:
                                                        subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL, start_new_session=True)
                                                        notifier.send_message(f"✅ Emby刷新指令已下发: {openlist_target_path}")
                                                    except: pass
                                        else:
                                            notifier.send_message(f"❌ 天翼云拒绝转存: {code}")
                                except Exception as e:
                                    err_str = str(e).upper()
                                    if "INVALIDSESSIONKEY" in err_str or "CHECK IP ERROR" in err_str or "UNKNOWN_ERROR" in err_str or "UNKNOWN" in err_str:
                                        notifier.send_message(f"⚠️ 检测到 IP 漂移！正在自愈...")
                                        if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                        auto_relogin(client_obj)
                                        notifier.send_message("✅ IP 漂移已修复！请重发指令。")
                                    else:
                                        notifier.send_message(f"❌ 补档异常: {e}")

                        elif text.startswith("订阅") or text.startswith("绑定" or text.startswith("/sub ")):
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
                                parts = text.split()
                                share_url, keyword, target_path = "", "", ""

                                url_index = -1
                                for i, p in enumerate(parts):
                                    if p.startswith("http"):
                                        share_url, url_index = p, i
                                        break

                                if url_index != -1:
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
                                            auto_relogin(client_obj)
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
                                        auto_relogin(client_obj)
                                        notifier.send_message("✅ 自愈完成，请重发指令。")
                                    else:
                                        notifier.send_message(f"❌ 云端拦截: {e}")
        except Exception: pass 
        time.sleep(2)

if __name__ == '__main__':
    os.makedirs(DB_DIR, exist_ok=True)
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    notifier.send_message(f"🤖 追剧转存引擎 (V5.8 智能路由版) 启动，获取凭证...")
    
    try:
        logger.info("✅ [系统] 189底层接口握手成功，正在挂载凭证...")
        client = Cloud189()
        client.login(ENV_189_CLIENT_ID, ENV_189_CLIENT_SECRET)
        last_login_time = time.time()
        notifier.send_message("✅ 网盘登录成功！全天候监控已就位。\n💡 提示：回复「取消X」清空假记忆，或回复「同步订阅」强制检查更新。")
    except Exception as e:
        logger.error(f"❌ [系统] 登录失败: {e}")
        notifier.send_message(f"❌ 首次登录失败: {e}\n(脚本已进入休眠模式防止被封)")
        time.sleep(600)
        sys.exit(-1)
        
    main_control_loop(client)
```

3.2电脑文件传到手机storage/downloads目录下

3.3复制脚本
```
cp ~/storage/downloads/auto189.py ~/189py/auto189.py
```
4.点火升空 & 日常操作
前台启动
```
python ~/189py/auto189.py
```
后台静默启动
```
nohup python ~/189py/auto189.py > run.log 2>&1 &
```
pm2启动：
```
cd ~/189py && pm2 start auto189.py --name "auto189" --interpreter python
```
或
```
(cd ~/189py && nohup python3 auto_189.py > run.log 2>&1 &) && echo "✅ TG 追剧管家已启动，日志记录于 ~/run.log"
```

5.清理脏数据，正式接客

在 Termux 里直接执行这两行命令（先清空数据库，再重新启动）：
```
# 1. 强制清空订阅数据库和历史记录（把脏数据扬了）
pkill -f auto189.py
rm -f ~/189py/db/history.json ~/189py/db/subscriptions.json

# 2. 重新启动脚本！
python ~/189py/auto189.py
```

### 三、秒传播放casplay.py
casplay.py与auto189.py同目录下操作方法一样

1.py内容：
```
import base64, json, time, random, hashlib, hmac, urllib.parse, threading, uuid, os, requests, logging, subprocess, math
import socket, re, functools
import urllib3
urllib3.util.connection.HAS_IPV6 = False
from collections import deque
from flask import Flask, request, redirect, render_template_string, jsonify
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import pad

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
# 🏠 局域网探针 (新增)
# ==========================================
def get_lan_server_ip(req):
    """嗅探请求头，判断是否处于局域网环境，并提取内网 IP"""
    host = req.host.split(':')[0]
    # 匹配标准的私有局域网网段
    if re.match(r'^(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[0-1])\.|127\.0\.0\.1)', host):
        return host
    return None

# ==========================================
# ⚙️ 默认系统配置 (主服务)
# ==========================================
DEFAULT_CONFIG = {
    "server_host": "https://play.363689.xyz",
    "delete_delay": 600,
    "shield_delay": 2700,
    "cloud_strategy": "hash", 
    "family_clouds": [],
    "openlist_host": "http://127.0.0.1:5244",
    "openlist_token": "", 
    "network_cas_path": "/177/177-秒传",
    "local_strm_dir": "/storage/emulated/0/Download/cas_strm",
    "pushplus_token": "",
    "tg_bot_token": "",
    "tg_chat_id": ""
}

# ==========================================
# 🎬 Emby 双核 302 劫持配置 (原 nginx_302 模块参数)
# ==========================================
EMBY_HOST = "http://127.0.0.1:8096"
API_KEY_LINUX = "751c095055f8493d8e63eb755369b9aa"
API_KEY_APP = "66644805d4bc45ea91b2a5e5eca22105"

app_main = Flask('cas_server_5000')
app_302 = Flask('nginx_302_5001')

last_refresh_time = 0
upload_cache = {}
cache_lock = threading.Lock()
rr_index = 0

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_DIR = os.path.join(BASE_DIR, "db")
os.makedirs(DB_DIR, exist_ok=True)

def get_db_path(): return os.path.join(DB_DIR, "config.json")

def format_size(size_in_bytes):
    try:
        size = float(size_in_bytes)
        if size < 1024 * 1024 * 1024:
            return f"{size / (1024 * 1024):.2f} MB"
        else:
            return f"{size / (1024 * 1024 * 1024):.2f} GB"
    except:
        return "未知大小"

# ==========================================
# 🔔 统一看板日志与推送系统
# ==========================================
log_buffer = deque(maxlen=150) 
class MemoryHandler(logging.Handler):
    def emit(self, record):
        msg = self.format(record)
        log_buffer.append({'time': time.strftime("%H:%M:%S"), 'level': record.levelname, 'msg': f"💖 [管家] {msg}"})

logger = logging.getLogger('CAS_Server')
logger.setLevel(logging.INFO)
mem_handler = MemoryHandler()
mem_handler.setFormatter(logging.Formatter('%(message)s'))
stream_handler = logging.StreamHandler()
stream_handler.setFormatter(logging.Formatter('%(asctime)s - %(levelname)s - %(message)s'))
logger.addHandler(mem_handler)
logger.addHandler(stream_handler)

logging.getLogger('werkzeug').setLevel(logging.ERROR)

@app_main.route('/api/remote_log', methods=['POST'])
def receive_remote_log():
    try:
        data = request.json
        log_buffer.append({'time': time.strftime("%H:%M:%S"), 'level': data.get('level', 'INFO'), 'msg': f"💜 [引擎] {data.get('msg', '')}"})
        return "OK", 200
    except: return "Error", 400

def send_push(title, content):
    def _do_push():
        cfg = read_config()
        if cfg.get('pushplus_token'):
            try: requests.get(f"http://www.pushplus.plus/send?token={cfg['pushplus_token']}&title={urllib.parse.quote(title)}&content={urllib.parse.quote(content)}&template=html", timeout=5)
            except Exception as e: logger.error(f"微信推送失败: {e}")
        if cfg.get('tg_bot_token') and cfg.get('tg_chat_id'):
            try: requests.post(f"https://api.telegram.org/bot{cfg['tg_bot_token']}/sendMessage", data={"chat_id": cfg['tg_chat_id'], "text": f"🚨 <b>{title}</b>\n\n{content.replace('<br>', '\n')}", "parse_mode": "HTML"}, timeout=5)
            except Exception as e: logger.error(f"TG推送失败: {e}")
    threading.Thread(target=_do_push, daemon=True).start()

# ==========================================
# 🔑 天翼云独立鉴权引擎
# ==========================================
def rsaEncrpt(password, public_key):
    rsakey = RSA.importKey(public_key)
    cipher = PKCS1_v1_5.new(rsakey)
    return cipher.encrypt(password.encode()).hex()

def get_session_key_via_api(session_obj, source="未知来源"):
    try:
        url = "https://cloud.189.cn/v2/getUserBriefInfo.action"
        headers = {"Accept": "application/json;charset=UTF-8", "Referer": "https://cloud.189.cn/"}
        res = session_obj.get(url, headers=headers, timeout=10).json()
        sk = res.get("sessionKey")
        if sk: logger.info(f"✨ [{source} 获取成功] 拿到新鲜 SESSION_KEY！凭证尾号: {sk[-6:]}")
        return sk
    except Exception as e:
        logger.error(f"提取 sessionKey 失败: {e}")
        return None

class Cloud189AuthEngine:
    def __init__(self):
        self.session = requests.session()
        self.session.headers = {
            'User-Agent': "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Accept": "application/json;charset=UTF-8",
        }

    def getEncrypt(self):
        return self.session.post("https://open.e.189.cn/api/logbox/config/encryptConf.do", data={'appId': 'cloud'}, timeout=15).json()['data']['pubKey']

    def getRedirectURL(self):
        rsp = self.session.get('https://cloud.189.cn/api/portal/loginUrl.action?redirectURL=https://cloud.189.cn/web/redirect.html?returnURL=/main.action', timeout=15)
        return urllib.parse.parse_qs(urllib.parse.urlparse(rsp.url).query)

    def do_login_and_get_key(self, username, password, slot_name="卡槽自愈"):
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
            sk = get_session_key_via_api(self.session, slot_name)
            if sk: return sk
            else: raise Exception("账号密码验证成功，但在接口中未找到 sessionKey")
        else:
            raise Exception(result['msg'])

slot6_cache = {"sk": "", "cookie_mtime": 0}

def get_auto189_session_key():
    cookie_file = os.path.join(DB_DIR, "cookies.json")
    if not os.path.exists(cookie_file): return ""
    mtime = os.path.getmtime(cookie_file)
    if slot6_cache["sk"] and slot6_cache["cookie_mtime"] == mtime: return slot6_cache["sk"]
    session = requests.Session()
    try:
        with open(cookie_file, 'r', encoding='utf-8') as f: session.cookies.update(json.load(f))
        sk = get_session_key_via_api(session, "卡槽6-外部同步") or ""
        slot6_cache["sk"] = sk
        slot6_cache["cookie_mtime"] = mtime
        return sk
    except: return ""

def save_config(cfg):
    cfg_path = get_db_path()
    with open(cfg_path, 'w', encoding='utf-8') as f: json.dump(cfg, f, ensure_ascii=False, indent=4)

def read_config():
    cfg_path = get_db_path()
    cfg = DEFAULT_CONFIG.copy()
    try:
        if os.path.exists(cfg_path):
            with open(cfg_path, 'r', encoding='utf-8') as f: cfg.update(json.load(f))
    except: pass
    
    if len(cfg.get('family_clouds', [])) > 5:
        if not cfg['family_clouds'][5].get('username'):
            old_sk = cfg['family_clouds'][5].get('session_key', '')
            new_sk = get_auto189_session_key()
            if new_sk:
                cfg['family_clouds'][5]['session_key'] = new_sk
                if old_sk != new_sk: save_config(cfg)
    return cfg

def refresh_slot_logic(slot_idx, cfg):
    if slot_idx < len(cfg.get('family_clouds', [])):
        fc = cfg['family_clouds'][slot_idx]
        user, pwd = fc.get('username'), fc.get('password')
        if user and pwd:
            logger.info(f"🔄 卡槽 {slot_idx+1} 凭证失效，正在使用账号 {user} 自动重登自愈...")
            send_push("🔄 追剧管家自愈启动", f"检测到 卡槽 {slot_idx+1} 凭证失效，引擎已介入执行自动重登。")
            try:
                auth = Cloud189AuthEngine()
                new_sk = auth.do_login_and_get_key(user, pwd, f"卡槽{slot_idx+1}")
                if new_sk:
                    # 1. 更新当前内存里的副本，保证当前次请求顺畅
                    fc['session_key'] = new_sk
                    
                    # 2. 🚀 修复核心：精准保存，防止旧内存覆盖前台刚刚手动修改的新配置
                    latest_cfg = read_config()
                    if slot_idx < len(latest_cfg.get('family_clouds', [])):
                        latest_cfg['family_clouds'][slot_idx]['session_key'] = new_sk
                        save_config(latest_cfg)
                        
                    if 'session_key' in fc and fc['session_key'] in client.rsa_keys:
                        del client.rsa_keys[fc['session_key']]
                    logger.info(f"✅ 卡槽 {slot_idx+1} 满血复活！")
                    return new_sk
            except Exception as e:
                logger.error(f"❌ 卡槽 {slot_idx+1} 自动续期失败: {e}")
                send_push("❌ 追剧管家自愈失败", f"卡槽 {slot_idx+1} 尝试自动重登失败！<br>报错信息: {e}")
        elif slot_idx == 5: 
            logger.warning("🚨 备用卡槽 6 外部凭证失效，已清理本地废弃凭证，等待外部定时任务更新。")
            send_push("⚠️ 卡槽 6 离线转移", "卡槽 6 外部凭证失效。已自动丢弃旧凭证，当前播放请求将无缝转移至主卡槽。")
            cookie_path = os.path.join(DB_DIR, "cookies.json")
            if os.path.exists(cookie_path): os.remove(cookie_path)
            slot6_cache["sk"] = ""
            if len(cfg.get('family_clouds', [])) > 5:
                # 1. 更新当前内存副本
                cfg['family_clouds'][5]['session_key'] = ""
                
                # 2. 🚀 修复核心：精准拉取最新配置再保存
                latest_cfg = read_config()
                if len(latest_cfg.get('family_clouds', [])) > 5:
                    latest_cfg['family_clouds'][5]['session_key'] = ""
                    save_config(latest_cfg)
            return None
        else: logger.error(f"❌ 卡槽 {slot_idx+1} 缺少账号密码配置，无法执行自愈！")
    return None

def get_target_cloud(cfg, bind_key="", file_size=0):
    global rr_index
    raw_clouds = cfg.get('family_clouds', [])
    def is_valid(c): return c and c.get('family_id') and c.get('hard_folder_id') and c.get('openlist_prefix')

    valid_clouds = [(c, i) for i, c in enumerate(raw_clouds) if is_valid(c)]
    if not valid_clouds: return None, -1

    strategy = cfg.get('cloud_strategy', 'hash')
    
    # 🎯 绝对优先的单卡槽锁定！只要用户手动指定了单卡槽，绝不被大文件拦截逻辑干扰
    if strategy == 'primary' and len(raw_clouds) > 0 and is_valid(raw_clouds[0]): return raw_clouds[0], 0
    elif strategy == 'slot2' and len(raw_clouds) > 1 and is_valid(raw_clouds[1]): return raw_clouds[1], 1
    elif strategy == 'slot3' and len(raw_clouds) > 2 and is_valid(raw_clouds[2]): return raw_clouds[2], 2
    elif strategy == 'slot4' and len(raw_clouds) > 3 and is_valid(raw_clouds[3]): return raw_clouds[3], 3
    elif strategy == 'slot5' and len(raw_clouds) > 4 and is_valid(raw_clouds[4]): return raw_clouds[4], 4
    elif strategy == 'slot6' and len(raw_clouds) > 5 and is_valid(raw_clouds[5]): return raw_clouds[5], 5

    # ⚖️ 容量调度介入：检测超大文件 (>28G)，无视常规策略，强制分配给卡槽 5 和 卡槽 6
    try: size_int = int(file_size)
    except: size_int = 0
    if size_int > 28 * 1024 * 1024 * 1024:
        big_clouds = [(c, i) for c, i in valid_clouds if i in [4, 5]]
        if big_clouds:
            # 在 5 和 6 之间利用哈希平摊超大文件压力
            if bind_key:
                hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(big_clouds)
                target = big_clouds[hash_idx]
            else:
                target = random.choice(big_clouds)
            logger.info(f"⚖️ 容量调度介入：检测到超大文件 ({size_int/(1024**3):.2f} GB)，无视常规策略，强制锁定大容量 [卡槽 {target[1]+1}]")
            return target[0], target[1]
        else:
            logger.warning("⚠️ 检测到大于28G的超大文件，但负责承接的 [卡槽 5 / 卡槽 6] 未配置或已失效！将降级落入常规池。")

    # 常规均衡池：1-4号卡槽参与 (index 0, 1, 2, 3)
    general_clouds = [(c, i) for c, i in valid_clouds if i < 4]
    if not general_clouds: general_clouds = valid_clouds # 兜底机制

    if strategy == 'hash':
        if bind_key:
            hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(general_clouds)
            return general_clouds[hash_idx][0], general_clouds[hash_idx][1]
    elif strategy == 'random': 
        target = random.choice(general_clouds)
        return target[0], target[1]
    elif strategy == 'round_robin':
        target = general_clouds[rr_index % len(general_clouds)]
        rr_index += 1
        return target[0], target[1]
        
    return valid_clouds[0][0], valid_clouds[0][1]

# ==========================================
# 🧹 全量清空逻辑
# ==========================================
def force_clear_all_worker():
    logger.info("🧨 收到最高指令：开始精准清理家庭云缓存及回收站...")
    cfg = read_config()
    with cache_lock: upload_cache.clear()
    for i, fc in enumerate(cfg.get('family_clouds', [])):
        fam_id = fc.get('family_id')
        fold_id = fc.get('hard_folder_id')
        sk = fc.get('session_key')
        if not sk: sk = refresh_slot_logic(i, cfg)
        if fam_id and fold_id and sk:
            logger.info(f"🧹 正在搜寻卡槽 {i+1} [家庭组:{fam_id[-4:]}] 的残留缓存...")
            try:
                items = client.get_family_items(fam_id, fold_id, sk)
                del_count = 0
                for item in items:
                    if client.delete_item(fam_id, item['fileId'], sk): del_count += 1
                time.sleep(1)
                if client.empty_family_recycle(fam_id, sk):
                    logger.info(f"✅ 卡槽 {i+1} 清理完毕: 移除了 {del_count} 个秒传文件，并清空了所在家庭云的回收站。")
            except Exception as e: logger.error(f"❌ 卡槽 {i+1} 清理异常: {e}")
    logger.info("🎉 缓存清空作业已圆满完成！")
    send_push("🧹 追剧管家清理完成", "全矩阵秒传缓存及回收站已强制清空完毕。")

@app_main.route('/api/clear_all', methods=['POST'])
def api_clear_all():
    threading.Thread(target=force_clear_all_worker, daemon=True).start()
    return "✅ 清空指令下发成功", 200

# ==========================================
# 🖥️ ADMIN 界面与配置路由
# ==========================================
@app_main.route('/admin/config', methods=['POST'])
def update_global_config():
    old_cfg = read_config()
    old_clouds = old_cfg.get('family_clouds', [])
    
    cfg = DEFAULT_CONFIG.copy()
    for k, v in old_cfg.items(): cfg[k] = v
    
    clouds = []
    for i in range(6):
        fid = request.form.get(f'fc_id_{i}', '').strip()
        hid = request.form.get(f'fc_fd_{i}', '').strip()
        px = request.form.get(f'fc_prefix_{i}', '').strip()
        mt = request.form.get(f'fc_mount_{i}', '').strip()
        user = request.form.get(f'fc_user_{i}', '').strip()
        pwd = request.form.get(f'fc_pwd_{i}', '').strip()
        sk = request.form.get(f'fc_sk_{i}', '').strip()
        
        # 智能重置凭证逻辑
        if i < len(old_clouds):
            old_user = old_clouds[i].get('username', '')
            old_pwd = old_clouds[i].get('password', '')
            if user != old_user or pwd != old_pwd:
                sk = "" 
                logger.info(f"🔄 检测到卡槽 {i+1} 账号或密码变更，已自动清除旧凭证，准备重新登录获取新 Key。")

        # 🔴 关键修复点：如果当前卡槽没填，必须塞一个空字典占位，严防后排数据错位塌陷！
        if fid and hid:
            cloud_data = {"family_id": fid, "hard_folder_id": hid, "openlist_prefix": px, "openlist_mount_path": mt, "username": user, "password": pwd, "session_key": sk}
            clouds.append(cloud_data)
        else:
            clouds.append({}) # <--- 必须有这个空字典强行占位！
            
    cfg['family_clouds'] = clouds
    cfg['cloud_strategy'] = request.form.get('cloud_strategy', 'hash')
    for k in cfg.keys():
        if k not in ['family_clouds', 'cloud_strategy'] and k in request.form:
            val = request.form.get(k, '').strip()
            if k in ['delete_delay', 'shield_delay']: cfg[k] = int(val) if val else (600 if k == 'delete_delay' else 2700)
            else: cfg[k] = val
    save_config(cfg)
    client.rsa_keys.clear() 
    logger.info(f"⚙️ 矩阵重组！当前挂载 {len(clouds)} 个独立播放节点。")
    return redirect("/admin?msg=所有配置已保存并实时生效")

ADMIN_HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>💖 追剧管家后台面板</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #f4f6f9; margin: 0; padding: 20px; color: #333; }
        .container { max-width: 900px; margin: 0 auto; }
        .header { background: #fff; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.03); margin-bottom: 20px; display: flex; justify-content: space-between; align-items: center; border-left: 5px solid #6366f1; }
        .card { background: #fff; padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.03); margin-bottom: 20px; }
        h2 { margin: 0; color: #1e293b; } h3 { margin-top: 0; color: #334155; font-size: 1.1rem; border-bottom: 1px solid #e2e8f0; padding-bottom: 10px; margin-bottom: 15px; }
        .badge { background: #10b981; color: white; padding: 5px 12px; border-radius: 20px; font-size: 12px; font-weight: bold; letter-spacing: 1px; }
        label { display: block; margin-bottom: 6px; font-weight: 600; color: #475569; font-size: 13px; }
        input, select { width: 100%; padding: 10px; margin-bottom: 15px; border: 1px solid #cbd5e1; border-radius: 6px; box-sizing: border-box; background: #f8fafc; transition: all 0.3s; }
        input:focus, select:focus { border-color: #6366f1; outline: none; box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.1); background: #fff; }
        button { background: #6366f1; color: white; border: none; padding: 10px 18px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s; }
        button:hover { background: #4f46e5; transform: translateY(-1px); }
        .btn-purple { background: #8b5cf6; width: 100%; } .btn-purple:hover { background: #7c3aed; }
        .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
        .status-grid { display: grid; grid-template-columns: 1fr 1.5fr 1fr; gap: 20px; align-items: center; }
        .status-msg { padding: 12px; border-radius: 6px; margin-bottom: 20px; background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; text-align: center; font-weight: bold; }
        .cloud-box { border: 1px dashed #cbd5e1; padding: 15px; border-radius: 8px; margin-bottom: 15px; background: #fafafa;}
        .cloud-title { font-size: 13px; font-weight: bold; color: #6366f1; margin-bottom: 10px;}
        .mac-window { background: #1e293b; border-radius: 10px; box-shadow: 0 10px 30px rgba(0,0,0,0.2); overflow: hidden; margin-bottom: 20px; }
        .mac-header { background: #0f172a; padding: 10px 15px; display: flex; gap: 8px; align-items: center; }
        .mac-btn { width: 12px; height: 12px; border-radius: 50%; }
        .btn-close { background: #ef4444; } .btn-min { background: #f59e0b; } .btn-max { background: #10b981; }
        .mac-title { color: #64748b; font-size: 12px; margin-left: 10px; font-weight: bold; letter-spacing: 1px; }
        .console { background: #1e293b; color: #cbd5e1; padding: 15px; height: 350px; overflow-y: auto; overflow-x: hidden; font-family: 'Consolas', 'Courier New', monospace; font-size: 13px; line-height: 1.6; white-space: pre-wrap; word-break: break-all; }
        .log-time { color: #64748b; margin-right: 8px; display: inline-block; vertical-align: top; }
        .log-msg { display: inline; }
        .log-INFO { color: #34d399; } .log-WARNING { color: #fbbf24; } .log-ERROR { color: #f87171; font-weight: bold; }
        .log-SUCCESS { color: #10b981; font-weight: bold; }
        @media (max-width: 768px) { .grid { grid-template-columns: 1fr; } .status-grid { grid-template-columns: 1fr; gap: 15px; } }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h2>💖 追剧管家 V7.5 <span style="font-size:12px; color:#8b5cf6;">6路火力全开版</span></h2>
            <span class="badge">SYSTEM ONLINE</span>
        </div>
        {% if msg %}<div class="status-msg">{{ msg }}</div>{% endif %}
        <div class="mac-window">
            <div class="mac-header">
                <div class="mac-btn btn-close"></div><div class="mac-btn btn-min"></div><div class="mac-btn btn-max"></div>
                <div class="mac-title">追剧控制台 - 实时运行日志</div>
            </div>
            <div class="console" id="logBox">Loading terminal...</div>
        </div>
        <div class="card">
            <h3>📊 系统运行状态 & 凭证雷达监控</h3>
            <div class="status-grid">
                <p style="color:#64748b; margin:0;">存活缓存视频：<br><b style="color:#1e293b; font-size:1.8rem;">{{ cache_count }}</b></p>
                <div style="color:#64748b; margin:0; font-size: 13px; line-height: 1.8; border-left: 2px solid #e2e8f0; padding-left: 15px;">
                    <b>🔑 矩阵凭证空投雷达：</b><br>
                    {% for i in range(6) %}
                        {% set fc = cfg.family_clouds[i] if i < cfg.family_clouds|length else {} %}
                        {% set sk = fc.get('session_key', '') %}
                        卡槽 {{ i+1 }}: 
                        {% if sk %}<span style="color:#10b981; font-weight:bold;">已挂载 (尾号{{ sk[-4:] }})</span>
                        {% else %}<span style="color:#f43f5e; font-weight:bold;">等待获取凭证...</span>{% endif %}<br>
                    {% endfor %}
                </div>
                <div style="display:flex; flex-direction:column; gap:10px;">
                    <button onclick="syncOpenList()" class="btn-purple" style="height:45px;">🔄 触发全局同步</button>
                    <button onclick="clearAllCache()" style="background:#ef4444; color:white; border:none; border-radius:6px; height:45px; cursor:pointer; font-weight:bold; transition: 0.2s;" onmouseover="this.style.background='#dc2626'" onmouseout="this.style.background='#ef4444'">🗑️ 一键清空秒传 & 回收站</button>
                </div>
            </div>
        </div>
        <div class="card">
            <h3>⚙️ 多路家庭云配置中心</h3>
            <form method="POST" action="/admin/config">
                <div style="margin-bottom: 20px;">
                    <label>点播分发策略 (Load Balancing)</label>
                    <select name="cloud_strategy">
                        <option value="hash" {% if cfg.cloud_strategy == 'hash' %}selected{% endif %}>🔗 剧名哈希绑定 (仅前4槽参与)</option>
                        <option value="round_robin" {% if cfg.cloud_strategy == 'round_robin' %}selected{% endif %}>🔁 顺序轮询分发 (仅前4槽参与)</option>
                        <option value="primary" {% if cfg.cloud_strategy == 'primary' %}selected{% endif %}>🥇 仅卡槽1分发</option>
                        <option value="slot2" {% if cfg.cloud_strategy == 'slot2' %}selected{% endif %}>🥈 仅卡槽2分发</option>
                        <option value="slot3" {% if cfg.cloud_strategy == 'slot3' %}selected{% endif %}>🥉 仅卡槽3分发</option>
                        <option value="slot4" {% if cfg.cloud_strategy == 'slot4' %}selected{% endif %}>💎 仅卡槽4分发</option>
                        <option value="slot5" {% if cfg.cloud_strategy == 'slot5' %}selected{% endif %}>🚀 仅卡槽5分发 (>28G重装甲)</option>
                        <option value="slot6" {% if cfg.cloud_strategy == 'slot6' %}selected{% endif %}>🛸 仅卡槽6分发 (备用大爷号专属)</option>
                        <option value="random" {% if cfg.cloud_strategy == 'random' %}selected{% endif %}>🎲 完全随机分发 (仅前4槽参与)</option>
                    </select>
                </div>
                <div class="grid">
                    <div>
                        {% for i in range(6) %}
                        {% set fc = cfg.family_clouds[i] if i < cfg.family_clouds|length else {} %}
                        <div class="cloud-box">
                            <div class="cloud-title">📌 独立挂载槽位 {{ i + 1 }} {% if i == 5 %}(填账号则自愈，留空则由 auto_189 接管){% else %}(静默自愈){% endif %}</div>
                            <input type="text" name="fc_id_{{ i }}" value="{{ fc.get('family_id', '') }}" placeholder="Family ID (留空则禁用此槽)">
                            <input type="text" name="fc_fd_{{ i }}" value="{{ fc.get('hard_folder_id', '') }}" placeholder="Folder ID">
                            <div style="display:flex; gap:10px;">
                                <input type="text" name="fc_user_{{ i }}" value="{{ fc.get('username', '') }}" placeholder="天翼云账号">
                                <input type="password" name="fc_pwd_{{ i }}" value="{{ fc.get('password', '') }}" placeholder="天翼云密码">
                            </div>
                            <input type="hidden" name="fc_sk_{{ i }}" value="{{ fc.get('session_key', '') }}">
                            <input type="text" name="fc_prefix_{{ i }}" value="{{ fc.get('openlist_prefix', '') }}" placeholder="OpenList 专属播放前缀">
                            <input type="text" name="fc_mount_{{ i }}" value="{{ fc.get('openlist_mount_path', '') }}" placeholder="OpenList 专属挂载目录" style="margin-bottom:0;">
                        </div>
                        {% endfor %}
                    </div>
                    <div>
                        <h4 style="margin-top:0;">🌐 全局基础设置</h4>
                        <label>基础外网域名 (Server Host)</label>
                        <input type="text" name="server_host" value="{{ cfg.server_host }}" required>
                        <div style="display: flex; gap: 10px; margin-bottom: 15px;">
                            <div style="flex: 1;"><label>绝对销毁倒计时</label><input type="number" name="delete_delay" value="{{ cfg.delete_delay }}" style="margin-bottom:0;" required></div>
                            <div style="flex: 1;"><label>预加载长效护盾</label><input type="number" name="shield_delay" value="{{ cfg.shield_delay }}" style="margin-bottom:0;" required></div>
                        </div>
                        <label>本地 STRM 保存路径</label><input type="text" name="local_strm_dir" value="{{ cfg.local_strm_dir }}" required>
                        <label>云端转存 CAS 路径</label><input type="text" name="network_cas_path" value="{{ cfg.network_cas_path }}" required>
                        <h4 style="margin-top:20px;">🌐 OpenList 核心设置</h4>
                        <label>OpenList 接口地址 (Host)</label><input type="text" name="openlist_host" value="{{ cfg.openlist_host }}" required>
                        <label>OpenList 授权 Token</label><input type="password" name="openlist_token" value="{{ cfg.openlist_token }}" placeholder="填入有效Token">
                        <h4 style="margin-top:20px;">📱 消息推送设置</h4>
                        <label>PushPlus Token (微信推送)</label><input type="text" name="pushplus_token" value="{{ cfg.pushplus_token }}" placeholder="留空则不推送">
                        <label>Telegram Bot Token</label><input type="text" name="tg_bot_token" value="{{ cfg.tg_bot_token }}" placeholder="例: 123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11">
                        <label>Telegram Chat ID</label><input type="text" name="tg_chat_id" value="{{ cfg.tg_chat_id }}" placeholder="例: 123456789">
                    </div>
                </div>
                <button type="submit" style="width:100%; margin-top:15px; font-size:16px;">💾 写入配置并重启引擎矩阵</button>
            </form>
        </div><div style="height: 40px;"></div>
    </div>
    <script>
        function syncOpenList() { fetch('/api/sync').then(r => alert('同步指令已下发！请看上方日志。')); }
        function clearAllCache() { if(confirm('⚠️ 确定要清空吗？')) { fetch('/api/clear_all', {method: 'POST'}).then(r => alert('🚀 核弹已发射！')); } }
        function fetchLogs() {
            fetch('/admin/logs').then(r => r.json()).then(logs => {
                const box = document.getElementById('logBox');
                const oldScrollTop = box.scrollTop, oldScrollHeight = box.scrollHeight, clientHeight = box.clientHeight;
                box.innerHTML = logs.map(l => `<span class="log-time">[${l.time}]</span><span class="log-msg ${l.msg.includes('✅') ? 'log-SUCCESS' : 'log-' + l.level}">${l.msg}</span><br>`).join('');
                if (oldScrollHeight - clientHeight - oldScrollTop < 30) { box.scrollTop = box.scrollHeight; } else { box.scrollTop = oldScrollTop; }
            });
        }
        setInterval(fetchLogs, 2000); fetchLogs();
    </script>
</body>
</html>
"""

@app_main.route('/admin')
def admin_index():
    cfg = read_config()
    with cache_lock: count = len(upload_cache)
    return render_template_string(ADMIN_HTML, cfg=cfg, cache_count=count, msg=request.args.get('msg'))

@app_main.route('/admin/logs')
def get_logs(): return jsonify(list(log_buffer))

# ==========================================
# ☁️ 天翼云核心功能类
# ==========================================
class TianyiFinalUploader:
    def __init__(self):
        self.rsa_keys = {}
        self.session = requests.Session()

    def get_base_headers(self, session_key):
        return {'User-Agent': 'ecloud/10.2.1 (Windows NT 10.0; Win64; x64)', 'Cookie': f"SESSION_KEY={session_key}; cookieUserSession={session_key}", 'Accept': 'application/json;charset=UTF-8', 'clientType': 'TELEMAC'}

    def _random_string(self, length=16): return ''.join(random.choices('0123456789abcdef', k=length))
    def _get_timestamp(self): return str(int(time.time() * 1000))

    def _get_slice_size(self, file_size):
        try: size = int(file_size)
        except: return '10485760'  
        D = 10485760
        if size > D * 2 * 999: return str(max(math.ceil(size / 1999 / D), 5) * D)
        elif size > D * 999: return str(D * 2)
        return str(D)

    def get_rsa_key(self, session_key):
        if session_key in self.rsa_keys: return self.rsa_keys[session_key]
        url = f"https://cloud.189.cn/api/security/generateRsaKey.action?sessionKey={urllib.parse.quote(session_key)}"
        for _ in range(3):
            try:
                res = self.session.get(url, headers=self.get_base_headers(session_key), timeout=10).json()
                if 'pubKey' in res:
                    self.rsa_keys[session_key] = res
                    return res
                if str(res.get('res_code')) == '111' or 'Session' in str(res): raise Exception("公钥获取拦截_AUTH_FAIL")
            except Exception as e: 
                if "AUTH_FAIL" in str(e): raise e
            time.sleep(2)
        raise Exception("无法获取公钥_AUTH_FAIL")

    def build_request(self, params, request_uri, req_id, session_key):
        rsa_key = self.get_rsa_key(session_key)
        uuid_key = self._random_string(16)
        ts = self._get_timestamp()
        p_str = '&'.join([f"{k}={v}" for k, v in params.items()])
        cipher = AES.new(uuid_key.encode('utf-8'), AES.MODE_ECB)
        enc_p = cipher.encrypt(pad(p_str.encode('utf-8'), 16)).hex().upper()
        rsa_cipher = PKCS1_v1_5.new(RSA.import_key(f"-----BEGIN PUBLIC KEY-----\n{rsa_key['pubKey']}\n-----END PUBLIC KEY-----"))
        enc_t = base64.b64encode(rsa_cipher.encrypt(uuid_key.encode('utf-8'))).decode('utf-8')
        sign = hmac.new(uuid_key.encode('utf-8'), f"SessionKey={session_key}&Operate=GET&RequestURI={request_uri}&Date={ts}&params={enc_p}".encode('utf-8'), hashlib.sha1).hexdigest().upper()
        h = self.get_base_headers(session_key)
        h.update({'X-Request-Date': ts, 'X-Request-ID': req_id, 'SessionKey': session_key, 'EncryptionText': enc_t, 'PkId': rsa_key['pkId'], 'Signature': sign})
        return f"https://upload.cloud.189.cn{request_uri}?params={enc_p}", h

    def get_family_items(self, family_id, folder_id, session_key):
        all_items = []
        params = {"familyId": family_id, "folderId": folder_id, "pageSize": 60, "sessionKey": session_key}
        res = self.session.get("https://cloud.189.cn/api/open/family/file/listFiles.action", params=params, headers=self.get_base_headers(session_key), timeout=10).json()
        if str(res.get('res_code')) == '111' or 'Session' in str(res): raise Exception("接口返回111_AUTH_FAIL")
        for f in res.get('fileListAO', {}).get('fileList', []): all_items.append({'fileName': f['name'], 'fileId': f['id']})
        return all_items

    def delete_item(self, family_id, file_id, session_key):
        url = "https://cloud.189.cn/api/open/family/file/deleteFile.action"
        p = {"familyId": family_id, "fileId": file_id, "sessionKey": session_key}
        try: return self.session.post(url, params=p, headers=self.get_base_headers(session_key), timeout=10).status_code == 200
        except: return False

    def empty_family_recycle(self, family_id, session_key):
        url = "https://cloud.189.cn/api/open/batch/createBatchTask.action"
        payload = {"type": "EMPTY_RECYCLE", "taskInfos": "[]", "targetFolderId": "", "familyId": family_id, "sessionKey": session_key}
        try:
            res = self.session.post(url, data=payload, headers=self.get_base_headers(session_key), timeout=10).json()
            if str(res.get("res_code")) == "0": return True
        except: pass
        return False

    def rapid_upload(self, family_id, parent_folder_id, md5, size, smd5, safe_name, session_key):
        req_id = str(uuid.uuid4())
        slice_size = self._get_slice_size(size)  
        init_p = {'familyId': family_id, 'parentFolderId': parent_folder_id, 'fileName': urllib.parse.quote(safe_name), 'fileSize': str(size), 'sliceSize': slice_size, 'fileMd5': md5, 'sliceMd5': smd5, 'lazyCheck': '1', 'opertype': '3'}
        url, h = self.build_request(init_p, '/family/initMultiUpload', req_id, session_key)
        res = self.session.get(url, headers=h).json()
        if res.get('code') != 'SUCCESS': 
            msg_str = str(res.get('msg', ''))
            if any(k in msg_str.lower() for k in ['session', 'privatekey', '111']): raise Exception(f"秒传初始化拒绝_AUTH_FAIL: {msg_str}")
            raise Exception(f"秒传初始化失败: {msg_str}")
        
        up_id = res['data']['uploadFileId']
        ck_p = {'familyId': family_id, 'fileMd5': md5, 'sliceMd5': smd5, 'uploadFileId': up_id}
        url, h = self.build_request(ck_p, '/family/checkTransSecond', req_id, session_key)
        if not self.session.get(url, headers=h).json().get('data', {}).get('fileDataExists'): raise Exception("云端无此文件")
        
        cm_p = {'familyId': family_id, 'uploadFileId': up_id, 'fileMd5': md5, 'sliceMd5': smd5, 'lazyCheck': '1', 'opertype': '3'}
        url, h = self.build_request(cm_p, '/family/commitMultiUploadFile', req_id, session_key)
        commit_res = self.session.get(url, headers=h).json()
        file_info = commit_res.get('file')
        if not file_info: raise Exception(f"秒传确认失败: {commit_res.get('msg', '未知错误')}")
        return file_info['userFileId']
            
client = TianyiFinalUploader()

def cleanup_worker(name, f_md5, fam_id, fold_id, session_key):
    with cache_lock:
        if f_md5 not in upload_cache: return
        expire_time = upload_cache[f_md5]['expire']
        
    expire_str = time.strftime("%H:%M:%S", time.localtime(expire_time))
    logger.info(f"⏳ 销毁启动: [{name}] 预计于 {expire_str} 删除")
    
    while True:
        with cache_lock:
            if f_md5 not in upload_cache: return
            expire_time = upload_cache[f_md5]['expire']
            
        now = time.time()
        if now >= expire_time: break
        
        sleep_time = expire_time - now
        if sleep_time > 0:
            time.sleep(sleep_time + 1)
        
    logger.info(f"🗑️ 视频删除: [{name}]")

    try:
        # --- 原版正常删除逻辑 ---
        items = client.get_family_items(fam_id, fold_id, session_key)
        real_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == name), None)
        
        deleted = False
        if real_fid and client.delete_item(fam_id, real_fid, session_key):
            deleted = True
            time.sleep(2) 
            if client.empty_family_recycle(fam_id, session_key):
                logger.info(f"♻️ 清空回收站: 存储卡槽 [{fam_id[-4:]}] 空间已释放")
                
        # --- 方案一：如果正常删除没成功，调出卡槽 6 的钥匙兜底 ---
        if not deleted:
            logger.warning(f"⚠️ 常规删除受阻，触发兜底: 尝试调用卡槽 6 凭证强删！")
            cfg = read_config()
            if len(cfg.get('family_clouds', [])) > 5 and cfg['family_clouds'][5].get('session_key'):
                master_sk = cfg['family_clouds'][5]['session_key']
                
                # 用卡槽 6 的钥匙重新查一次
                master_items = client.get_family_items(fam_id, fold_id, master_sk)
                master_fid = next((i['fileId'] for i in master_items if f_md5 in i['fileName'] or i['fileName'] == name), None)
                
                # 用卡槽 6 的钥匙去删和清空回收站
                if master_fid and client.delete_item(fam_id, master_fid, master_sk):
                    logger.info(f"✅ 兜底强删成功: 已使用卡槽 6 凭证销毁 [{name}]")
                    time.sleep(2)
                    if client.empty_family_recycle(fam_id, master_sk):
                        logger.info(f"♻️ 借用主号权限清空回收站完毕。")
                else:
                    logger.error(f"❌ 兜底失败：卡槽 6 凭证也无法删除该文件。")
            else:
                logger.warning(f"❌ 无法执行兜底删除：卡槽 6 未配置或凭证无效。")

    except Exception as e:
        logger.error(f"🗑️ 后台删除作业受阻: {e}")
        
    with cache_lock:
        if f_md5 in upload_cache: del upload_cache[f_md5]

@app_main.route('/play', methods=['GET', 'HEAD'])
def play():
    if request.method == 'HEAD': 
        try:
            cas = request.args.get('cas')
            j = json.loads(base64.b64decode(cas).decode())
            
            raw_size = j.get('size') or j.get('fileSize') or 0
            name = j.get('name') or j.get('fileName') or ""
            ext = name.split('.')[-1].lower() if '.' in name else 'mp4'
            
            mime_dict = {
                'mp4': 'video/mp4', 'mkv': 'video/x-matroska', 'ts':  'video/mp2t',
                'avi': 'video/x-msvideo', 'mov': 'video/quicktime', 'webm':'video/webm'
            }
            content_type = mime_dict.get(ext, 'video/mp4')

            return "", 200, {
                'Content-Type': content_type, 
                'Accept-Ranges': 'bytes',
                'Content-Length': str(raw_size)
            }
        except:
            return "", 200, {'Content-Type': 'video/mp4', 'Accept-Ranges': 'bytes'}

    cas = request.args.get('cas')
    show_name_from_url = request.args.get('show', '').strip()
    safe_name = "未知文件"
    try:
        j = json.loads(base64.b64decode(cas).decode())
        f_md5 = str(j.get('md5') or j.get('fileMd5')).upper()
        s_md5 = str(j.get('slice_md5') or j.get('sliceMd5')).upper()
        raw_size = j.get('size') or j.get('fileSize')
        human_size = format_size(raw_size)
        name = j.get('name') or j.get('fileName')
        base_safe_name = "".join(x for x in name if x not in r'\/:*?"<>|')
        
        if show_name_from_url: show_identifier = re.sub(r'\s*\(\d{4}\)$', '', show_name_from_url).strip()
        else:
            guess = re.split(r'(?i)\.S\d+|\.E\d+|-第\d+集', base_safe_name)[0]
            show_identifier = re.sub(r'\s*\(\d{4}\)$', '', guess).strip()
            
        bind_key = show_identifier
        ext = os.path.splitext(base_safe_name)[1]
        if not ext or len(ext) > 6: ext = ".mp4"

        ep_num = None
        for p in [r'(?i)E(?:P)?\s*0*(\d+)', r'第\s*0*(\d+)\s*[集话期]', r'(?:\[|\()0*(\d+)(?:\]|\))', r'(?i)episode\s*0*(\d+)']:
            m = re.search(p, base_safe_name)
            if m: 
                ep_num = int(m.group(1))
                break
        s_match = re.search(r'(?i)S0*(\d+)', base_safe_name)
        s_num = int(s_match.group(1)) if s_match else 1

        if show_identifier and ep_num is not None: safe_name = f"{show_identifier}.S{s_num:02d}E{ep_num:02d}{ext}"
        else:
            year_match = re.search(r'(?<!\d)(19\d{2}|20\d{2})(?!\d)', base_safe_name)
            year_str = f".{year_match.group(1)}" if year_match else ""
            if show_identifier: safe_name = f"{show_identifier}{year_str}{ext}"
            else: safe_name = base_safe_name

        tags = []
        if re.search(r'(?i)\bSDR\b', base_safe_name): tags.append("SDR")
        if re.search(r'(?i)\bHDR\b', base_safe_name): tags.append("HDR")
        if re.search(r'(?i)\bHQ\b', base_safe_name): tags.append("HQ")
        if re.search(r'(?i)\bDV\b', base_safe_name): tags.append("DV")
        tag_str = "." + ".".join(tags) if tags else ""
        
        if tag_str:
            if safe_name.endswith(ext): safe_name = safe_name[:-len(ext)] + tag_str + ext
            else: safe_name = safe_name + tag_str + ext
                
        cfg = read_config()
        current_time = time.time()
        fam_fid, target_fam_id, target_fold_id, target_session_key, target_prefix, target_mount, is_new, requires_cleanup = None, None, None, None, None, None, False, False
        s_idx = 0

        with cache_lock:
            if f_md5 in upload_cache:
                target_prefix = upload_cache[f_md5]['prefix']
                
                # 🔪 砍掉互相加持的毒瘤逻辑！维持原判，各自安好！
                
                last_log_time = upload_cache[f_md5].get('last_log', 0)
                if current_time - last_log_time > 60: 
                    expire_self_str = time.strftime("%H:%M:%S", time.localtime(upload_cache[f_md5]['expire']))
                    logger.info(f"📡 探针侦测: [{safe_name}] 预计于 {expire_self_str} 准时处决！")
                    upload_cache[f_md5]['last_log'] = current_time
            else:
                cloud, s_idx = get_target_cloud(cfg, bind_key, file_size=raw_size)
                if not cloud: raise Exception("未配置有效的家庭云阵列！请检查后台。")
                target_fam_id = cloud['family_id']
                target_fold_id = cloud['hard_folder_id']
                target_session_key = cloud['session_key']
                target_prefix = cloud['openlist_prefix']
                target_mount = cloud['openlist_mount_path']

                brother_exists = any(v.get('show_name') == bind_key for v in upload_cache.values())
                if brother_exists: initial_expire = current_time + cfg.get('shield_delay', 2700)
                else: initial_expire = current_time + cfg.get('delete_delay', 600)

                upload_cache[f_md5] = {'fid': 'processing', 'expire': initial_expire, 'fam_id': target_fam_id, 'fold_id': target_fold_id, 'session_key': target_session_key, 'prefix': target_prefix, 'mount': target_mount, 'show_name': bind_key}
                requires_cleanup = True
                
        if requires_cleanup:
            try:
                if not target_session_key: raise Exception("AUTH_FAIL: 本地凭证为空，强制触发自愈流程")
                items = client.get_family_items(target_fam_id, target_fold_id, target_session_key)
                fam_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                if not fam_fid:
                    # 🚀 修复日志提示：打印真实的卡槽索引
                    logger.info(f"🔄 路由调度: [{bind_key}] ({human_size}) -> 卡槽 {s_idx+1}")
                    fam_fid = client.rapid_upload(target_fam_id, target_fold_id, f_md5, raw_size, s_md5, safe_name, target_session_key)
                    is_new = True
            except Exception as e:
                err_str = str(e).lower()
                if any(k in err_str for k in ["auth_fail", "invalidsessionkey", "sessionkey", "session", "111", "privatekey"]):
                    logger.warning(f"⚠️ 探测到卡槽 {s_idx+1} 凭证失效(或私钥丢失)，引擎介入执行自愈...")
                    new_sk = refresh_slot_logic(s_idx, cfg)
                    if new_sk:
                        target_session_key = new_sk
                        with cache_lock: upload_cache[f_md5]['session_key'] = new_sk
                        items = client.get_family_items(target_fam_id, target_fold_id, target_session_key)
                        fam_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                        if not fam_fid:
                            fam_fid = client.rapid_upload(target_fam_id, target_fold_id, f_md5, raw_size, s_md5, safe_name, target_session_key)
                            is_new = True
                    else:
                        if s_idx == 5 and not cfg['family_clouds'][5].get('username'):
                            logger.warning("🔄 卡槽 6 外部凭证失效，正在触发【无缝故障转移】至主卡槽(卡槽1)...")
                            if len(cfg['family_clouds']) > 0 and cfg['family_clouds'][0].get('session_key'):
                                target_fam_id = cfg['family_clouds'][0]['family_id']
                                target_fold_id = cfg['family_clouds'][0]['hard_folder_id']
                                target_session_key = cfg['family_clouds'][0]['session_key']
                                target_prefix = cfg['family_clouds'][0]['openlist_prefix']
                                target_mount = cfg['family_clouds'][0]['openlist_mount_path']
                                with cache_lock:
                                    upload_cache[f_md5].update({'fam_id': target_fam_id, 'fold_id': target_fold_id, 'session_key': target_session_key, 'prefix': target_prefix, 'mount': target_mount})
                                items = client.get_family_items(target_fam_id, target_fold_id, target_session_key)
                                fam_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                                if not fam_fid:
                                    fam_fid = client.rapid_upload(target_fam_id, target_fold_id, f_md5, raw_size, s_md5, safe_name, target_session_key)
                                    is_new = True
                            else: raise Exception("卡槽 6 无法恢复，且主卡槽(1)不可用，降级转移彻底失败！")
                        else: raise Exception(f"卡槽 {s_idx+1} 防线自愈失败，无法重建有效凭证！")
                else: raise e
                
            if fam_fid:
                if is_new:
                    if brother_exists: logger.info(f"✅ 秒传成功: [{safe_name}] ({human_size}) (被动预加载，已天生加持长效护盾)")
                    else: 
                        # 🚀 修复日志提示：精准播报具体卡槽
                        logger.info(f"✅ 秒传成功: [{safe_name}] ({human_size}) 已利用 卡槽 {s_idx+1} 上传！")
                        send_push("▶️ 新剧集入库并播放", f"<b>{safe_name}</b> ({human_size})<br>已成功利用 <b>卡槽 {s_idx+1}</b> 秒传并启动播放通道。")
                with cache_lock: upload_cache[f_md5]['fid'] = fam_fid
                threading.Thread(target=cleanup_worker, args=(safe_name, f_md5, target_fam_id, target_fold_id, target_session_key), daemon=True).start()
        
        # === 替换 play 函数底部的 if is_new: 以及之后的部分 ===
        
        if is_new:
            logger.info(f"🔔 启动 OpenList 智能嗅探: 等待天翼云底层数据同步...")
            found = False
            # ⚡ 极限轮询：最多查 4 次（最多约 0.8秒），只要查到文件立刻放行，绝不多等 1 毫秒！
            for step in range(4):
                time.sleep(0.2)
                try:
                    if cfg['openlist_token'] and target_mount: 
                        res = requests.post(f"{cfg['openlist_host']}/api/fs/list", json={"path": target_mount, "refresh": True}, headers={"Authorization": cfg['openlist_token']}, timeout=3).json()
                        if res.get('code') == 200:
                            # 把 OpenList 当前目录下的所有文件名扫一遍
                            files = [f.get('name') for f in res.get('data', {}).get('content', [])]
                            # 如果咱们刚传的文件已经冒头了
                            if safe_name in files:
                                logger.info(f"✅ 第 {step+1} 次嗅探成功！文件已在 OpenList 极速就绪！")
                                found = True
                                break
                except Exception as e:
                    pass
                
            if not found:
                logger.warning(f"⚠️ 嗅探结束，天翼云底层同步严重延迟，强行放行赌一把！")

        # 👇 新增：内网直链解析
        lan_ip = get_lan_server_ip(request)
        if lan_ip:
            parsed_prefix = urllib.parse.urlparse(target_prefix)
            # 剥离外网穿透，强制替换为内网 openlist 专属的 5244 访问端口
            local_alist_host = f"http://{lan_ip}:5244"
            target_prefix = target_prefix.replace(f"{parsed_prefix.scheme}://{parsed_prefix.netloc}", local_alist_host)
            logger.info(f"🏠 [内网嗅探] 真实流媒体通道已切换为内网直连: {local_alist_host}")
        # 👆 新增结束

        # 🎯 无论新旧文件，全部老老实实走 OpenList 通道！
        logger.info(f"🔄 路由导向: [{safe_name}] 已移交 OpenList 播放通道！")
        return redirect(f"{target_prefix.rstrip('/')}/{urllib.parse.quote(safe_name)}", code=302)
        
    except Exception as e:
        logger.error(f"❌ 链路故障: {e}")
        with cache_lock:
            if 'f_md5' in locals() and f_md5 in upload_cache and upload_cache[f_md5].get('fid') == 'processing':
                del upload_cache[f_md5]
                logger.info("🧹 已清除因报错假死的转存缓存，通道已重新释放！")
        if "云端无此文件" in str(e) or "秒传初始化失败" in str(e) or "AUTH_FAIL" in str(e) or "privatekey" in str(e).lower():
            send_push("💔 播放链路故障", f"点播 <b>{safe_name}</b> 失败！<br>原因: {e}")
        return f"错误: {e}", 500

def warm_up_parent(target_path, headers, cfg):
    if not target_path: return
    base_path = cfg.get('network_cas_path', '').rstrip('/')
    if target_path.startswith(base_path):
        rel_path = target_path[len(base_path):].strip('/')
        parts = rel_path.split('/')
        current_path = base_path
        for part in parts[:-1]:
            current_path = f"{current_path}/{part}"
            logger.info(f"🧊 破冰行动：逐级向下唤醒缓存 -> {current_path}")
            try:
                requests.post(f"{cfg['openlist_host']}/api/fs/list", json={"path": current_path, "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=5)
                time.sleep(0.5)
            except: pass

def scan_openlist_recursive(current_path, headers, result_list, cfg):
    logger.info(f"🔎 正在探测目录: {current_path}")
    try:
        req_headers = headers if cfg['openlist_token'] else {}
        res = requests.post(f"{cfg['openlist_host']}/api/fs/list", json={"path": current_path, "page": 1, "per_page": 1000, "refresh": True}, headers=req_headers, timeout=15).json()
        if res.get("code") != 200: 
            logger.error(f"❌ OpenList 扫描拒绝: {res.get('message')}")
            return
        for f in res.get("data", {}).get("content", []):
            if f.get("is_dir"): scan_openlist_recursive(f"{current_path}/{f['name']}", headers, result_list, cfg)
            elif f['name'].endswith('.cas'): result_list.append(f"{current_path}/{f['name']}")
    except Exception as e: logger.error(f"❌ 探测目录失败 {current_path}: {e}")

def generate_strm_from_openlist_to_local(target_path=None):
    cfg = read_config()
    scan_root = target_path if target_path else cfg['network_cas_path']
    base_cas_path = cfg['network_cas_path']
    os.makedirs(cfg['local_strm_dir'], exist_ok=True)
    headers = {"Authorization": cfg['openlist_token']} if cfg['openlist_token'] else {}
    if target_path: warm_up_parent(target_path, headers, cfg)
    
    logger.info(f"🔄 启动 OpenList 扫描 -> 目标区域: {scan_root}")
    cas_files = []
    scan_openlist_recursive(scan_root, headers, cas_files, cfg)
    if not cas_files: return logger.info(f"⚠️ 扫描完毕：该区域下未找到任何 .cas 文件")
        
    count = 0
    # 🌟 依然保留 Session 复用，这能省下大量的 TCP 握手时间，本身就能提速
    req_session = requests.Session() 
    
    for full_path in cas_files:
        try:
            if full_path.startswith(base_cas_path): rel_path = full_path[len(base_cas_path):].lstrip('/')
            else: rel_path = full_path.split('/')[-1]
            rel_dir = os.path.dirname(rel_path)
            dir_parts = [p for p in rel_dir.split('/') if p]
            show_name = ""
            for part in reversed(dir_parts):
                if not re.match(r'(?i)^(season\s*\d+|specials|电视剧|电影|动漫|纪录片|综艺)$', part):
                    show_name = part; break
            if not show_name and dir_parts: show_name = dir_parts[-1] 
            if not show_name: show_name = "未知剧集"
            show_name = re.sub(r'\s*\(\d{4}\)$', '', show_name).strip()
            base_name = os.path.basename(rel_path).rsplit('.', 1)[0]
            
            target_local_dir = os.path.join(cfg['local_strm_dir'], rel_dir)
            os.makedirs(target_local_dir, exist_ok=True)
            strm_path = os.path.join(target_local_dir, f"{base_name}.strm")
            if os.path.exists(strm_path): continue
            
            # 🌟 核心智能引擎：普通剧集全速狂飙，遇到超长篇才动态错峰！
            # 每成功拉取 50 个文件，强制让网盘接口休息 3 秒，清空天翼云的频率阻断计数器
            if count > 0 and count % 50 == 0:
                logger.info(f"🚦 [智能防刷] 已极速突发拉取 {count} 集，触发冷却机制，让接口喘息 3 秒...")
                time.sleep(3)
                
            get_res = req_session.post(f"{cfg['openlist_host']}/api/fs/get", json={"path": full_path}, headers=headers, timeout=10).json()
            raw_url = get_res.get("data", {}).get("raw_url")
            if not raw_url: continue
            
            cas_content = req_session.get(raw_url, timeout=10).text.strip()
            strm_data = f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            
            with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)
            count += 1
            
        except Exception as e:
            logger.error(f"❌ 读取 {base_name} 时受阻: {e}")
            # 遇到真实的风控拦截或报错，才老老实实低头认怂休眠一下
            time.sleep(2) 
            pass

    if count > 0: 
        logger.info(f"🎉 同步完毕，成功生成 {count} 个带精准剧名标记的 STRM 文件")
        try: 
            subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", scan_root])
            logger.info("🎬 已触发本地 Emby 媒体库刷新指令")
            send_push("🎬 STRM同步成功", f"成功生成并归档了 {count} 个媒体文件，并已触发媒体库刷新。")
        except: pass

@app_main.route('/api/sync')
def trigger_sync():
    target_path = request.args.get('path') 
    threading.Thread(target=generate_strm_from_openlist_to_local, args=(target_path,), daemon=True).start()
    return "✅ 同步指令下发成功", 200

# ==========================================
# 🔀 独立服务二号头：5001 端口专属 302 劫持 (完全迎合旧 Nginx)
# ==========================================
emby_session = requests.Session()

@functools.lru_cache(maxsize=256)
def get_emby_item_path(item_id):
    try:
        url = f"{EMBY_HOST}/emby/Items?Ids={item_id}&Fields=Path&api_key={API_KEY_LINUX}"
        res = emby_session.get(url, timeout=3)
        if res.status_code == 200:
            items = res.json().get('Items', [])
            if items: return items[0].get('Path', ''), "Linux(主力)"
    except: pass
    try:
        url = f"{EMBY_HOST}/emby/Items?Ids={item_id}&Fields=Path&api_key={API_KEY_APP}"
        res = emby_session.get(url, timeout=3)
        if res.status_code == 200:
            items = res.json().get('Items', [])
            if items: return items[0].get('Path', ''), "APP(备用)"
    except: pass
    return None, None

@app_302.route('/', defaults={'path': ''}, methods=['GET', 'HEAD', 'POST', 'OPTIONS'])
@app_302.route('/<path:full_path>', methods=['GET', 'HEAD', 'POST', 'OPTIONS'])
def catch_all_for_emby(full_path):
    # 🚀 扩大雷达：把 Vidhub、Infuse 爱用的 Download 路径全部罩进去！
    # （注：不要拦截 PlaybackInfo，因为那是 Emby 客户端要 JSON 数据的，强行 302 会报错）
    match = re.search(r'/(?:videos|Items)/(\d+)/(?:stream|original|Download)', request.path, re.IGNORECASE)
    
    if not match:
        return redirect(f"{EMBY_HOST}{request.full_path}", code=302)
    item_id = match.group(1)
    
    try:
        file_path, version = get_emby_item_path(item_id)
        if not file_path: return redirect(f"{EMBY_HOST}{request.full_path}", code=302)
        file_name = file_path.split('/')[-1] if file_path else "未知文件"
        if file_path.lower().endswith('.strm') and os.path.exists(file_path):
                with open(file_path, 'r', encoding='utf-8') as f: strm_url = f.read().strip()
                
                # 👇 新增：内网智能分流
                cfg = read_config()
                lan_ip = get_lan_server_ip(request)
                if lan_ip:
                    # 检测到内网访问，将 STRM 里的外网 host 动态替换为当前 Termux 的局域网 IP + 5000 重定向端口
                    local_cas_host = f"http://{lan_ip}:5000"
                    strm_url = strm_url.replace(cfg['server_host'], local_cas_host)
                    logger.info(f"🏠 [内网嗅探] 识别到局域网播放，路由已切至: {local_cas_host}")
                # 👆 新增结束
                
                logger.info(f"✅ 🔺[{version}劫持成功] Player -> {file_name}")
                return redirect(strm_url, code=302)
        else:
            logger.info(f"▶️ 🔻[{version}常规播放] 丢回Emby -> {file_name}")
            return redirect(f"{EMBY_HOST}{request.full_path}", code=302)
    except Exception as e:
        logger.error(f"❌ [劫持出错] 兜底放行: {e}")
        return redirect(f"{EMBY_HOST}{request.full_path}", code=302)

# ==========================================
# 🚀 引擎启动：双线程齐发
# ==========================================
def run_main():
    # 改为 '::' 监听所有 IPv4 和 IPv6，配合上面的智能锁，局域网全通！
    app_main.run(host='::', port=5000, use_reloader=False)

def run_302():
    app_302.run(host='::', port=5001, use_reloader=False)

# ==========================================
# 🛡️ 后台错峰保活机制 (防风控打更人)
# ==========================================
def keep_alive_worker():
    logger.info("🛡️ 后台错峰保活机制已启动，将在后台默默守护你的天翼云 Key...")
    # 刚启动时别急着刷，先等 5 分钟，让主程序安稳落地
    time.sleep(600)
    
    while True:
        try:
            cfg = read_config()
            clouds = cfg.get('family_clouds', [])
            
            for i, fc in enumerate(clouds):
                # 💥 核心防御 1：跳过无账号密码的槽位（避开外部同步卡槽和废弃卡槽）
                if not fc.get('username') or not fc.get('password'):
                    continue
                
                sk = fc.get('session_key')
                fam_id = fc.get('family_id')
                fold_id = fc.get('hard_folder_id')
                
                logger.info(f"🛡️ [保活巡更] 正在探测 卡槽 {i+1} 凭证健康度...")
                
                is_alive = False
                if sk and fam_id and fold_id:
                    try:
                        # 🎯 智能探针：用现有的 Key 发起一次极小代价的目录查询
                        # 如果没有报错，说明 Key 是活的，并且这次请求也起到了防止服务器休眠的作用
                        client.get_family_items(fam_id, fold_id, sk)
                        is_alive = True
                        logger.info(f"✨ 卡槽 {i+1} 凭证状态极其健康，无需重登！(已完成触碰保活)")
                    except Exception as e:
                        err_str = str(e).lower()
                        if any(k in err_str for k in ["auth_fail", "111", "session"]):
                            # 明确是被踢下线或过期了
                            is_alive = False 
                        else:
                            # 可能是网络波动、请求超时等，暂定存活，绝不盲目重登
                            is_alive = True
                            logger.warning(f"⚠️ 卡槽 {i+1} 探测遇到网络波动，跳过刷新: {e}")
                
                # 只有明确探测到 Key 死透了，或者压根就没有 Key，才触发自愈重登
                if not is_alive:
                    refresh_slot_logic(i, cfg)
                
                # 💥 核心防御 2：绝对错峰！刷完一个号，强行随机休眠 10 到 15 分钟！
                sleep_time = random.randint(600, 900)
                logger.info(f"💤 [防风控] 卡槽 {i+1} 巡检完毕，打更人休眠 {sleep_time} 秒后去下一个卡槽...")
                time.sleep(sleep_time)

            logger.info("✅ 本轮打更巡检完成！所有主力卡槽凭证已全员满血！")
        except Exception as e:
            logger.error(f"❌ 保活线程遇到意外: {e}")

        # 整体大循环间隔：每隔 120 分钟（7200秒）启动下一轮大巡检
        logger.info("⏳ 打更人进入深度睡眠，120 分钟后开启下一轮巡视。")
        time.sleep(7200)

if __name__ == '__main__':
    logger.info("✅ 🚦 真·双头蛇引擎启动！(5000端口负责直链/后台 | 5001端口负责Emby劫持)")
    
    # 🌟 把保活打更人作为守护线程放出去跑
    threading.Thread(target=keep_alive_worker, daemon=True).start()
    
    t1 = threading.Thread(target=run_main)
    t2 = threading.Thread(target=run_302)
    
    t1.start()
    t2.start()
    
    t1.join()
    t2.join()
```
2.pm2启动

```
cd ~/189py && pm2 start casplay.py --name "casplay" --interpreter python
```

### 四、添加机器人指令
1.打开BotFather机器人

2.发指令/setcommands

3.选择自己的机器人

4.粘贴如下内容：
```
sub - 📥 [订阅/绑定] 绑定外部链接追剧
harvest - 🚜 [收割/处理/添加] 洗名并入库CAS文件
feed - 📡 [动态/广场] 订阅中心最新情报
search - 🔍 [搜 关键词] 穿甲雷达搜索
check - 🔍 [查 剧名] 剧名查找
author - 🕵️‍♂️ [查作者\查人] 大佬真实时间线
refresh- 🔄 [刷新\入库] 刷新入库某剧
sync - 🔄 [同步订阅] 强制检查所有更新
list - 📋 [列表] 查看当前追剧清单
ascan - ✔️ [开启自动收割] 自动收割扫描
sscan - ⭕️ [关闭自动收割] 关闭收割扫描
asub - ✅ [开启订阅检查] 开启订阅检查
ssub - ❎ [关闭订阅检查] 关闭订阅检查
ldir - 🔍 [查目录] 查看收割目录
adir - ➕ [加目录] 增加收割目录
hsub - ➕ [加库] 增加收割入库记录
dsub - ❌ [删库] 删除收割入库记录
dsub - 🔍 [查库] 查看收割入库记录
```