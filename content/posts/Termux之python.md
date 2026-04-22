---

title: "Termux之python"

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

### 二、自动转存auto_189

1.建立专属工作台与配置文件
```
# 1. 创建专属文件夹并进去

mkdir -p ~/tcloud/db
cd ~/tcloud

# 2. 创建环境变量文件并编辑
nano sys.env
```
执行完 nano sys.env 后，屏幕会变成黑底白字的编辑器。把下面这段内容修改成你自己的真实信息后，粘贴进去（注意等号两边不要有空格）：
```
# 你的天翼云盘账号和密码
ENV_189_CLIENT_ID=13800138000
ENV_189_CLIENT_SECRET=你的天翼密码

# 你的 TG 机器人配置
ENV_TG_BOT_TOKEN=123456789:ABCDEFGHIJKLMNOPQRSTUVWXYZ
ENV_TG_ADMIN_USER_ID=你的TG数字ID
```
填完后，按 Ctrl + O（字母O），回车保存；然后按 Ctrl + X 退出。

3.auto_189.py脚本

3.1在电脑中新建文件auto_189.py内容如下：
```
import os
import json
import time
import requests
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

# 🛡️ 强制锁死 IPv4
old_getaddrinfo = socket.getaddrinfo
def new_getaddrinfo(*args, **kwargs):
    responses = old_getaddrinfo(*args, **kwargs)
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
# 📁 核心目录与挂载配置
# ==========================================
DIR_CAS_ROOT = "/177-秒传"
DIR_VIDEO_ROOT = "/177-视频"    # 🌟 新增：新版普通视频的专属智能路由根目录      
DIR_MEDIA_PREFIX = "/177-"      
OPENLIST_MOUNT_POINT = "177"    

SUBS_FILE = os.path.join(DB_DIR, "subscriptions.json")
HISTORY_FILE = os.path.join(DB_DIR, "history.json")
COOKIES_FILE = os.path.join(DB_DIR, "cookies.json")

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

    if "电影" in sub_path or "movie" in sub_path.lower():
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

def check_subscriptions(client_obj, force_target_id=None, is_first_run=False):
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

            if not force_target_id and not is_first_run:
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
                # 🎯 进阶集数去重逻辑
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
                success_count = 0
                renamed_count = 0
                
                code = info.saveShareFiles(taskInfos, target_id)
                
                if code in [0, '0', None, False, '']:
                    success_count = len(taskInfos)
                    for task in taskInfos:
                        history[str(task["fileId"])] = {"name": task["fileName"], "sub_id": str(target_id)}
                else:
                    notifier.send_message(f"❌ 批次转存被天翼云拦截！错误码: {code}")

                if success_count > 0:
                    save_json(HISTORY_FILE, history)
                    if isinstance(subs.get(str(target_id)), dict):
                        now_dt = datetime.now()
                        subs[str(target_id)]["last_update"] = int(time.time())
                        if freq == "周更" or "周更" in path or "动漫" in path:
                            subs[str(target_id)]["last_success_week"] = now_dt.strftime("%Y-%V")
                            subs[str(target_id)]["last_success_day"] = now_dt.strftime("%Y-%m-%d")
                        save_json(SUBS_FILE, subs)
                        
                    time.sleep(8) 
                    
                    fresh_cloud_files = client_obj.listPrivateFiles(target_id)
                    for cf in fresh_cloud_files:
                        hist_info = history.get(str(cf["id"]))
                        orig_name = hist_info["name"] if hist_info else cf["name"]
                        new_name = generate_smart_name(orig_name, path)
                        
                        if new_name and cf["name"] != new_name:
                            if client_obj.renameFile(cf["id"], new_name):
                                renamed_count += 1
                                time.sleep(0.5) 
                                
                    notifier.send_message(f"✅【追剧/补档更新】\n🔗 来源: {share_url}\n📂 成功抓取 {success_count} 个文件！")
                    if renamed_count > 0:
                        logger.info(f"✨ [刮削] 云端智能命名执行完毕，规范化 {renamed_count} 个文件")
                        notifier.send_message(f"✨ 云端强制更名: 已规范化 {renamed_count} 个文件！")

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
            elif any(kw in error_msg for kw in ["掉线", "失败", "拦截", "风控"]): 
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
            subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", p], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
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
        logger.info("==========================================")
        logger.info("🛸 [系统] 定时唤醒：引擎升空，接管侦测作业...")
        check_subscriptions(client_obj)
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
                                # 非周更，直接跳到分类选项
                                kb = {"inline_keyboard": [
                                    [{"text": "📺 华语剧", "callback_data": "wiz_cat_华语剧"}, {"text": "📺 欧美剧", "callback_data": "wiz_cat_欧美剧"}],
                                    [{"text": "🎬 华语电影", "callback_data": "wiz_cat_华语电影"}, {"text": "🎬 欧美电影", "callback_data": "wiz_cat_欧美电影"}],
                                    [{"text": "🐼 日漫/番剧", "callback_data": "wiz_cat_日漫"}, {"text": "🐼 国漫", "callback_data": "wiz_cat_国漫"}],
                                    [{"text": "📺 日韩剧", "callback_data": "wiz_cat_日韩剧"}, {"text": "🎬 日韩电影", "callback_data": "wiz_cat_日韩电影"}],
                                    [{"text": "🎤 综艺", "callback_data": "wiz_cat_综艺"}, {"text": "🎥 纪录片", "callback_data": "wiz_cat_纪录片"}],
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
                            
                            kw_map = {"视频": "mp4 mkv ts", "CAS": "cas", "全盘": ""}
                            kw = kw_map.get(f_type, "")
                            
                            # 🌟 核心魔法：把刚才选的周几拼接到指令里！
                            day_tag = f" #{state['day']}" if "day" in state else ""
                            s_cmd = f"订阅{state['s_num']}" 
                            
                            # 最终组装的命令形态：订阅 绝命毒师 https... #周更 #周三 #美剧 mp4
                            cmd = f"{s_cmd} {state['title']} {state['url']} #{state['freq']}{day_tag} #{state['cat']} {kw}".strip()
                            
                            notifier.edit_message(msg_id, f"🎉 <b>向导收集完毕！</b>\n正在为您下发指令:\n<code>{cmd}</code>")
                            
                            # 把指令原样注入消息队列，喂给底下的原始逻辑
                            item['message'] = {'chat': {'id': chat_id}, 'text': cmd}

                    # ==========================================
                    # 📝 恢复原有的文字处理逻辑
                    # ==========================================

                    msg = item.get('message', {})
                    text = msg.get('text', '')
                    chat_id = msg.get('chat', {}).get('id')

                    if str(chat_id) == str(TG_ADMIN_USER_ID):
                        text = text.strip()
                        
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
                                [{"text": "🎬 电影/单次任务", "callback_data": "wiz_freq_电影"}],
                                [{"text": "❌ 取消", "callback_data": "wiz_cancel"}]
                            ]}
                            
                            s_tip = f" (第 {wizard_states[chat_id]['s_num']} 季)" if wizard_states[chat_id]["s_num"] else ""
                            notifier.send_message(f"✅ 已记录剧名: {wizard_states[chat_id]['title']}{s_tip}\n\n<b>🏷️ 请选择【更新频率】:</b>", kb)
                            continue
                            
                        # === 原版指令识别开始 ===
                        logger.info(f"🛠️ [指令] 接收到远程终端最高权限指令: {text}")

                        if text == "列表" or text == "清单":
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

                        elif text.startswith("取消"):
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
                                m = re.match(r'^(.*?)\s*[sS第]?0?(\d+)[季]?$', keyword_input)
                                if m and m.group(1).strip(): base_kw, s_num = m.group(1).strip(), int(m.group(2))
                                else: base_kw, s_num = keyword_input, None

                                notifier.send_message(f"🔍 收到入库指令，正在检索: {base_kw}...")
                                
                                subs = load_json(SUBS_FILE)
                                matched_paths = []
                                for t_id, info in subs.items():
                                    path_in_db = info.get("path", "") if isinstance(info, dict) else ""
                                    if base_kw.lower() in path_in_db.lower():
                                        if s_num is not None:
                                            s_patterns = [f"season {s_num}", f"s{s_num:02d}", f"s{s_num}"]
                                            if any(p in path_in_db.lower() for p in s_patterns) or str(s_num) in path_in_db.split('/')[-1]:
                                                if path_in_db not in matched_paths: matched_paths.append(path_in_db)
                                        else:
                                            if path_in_db not in matched_paths: matched_paths.append(path_in_db)
                                                
                                if matched_paths:
                                    notifier.send_message(f"🎯 共命中 {len(matched_paths)} 个关联目录，批量刷新...")
                                    for mp in matched_paths:
                                        openlist_p = get_openlist_path(mp)
                                        
                                        if mp.startswith(DIR_CAS_ROOT) or mp.startswith(DIR_CAS_ROOT.strip('/')):
                                            try: 
                                                requests.get("http://127.0.0.1:5000/api/sync", params={"path": openlist_p}, timeout=3) 
                                                notifier.send_message(f"✅ 管家同步指令已下发: {openlist_p}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            try:
                                                subprocess.run(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_p], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                                                notifier.send_message(f"✅ 目录刷新成功: {openlist_p}")
                                            except: pass
                                    notifier.send_message("🎉 批量指令已全部执行！")
                                else:
                                    openlist_p_emby_old = get_openlist_path(DIR_MEDIA_PREFIX.strip('/'))
                                    openlist_p_emby_new = get_openlist_path(DIR_VIDEO_ROOT.strip('/'))   
                                    openlist_p_cas = get_openlist_path(DIR_CAS_ROOT.strip('/'))          
                                    
                                    notifier.send_message(f"⚠️ 库中无记录，开启大范围雷达扫描！\n请稍候前往 Emby 查看。")
                                    
                                    try:
                                        requests.get("http://127.0.0.1:5000/api/sync", params={"path": openlist_p_cas}, timeout=3) 
                                    except: pass
                                    
                                    try:
                                        subprocess.run(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_p_emby_old], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                                        subprocess.run(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_p_emby_new], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                                    except: pass
                                    
                                    notifier.send_message(f"✅ 全区最高指令已下发！")

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
                                                requests.get("http://127.0.0.1:5000/api/sync", params={"path": openlist_target_path}, timeout=3) 
                                                notifier.send_message(f"✅ 管家同步指令已下发: {openlist_target_path}")
                                            except Exception as e: 
                                                notifier.send_message(f"❌ 管家同步无响应: {e}")
                                        else:
                                            try: 
                                                subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                                                notifier.send_message(f"✅ Emby刷新指令已下发: {openlist_target_path}")
                                            except: pass
                                    else:
                                        taskInfos = [{"fileId": f["id"], "fileName": clean_filename(f["name"]), "isFolder": 0} for f in new_files]
                                        success_count, renamed_count = 0, 0
                                        history_data = load_json(HISTORY_FILE)
                                        error_msgs = []
                                        
                                        for i in range(0, len(taskInfos), 50):
                                            batch = taskInfos[i:i+50]
                                            code = info_s.saveShareFiles(batch, target_id)
                                            
                                            if code in [0, '0', None, False, '']:
                                                success_count += len(batch)
                                                for task in batch:
                                                    history_data[str(task["fileId"])] = {"name": task["fileName"], "sub_id": str(target_id)}
                                            else:
                                                error_msgs.append(str(code))
                                                
                                        if success_count > 0:
                                            time.sleep(3)
                                            fresh_cloud_files = client_obj.listPrivateFiles(target_id)
                                            for task in taskInfos:
                                                original_name = task["fileName"]
                                                new_name = generate_smart_name(original_name, target_path)
                                                if new_name != original_name:
                                                    for cf in fresh_cloud_files:
                                                        if cf["name"] == original_name:
                                                            if client_obj.renameFile(cf["id"], new_name): renamed_count += 1
                                                            break

                                        if success_count > 0:
                                            save_json(HISTORY_FILE, history_data)
                                            notifier.send_message(f"✅ 补档完美结束！\n📂 抓取 {success_count} 个缺失文件。")
                                            if renamed_count > 0:
                                                notifier.send_message(f"✨ 云端更名: 规范化 {renamed_count} 个文件！")
                                                time.sleep(6)
                                            
                                            if target_path.startswith(DIR_CAS_ROOT) or target_path.startswith(DIR_CAS_ROOT.strip('/')):
                                                try:
                                                    requests.get("http://127.0.0.1:5000/api/sync", params={"path": openlist_target_path}, timeout=3) 
                                                    notifier.send_message(f"✅ 管家同步指令已下发: {openlist_target_path}")
                                                except Exception as e: 
                                                    notifier.send_message(f"❌ 管家同步无响应: {e}")
                                            else:
                                                try:
                                                    subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", openlist_target_path], stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
                                                    notifier.send_message(f"✅ Emby刷新指令已下发: {openlist_target_path}")
                                                except: pass
                                        else:
                                            err_info = ", ".join(error_msgs) if error_msgs else "全盘拦截"
                                            notifier.send_message(f"❌ 天翼云拒绝转存: {err_info}")
                                except Exception as e:
                                    if "InvalidSessionKey" in str(e) or "check ip error" in str(e):
                                        notifier.send_message(f"⚠️ 检测到 IP 漂移！正在自愈...")
                                        if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                        auto_relogin(client_obj)
                                        notifier.send_message("✅ IP 漂移已修复！请重发指令。")
                                    else:
                                        notifier.send_message(f"❌ 补档异常: {e}")

                        elif text.startswith("订阅") or text.startswith("绑定"):
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
                                        
                                        if is_cas:
                                            if base_dir_sub:
                                                target_path = f"{DIR_CAS_ROOT}/{base_dir_large}/{base_dir_sub}/{current_ym}/{clean_user_path}".replace("//", "/")
                                            else:
                                                target_path = f"{DIR_CAS_ROOT}/{base_dir_large}/{current_ym}/{clean_user_path}".replace("//", "/")
                                            type_msg = "CAS秒传"
                                        else:
                                            if base_dir_sub:
                                                target_path = f"{DIR_VIDEO_ROOT}/{base_dir_large}/{base_dir_sub}/{current_ym}/{clean_user_path}".replace("//", "/")
                                            else:
                                                target_path = f"{DIR_VIDEO_ROOT}/{base_dir_large}/{current_ym}/{clean_user_path}".replace("//", "/")
                                            type_msg = "智能直连(带时间轴)"
                                        
                                        if base_dir_large == "电影" and not freq_tag:
                                            freq_tag = "电影"
                                            
                                        notifier.send_message(f"🧠 路由组装完毕 [{type_msg}]！\n📂 建档至: {target_path}")
                                    except Exception as e:
                                        if "InvalidSessionKey" in str(e) or "check ip error" in str(e):
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
                                    if "InvalidSessionKey" in str(e) or "check ip error" in str(e):
                                        notifier.send_message(f"⚠️ 建档探测到 IP 漂移！正在自愈...")
                                        if os.path.exists(COOKIES_FILE): os.remove(COOKIES_FILE)
                                        auto_relogin(client_obj)
                                        notifier.send_message("✅ 自愈完成，请重发指令。")
                                    else:
                                        notifier.send_message(f"❌ 云端拦截: {e}")
        except Exception: pass 
        time.sleep(2)

# ==========================================
# 🚀 启动入口
# ==========================================
if __name__ == '__main__':
    os.makedirs(DB_DIR, exist_ok=True)
    notifier = TelegramNotifier(TG_BOT_TOKEN, TG_ADMIN_USER_ID)
    notifier.send_message(f"🤖 追剧转存引擎 (V4.8 智能路由版) 启动，获取凭证...")
    
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
        exit(-1)
        
    main_control_loop(client)
```

3.2电脑文件传到手机storage/downloads目录下

3.3复制脚本
```
cp ~/storage/downloads/auto_189.py ~/tcloud/auto_189.py
```
4.点火升空 & 日常操作
前台启动
```
python auto_189.py
```
后台静默启动
```
nohup python auto_189.py > run.log 2>&1 &
```
pm2启动：
```
cd ~/tcloud && pm2 start auto_189.py --name "auto_189" --interpreter python
```
或
```
(cd ~/tcloud && nohup python3 auto_189.py > run.log 2>&1 &) && echo "✅ TG 追剧管家已启动，日志记录于 ~/run.log"
```

5.清理脏数据，正式接客

在 Termux 里直接执行这两行命令（先清空数据库，再重新启动）：
```
# 1. 强制清空订阅数据库和历史记录（把脏数据扬了）
pkill -f auto_189.py
rm -f db/history.json db/subscriptions.json

# 2. 重新启动脚本！
python auto_189.py
```

### 三、秒传播放cas_server.py
cas_server.py与auto_189.py同目录下操作方法一样

1.py内容：
```
import base64, json, time, random, hashlib, hmac, urllib.parse, threading, uuid, os, requests, logging, subprocess
import socket, re
from collections import deque
from flask import Flask, request, redirect, render_template_string, jsonify
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import pad

# 🛡️ 强制锁死 IPv4
old_getaddrinfo = socket.getaddrinfo
def new_getaddrinfo(*args, **kwargs):
    responses = old_getaddrinfo(*args, **kwargs)
    return [res for res in responses if res[0] == socket.AF_INET]
socket.getaddrinfo = new_getaddrinfo

# ==========================================
# ⚙️ 默认系统配置
# ==========================================
DEFAULT_CONFIG = {
    "server_host": "https://on.363689.xyz",
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

app = Flask(__name__)
last_refresh_time = 0
upload_cache = {}
cache_lock = threading.Lock()
rr_index = 0

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_DIR = os.path.join(BASE_DIR, "db")
os.makedirs(DB_DIR, exist_ok=True)

def get_db_path(): return os.path.join(DB_DIR, "config.json")

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
        if sk:
            logger.info(f"✨ [{source} 获取成功] 拿到新鲜 SESSION_KEY！凭证尾号: {sk[-6:]}")
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

slot3_cache = {"sk": "", "cookie_mtime": 0}

def get_auto189_session_key():
    cookie_file = os.path.join(DB_DIR, "cookies.json")
    if not os.path.exists(cookie_file): return ""
    
    mtime = os.path.getmtime(cookie_file)
    if slot3_cache["sk"] and slot3_cache["cookie_mtime"] == mtime:
        return slot3_cache["sk"]
        
    session = requests.Session()
    try:
        with open(cookie_file, 'r', encoding='utf-8') as f: 
            session.cookies.update(json.load(f))
        sk = get_session_key_via_api(session, "卡槽3-外部同步") or ""
        slot3_cache["sk"] = sk
        slot3_cache["cookie_mtime"] = mtime
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
            with open(cfg_path, 'r', encoding='utf-8') as f:
                saved_cfg = json.load(f)
                cfg.update(saved_cfg)
    except: pass
    
    if len(cfg.get('family_clouds', [])) > 2:
        if not cfg['family_clouds'][2].get('username'):
            old_sk = cfg['family_clouds'][2].get('session_key', '')
            new_sk = get_auto189_session_key()
            if new_sk:
                cfg['family_clouds'][2]['session_key'] = new_sk
                if old_sk != new_sk:
                    save_config(cfg)
            
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
                    fc['session_key'] = new_sk
                    save_config(cfg)
                    if 'session_key' in fc and fc['session_key'] in client.rsa_keys:
                        del client.rsa_keys[fc['session_key']]
                    logger.info(f"✅ 卡槽 {slot_idx+1} 满血复活！")
                    return new_sk
            except Exception as e:
                logger.error(f"❌ 卡槽 {slot_idx+1} 自动续期失败: {e}")
                send_push("❌ 追剧管家自愈失败", f"卡槽 {slot_idx+1} 尝试自动重登失败！<br>报错信息: {e}")
                
        elif slot_idx == 2:
            # 🌟 V7.4 容灾模式：撕毁缓存，不再唤醒外部脚本，直接返回 None 触发降级转移
            logger.warning("🚨 备用卡槽 3 外部凭证失效，已清理本地废弃凭证，等待外部定时任务更新。")
            send_push("⚠️ 卡槽 3 离线转移", "卡槽 3 外部凭证失效。已自动丢弃旧凭证，当前播放请求将无缝转移至主卡槽。")
            
            cookie_path = os.path.join(DB_DIR, "cookies.json")
            if os.path.exists(cookie_path): os.remove(cookie_path)
            slot3_cache["sk"] = ""
            
            if len(cfg.get('family_clouds', [])) > 2:
                cfg['family_clouds'][2]['session_key'] = ""
                save_config(cfg)
                
            return None # 无法当场自愈，返回 None 让主程序去执行故障转移
        else:
            logger.error(f"❌ 卡槽 {slot_idx+1} 缺少账号密码配置，无法执行自愈！")
            
    return None

def get_target_cloud(cfg, bind_key=""):
    global rr_index
    clouds = [c for c in cfg.get('family_clouds', []) if c.get('family_id') and c.get('hard_folder_id') and c.get('openlist_prefix')]
    if not clouds: return None, -1
    
    strategy = cfg.get('cloud_strategy', 'hash')
    target = clouds[0]
    
    def is_valid(c):
        return c and c.get('family_id') and c.get('hard_folder_id') and c.get('openlist_prefix')

    if strategy == 'primary' and len(clouds) > 0 and is_valid(clouds[0]):
        return clouds[0], 0
    elif strategy == 'slot2' and len(clouds) > 1 and is_valid(clouds[1]):
        return clouds[1], 1
    elif strategy == 'slot3' and len(clouds) > 2 and is_valid(clouds[2]):
        return clouds[2], 2

    valid_clouds = [c for c in clouds if is_valid(c)]
    if not valid_clouds: return None, -1
    
    target = valid_clouds[0]
    
    if strategy == 'hash':
        if bind_key:
            hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(valid_clouds)
            target = valid_clouds[hash_idx]
    elif strategy == 'random': 
        target = random.choice(valid_clouds)
    elif strategy == 'round_robin':
        target = valid_clouds[rr_index % len(valid_clouds)]
        rr_index += 1
        
    try:
        slot_idx = clouds.index(target)
    except:
        slot_idx = 0
        
    return target, slot_idx

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

@app.route('/api/remote_log', methods=['POST'])
def receive_remote_log():
    try:
        data = request.json
        log_buffer.append({'time': time.strftime("%H:%M:%S"), 'level': data.get('level', 'INFO'), 'msg': f"💜 [引擎] {data.get('msg', '')}"})
        return "OK", 200
    except: return "Error", 400

def send_push(title, content):
    cfg = read_config()
    if cfg.get('pushplus_token'):
        try: 
            requests.get(f"http://www.pushplus.plus/send?token={cfg['pushplus_token']}&title={urllib.parse.quote(title)}&content={urllib.parse.quote(content)}&template=html", timeout=5)
        except Exception as e: 
            logger.error(f"微信推送失败: {e}")
            
    if cfg.get('tg_bot_token') and cfg.get('tg_chat_id'):
        try: 
            requests.post(f"https://api.telegram.org/bot{cfg['tg_bot_token']}/sendMessage", data={"chat_id": cfg['tg_chat_id'], "text": f"🚨 <b>{title}</b>\n\n{content.replace('<br>', '\n')}", "parse_mode": "HTML"}, timeout=5)
        except Exception as e: 
            logger.error(f"TG推送失败: {e}")

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
            logger.info(f"🧹 正在搜寻卡槽 {i+1} [{fam_id[-4:]}] 的残留缓存...")
            try:
                items = client.get_family_items(fam_id, fold_id, sk)
                del_count = 0
                for item in items:
                    if client.delete_item(fam_id, item['fileId'], sk): del_count += 1
                
                time.sleep(1)
                if client.empty_family_recycle(fam_id, sk):
                    logger.info(f"✅ 卡槽 {i+1} 清理完毕: 移除了 {del_count} 个秒传文件，并清空了该家庭云回收站。")
            except Exception as e:
                logger.error(f"❌ 卡槽 {i+1} 清理异常: {e}")
    logger.info("🎉 缓存清空作业已圆满完成！")
    send_push("🧹 追剧管家清理完成", "全矩阵秒传缓存及回收站已强制清空完毕。")

@app.route('/api/clear_all', methods=['POST'])
def api_clear_all():
    threading.Thread(target=force_clear_all_worker, daemon=True).start()
    return "✅ 清空指令下发成功", 200

# ==========================================
# 🖥️ ADMIN 界面与配置路由
# ==========================================
@app.route('/admin/config', methods=['POST'])
def update_global_config():
    cfg = read_config()
    clouds = []
    for i in range(3):
        fid = request.form.get(f'fc_id_{i}', '').strip()
        hid = request.form.get(f'fc_fd_{i}', '').strip()
        px = request.form.get(f'fc_prefix_{i}', '').strip()
        mt = request.form.get(f'fc_mount_{i}', '').strip()
        user = request.form.get(f'fc_user_{i}', '').strip()
        pwd = request.form.get(f'fc_pwd_{i}', '').strip()
        sk = request.form.get(f'fc_sk_{i}', '').strip()
        
        if fid and hid:
            cloud_data = {"family_id": fid, "hard_folder_id": hid, "openlist_prefix": px, "openlist_mount_path": mt}
            cloud_data["username"] = user
            cloud_data["password"] = pwd
            cloud_data["session_key"] = sk
            clouds.append(cloud_data)
    
    cfg['family_clouds'] = clouds
    cfg['cloud_strategy'] = request.form.get('cloud_strategy', 'hash')
    
    for k in cfg.keys():
        if k not in ['family_clouds', 'cloud_strategy'] and k in request.form:
            val = request.form.get(k, '').strip()
            if k in ['delete_delay', 'shield_delay']:
                cfg[k] = int(val) if val else (600 if k == 'delete_delay' else 2700)
            else:
                cfg[k] = val
            
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
            <h2>💖 追剧管家 V7.4 <span style="font-size:12px; color:#8b5cf6;">无感故障转移版</span></h2>
            <span class="badge">SYSTEM ONLINE</span>
        </div>
        
        {% if msg %}<div class="status-msg">{{ msg }}</div>{% endif %}

        <div class="mac-window">
            <div class="mac-header">
                <div class="mac-btn btn-close"></div>
                <div class="mac-btn btn-min"></div>
                <div class="mac-btn btn-max"></div>
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
                    {% for i in range(3) %}
                        {% set fc = cfg.family_clouds[i] if i < cfg.family_clouds|length else {} %}
                        {% set sk = fc.get('session_key', '') %}
                        卡槽 {{ i+1 }}: 
                        {% if sk %}
                            <span style="color:#10b981; font-weight:bold;">已挂载 (尾号{{ sk[-4:] }})</span>
                        {% else %}
                            <span style="color:#f43f5e; font-weight:bold;">等待获取凭证...</span>
                        {% endif %}
                        <br>
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
                        <option value="hash" {% if cfg.cloud_strategy == 'hash' %}selected{% endif %}>🔗 剧名哈希绑定 (推荐！同剧永不跳云)</option>
                        <option value="round_robin" {% if cfg.cloud_strategy == 'round_robin' %}selected{% endif %}>🔁 顺序轮询分发 (常规平摊风控压力)</option>
                        <option value="primary" {% if cfg.cloud_strategy == 'primary' %}selected{% endif %}>🥇 仅卡槽1分发 (永远只用卡槽 1)</option>
                        <option value="slot2" {% if cfg.cloud_strategy == 'slot2' %}selected{% endif %}>🥈 仅卡槽2分发 (永远只用卡槽 2)</option>
                        <option value="slot3" {% if cfg.cloud_strategy == 'slot3' %}selected{% endif %}>🥉 仅卡槽3分发 (永远只用卡槽 3)</option>
                        <option value="random" {% if cfg.cloud_strategy == 'random' %}selected{% endif %}>🎲 完全随机分发 (看脸分配)</option>
                    </select>
                </div>

                <div class="grid">
                    <div>
                        {% for i in range(3) %}
                        {% set fc = cfg.family_clouds[i] if i < cfg.family_clouds|length else {} %}
                        <div class="cloud-box">
                            <div class="cloud-title">📌 独立挂载槽位 {{ i + 1 }} {% if i == 2 %}(填账号则自愈，留空则由 auto_189 接管){% else %}(静默自愈){% endif %}</div>
                            <input type="text" name="fc_id_{{ i }}" value="{{ fc.get('family_id', '') }}" placeholder="Family ID (留空则禁用此槽)">
                            <input type="text" name="fc_fd_{{ i }}" value="{{ fc.get('hard_folder_id', '') }}" placeholder="Folder ID">
                            
                            <div style="display:flex; gap:10px;">
                                <input type="text" name="fc_user_{{ i }}" value="{{ fc.get('username', '') }}" placeholder="天翼云账号{% if i == 2 %} (留空则由外部接管){% endif %}">
                                <input type="password" name="fc_pwd_{{ i }}" value="{{ fc.get('password', '') }}" placeholder="天翼云密码">
                            </div>
                            <input type="hidden" name="fc_sk_{{ i }}" value="{{ fc.get('session_key', '') }}">
                            
                            <input type="text" name="fc_prefix_{{ i }}" value="{{ fc.get('openlist_prefix', '') }}" placeholder="OpenList 专属播放前缀 (Prefix)">
                            <input type="text" name="fc_mount_{{ i }}" value="{{ fc.get('openlist_mount_path', '') }}" placeholder="OpenList 专属挂载目录" style="margin-bottom:0;">
                        </div>
                        {% endfor %}
                    </div>
                    
                    <div>
                        <h4 style="margin-top:0;">🌐 全局基础设置</h4>
                        <label>基础外网域名 (Server Host)</label>
                        <input type="text" name="server_host" value="{{ cfg.server_host }}" required>
                        
                        <div style="display: flex; gap: 10px; margin-bottom: 15px;">
                            <div style="flex: 1;">
                                <label>绝对销毁倒计时 (秒)</label>
                                <input type="number" name="delete_delay" value="{{ cfg.delete_delay }}" style="margin-bottom:0;" required>
                            </div>
                            <div style="flex: 1;">
                                <label>预加载长效护盾 (秒)</label>
                                <input type="number" name="shield_delay" value="{{ cfg.shield_delay }}" style="margin-bottom:0;" required>
                            </div>
                        </div>

                        <label>本地 STRM 保存路径</label>
                        <input type="text" name="local_strm_dir" value="{{ cfg.local_strm_dir }}" required>
                        <label>云端转存 CAS 路径</label>
                        <input type="text" name="network_cas_path" value="{{ cfg.network_cas_path }}" required>
                        
                        <h4 style="margin-top:20px;">🌐 OpenList 核心设置</h4>
                        <label>OpenList 接口地址 (Host)</label>
                        <input type="text" name="openlist_host" value="{{ cfg.openlist_host }}" required>
                        <label>OpenList 授权 Token</label>
                        <input type="password" name="openlist_token" value="{{ cfg.openlist_token }}" placeholder="在此填入有效的 OpenList Token">
                    </div>
                </div>

                <button type="submit" style="width:100%; margin-top:15px; font-size:16px;">💾 写入配置并重启引擎矩阵</button>
            </form>
        </div>
        <div style="height: 40px;"></div>
    </div>

    <script>
        function syncOpenList() {
            fetch('/api/sync').then(r => alert('同步指令已下发！请看上方日志。'));
        }
        function clearAllCache() {
            if(confirm('⚠️ 警告：这会清空所有 3 个卡槽 [秒传目录] 下的缓存视频，并强制清空天翼云回收站！\\n\\n确定要执行物理粉碎吗？')) {
                fetch('/api/clear_all', {method: 'POST'}).then(r => alert('🚀 核弹已发射！请看上方日志面板查看清理进度。'));
            }
        }
        function fetchLogs() {
            fetch('/admin/logs').then(r => r.json()).then(logs => {
                const box = document.getElementById('logBox');
                const oldScrollTop = box.scrollTop;
                const oldScrollHeight = box.scrollHeight;
                const clientHeight = box.clientHeight;
                
                box.innerHTML = logs.map(l => `<span class="log-time">[${l.time}]</span><span class="log-msg ${l.msg.includes('✅') ? 'log-SUCCESS' : 'log-' + l.level}">${l.msg}</span><br>`).join('');
                
                if (oldScrollHeight - clientHeight - oldScrollTop < 30) {
                    box.scrollTop = box.scrollHeight;
                } else {
                    box.scrollTop = oldScrollTop;
                }
            });
        }
        setInterval(fetchLogs, 2000);
        fetchLogs();
    </script>
</body>
</html>
"""

@app.route('/admin')
def admin_index():
    cfg = read_config()
    with cache_lock: count = len(upload_cache)
    return render_template_string(ADMIN_HTML, cfg=cfg, cache_count=count, msg=request.args.get('msg'))

@app.route('/admin/logs')
def get_logs():
    return jsonify(list(log_buffer))

# ==========================================
# ☁️ 天翼云核心功能类
# ==========================================
class TianyiFinalUploader:
    def __init__(self):
        self.rsa_keys = {}

    def get_base_headers(self, session_key):
        return {
            'User-Agent': 'ecloud/10.2.1 (Windows NT 10.0; Win64; x64)', 
            'Cookie': f"SESSION_KEY={session_key}; cookieUserSession={session_key}", 
            'Accept': 'application/json;charset=UTF-8',
            'clientType': 'TELEMAC'
        }

    def _random_string(self, length=16): return ''.join(random.choices('0123456789abcdef', k=length))
    def _get_timestamp(self): return str(int(time.time() * 1000))

    def get_rsa_key(self, session_key):
        if session_key in self.rsa_keys: return self.rsa_keys[session_key]
        url = f"https://cloud.189.cn/api/security/generateRsaKey.action?sessionKey={urllib.parse.quote(session_key)}"
        for _ in range(3):
            try:
                res = requests.get(url, headers=self.get_base_headers(session_key), timeout=10).json()
                if 'pubKey' in res:
                    self.rsa_keys[session_key] = res
                    return res
                if str(res.get('res_code')) == '111' or 'Session' in str(res):
                    raise Exception("公钥获取拦截_AUTH_FAIL")
            except Exception as e: 
                if "AUTH_FAIL" in str(e): raise e
            time.sleep(2)
            
        logger.error(f"❌ 无法获取公钥！凭证尾号 [{session_key[-4:]}] 可能已失效！")
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
        res = requests.get("https://cloud.189.cn/api/open/family/file/listFiles.action", params=params, headers=self.get_base_headers(session_key), timeout=10).json()
        
        if str(res.get('res_code')) == '111' or 'Session' in str(res):
            raise Exception("接口返回111_AUTH_FAIL")
            
        for f in res.get('fileListAO', {}).get('fileList', []):
            all_items.append({'fileName': f['name'], 'fileId': f['id']})
        return all_items

    def delete_item(self, family_id, file_id, session_key):
        url = "https://cloud.189.cn/api/open/family/file/deleteFile.action"
        p = {"familyId": family_id, "fileId": file_id, "sessionKey": session_key}
        try: return requests.post(url, params=p, headers=self.get_base_headers(session_key), timeout=10).status_code == 200
        except: return False

    def empty_family_recycle(self, family_id, session_key):
        url = "https://cloud.189.cn/api/open/batch/createBatchTask.action"
        payload = {"type": "EMPTY_RECYCLE", "taskInfos": "[]", "targetFolderId": "", "familyId": family_id, "sessionKey": session_key}
        try:
            res = requests.post(url, data=payload, headers=self.get_base_headers(session_key), timeout=10).json()
            if str(res.get("res_code")) == "0": return True
        except: pass
        return False

    def rapid_upload(self, family_id, parent_folder_id, md5, size, smd5, safe_name, session_key):
        req_id = str(uuid.uuid4())
        init_p = {'familyId': family_id, 'parentFolderId': parent_folder_id, 'fileName': urllib.parse.quote(safe_name), 'fileSize': str(size), 'sliceSize': '10485760', 'fileMd5': md5, 'sliceMd5': smd5, 'lazyCheck': '1', 'opertype': '3'}
        url, h = self.build_request(init_p, '/family/initMultiUpload', req_id, session_key)
        res = requests.get(url, headers=h).json()
        if res.get('code') != 'SUCCESS': 
            msg_str = str(res.get('msg', ''))
            if any(k in msg_str.lower() for k in ['session', 'privatekey', '111']):
                raise Exception(f"秒传初始化拒绝_AUTH_FAIL: {msg_str}")
            raise Exception(f"秒传初始化失败: {msg_str}")
        
        up_id = res['data']['uploadFileId']
        
        ck_p = {'familyId': family_id, 'fileMd5': md5, 'sliceMd5': smd5, 'uploadFileId': up_id}
        url, h = self.build_request(ck_p, '/family/checkTransSecond', req_id, session_key)
        if not requests.get(url, headers=h).json().get('data', {}).get('fileDataExists'): 
            raise Exception("云端无此文件 (可能是死链或MD5未入库)")
        
        cm_p = {'familyId': family_id, 'uploadFileId': up_id, 'fileMd5': md5, 'sliceMd5': smd5, 'lazyCheck': '1', 'opertype': '3'}
        url, h = self.build_request(cm_p, '/family/commitMultiUploadFile', req_id, session_key)
        
        commit_res = requests.get(url, headers=h).json()
        file_info = commit_res.get('file')
        if not file_info:
            raise Exception(f"秒传确认失败，云端异常拦截: {commit_res.get('msg', '未知错误')}")
            
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
        items = client.get_family_items(fam_id, fold_id, session_key)
        real_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == name), None)
        
        if real_fid and client.delete_item(fam_id, real_fid, session_key):
            time.sleep(2) 
            if client.empty_family_recycle(fam_id, session_key):
                logger.info(f"♻️ 清空回收站: 存储卡槽 [{fam_id[-4:]}] 空间已释放")
    except Exception as e:
        logger.error(f"🗑️ 后台删除作业受阻: {e}")
        
    with cache_lock:
        if f_md5 in upload_cache: del upload_cache[f_md5]

@app.route('/play', methods=['GET', 'HEAD'])
def play():
    if request.method == 'HEAD': return "", 200, {'Content-Type': 'video/mp4', 'Accept-Ranges': 'bytes'}

    cas = request.args.get('cas')
    show_name_from_url = request.args.get('show', '').strip()
    safe_name = "未知文件"
    
    try:
        j = json.loads(base64.b64decode(cas).decode())
        f_md5 = str(j.get('md5') or j.get('fileMd5')).upper()
        s_md5 = str(j.get('slice_md5') or j.get('sliceMd5')).upper()
        
        name = j.get('name') or j.get('fileName')
        base_safe_name = "".join(x for x in name if x not in r'\/:*?"<>|')
        
        if show_name_from_url:
            show_identifier = re.sub(r'\s*\(\d{4}\)$', '', show_name_from_url).strip()
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

        # ============================================
        # 1. 绝对原版的 safe_name 生成逻辑 (一字不动)
        # ============================================
        if show_identifier and ep_num is not None:
            safe_name = f"{show_identifier}.S{s_num:02d}E{ep_num:02d}{ext}"
        else:
            if bind_key and bind_key not in base_safe_name:
                safe_name = f"{bind_key}.{base_safe_name}"
            else:
                safe_name = base_safe_name

        # ============================================
        # 2. 仅在生成好的原版 safe_name 后缀前插入画质标签
        # ============================================
        tags = []
        if re.search(r'(?i)\bSDR\b', base_safe_name): tags.append("SDR")
        if re.search(r'(?i)\bHDR\b', base_safe_name): tags.append("HDR")
        if re.search(r'(?i)\bHQ\b', base_safe_name): tags.append("HQ")
        if re.search(r'(?i)\bDV\b', base_safe_name): tags.append("DV")
        tag_str = "." + ".".join(tags) if tags else ""
        
        if tag_str:
            if safe_name.endswith(ext):
                safe_name = safe_name[:-len(ext)] + tag_str + ext
            else:
                safe_name = safe_name + tag_str + ext
                
        cfg = read_config()
        current_time = time.time()
        
        fam_fid, target_fam_id, target_fold_id, target_session_key, target_prefix, target_mount, is_new, requires_cleanup = None, None, None, None, None, None, False, False
        s_idx = 0

        with cache_lock:
            if f_md5 in upload_cache:
                target_prefix = upload_cache[f_md5]['prefix']
                
                shield_delay = cfg.get('shield_delay', 2700) 
                brother_shield_time = current_time + shield_delay
                extended_count = 0
                
                for k, v in upload_cache.items():
                    if v.get('show_name') == bind_key and k != f_md5:
                        if v['expire'] < brother_shield_time:
                            v['expire'] = brother_shield_time
                            extended_count += 1
                
                last_log_time = upload_cache[f_md5].get('last_log', 0)
                if current_time - last_log_time > 60: 
                    expire_self_str = time.strftime("%H:%M:%S", time.localtime(upload_cache[f_md5]['expire']))
                    logger.info(f"📡 探针侦测: [{safe_name}] 预计于 {expire_self_str} 删除 (已为 {extended_count} 个续集加持护盾)")
                    upload_cache[f_md5]['last_log'] = current_time
            else:
                cloud, s_idx = get_target_cloud(cfg, bind_key)
                if not cloud: raise Exception("未配置有效的家庭云阵列！请检查后台。")
                
                target_fam_id = cloud['family_id']
                target_fold_id = cloud['hard_folder_id']
                target_session_key = cloud['session_key']
                target_prefix = cloud['openlist_prefix']
                target_mount = cloud['openlist_mount_path']

                brother_exists = any(v.get('show_name') == bind_key for v in upload_cache.values())
                
                if brother_exists:
                    shield_delay = cfg.get('shield_delay', 2700)
                    initial_expire = current_time + shield_delay
                else:
                    initial_expire = current_time + cfg.get('delete_delay', 600)

                upload_cache[f_md5] = {
                    'fid': 'processing', 
                    'expire': initial_expire, 
                    'fam_id': target_fam_id, 'fold_id': target_fold_id, 
                    'session_key': target_session_key, 'prefix': target_prefix, 'mount': target_mount,
                    'show_name': bind_key
                }
                requires_cleanup = True
                
        if requires_cleanup:
            try:
                if not target_session_key:
                    raise Exception("AUTH_FAIL: 本地凭证为空，强制触发自愈流程")
                    
                items = client.get_family_items(target_fam_id, target_fold_id, target_session_key)
                fam_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                
                if not fam_fid:
                    logger.info(f"🔄 路由调度: [{bind_key}] -> 卡槽 {target_fam_id[-4:]}")
                    fam_fid = client.rapid_upload(target_fam_id, target_fold_id, f_md5, j.get('size') or j.get('fileSize'), s_md5, safe_name, target_session_key)
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
                            fam_fid = client.rapid_upload(target_fam_id, target_fold_id, f_md5, j.get('size') or j.get('fileSize'), s_md5, safe_name, target_session_key)
                            is_new = True
                    else:
                        if s_idx == 2 and not cfg['family_clouds'][2].get('username'):
                            logger.warning("🔄 卡槽 3 外部凭证失效，正在触发【无缝故障转移】至主卡槽...")
                            if len(cfg['family_clouds']) > 0 and cfg['family_clouds'][0].get('session_key'):
                                target_fam_id = cfg['family_clouds'][0]['family_id']
                                target_fold_id = cfg['family_clouds'][0]['hard_folder_id']
                                target_session_key = cfg['family_clouds'][0]['session_key']
                                target_prefix = cfg['family_clouds'][0]['openlist_prefix']
                                target_mount = cfg['family_clouds'][0]['openlist_mount_path']
                                
                                with cache_lock:
                                    upload_cache[f_md5]['fam_id'] = target_fam_id
                                    upload_cache[f_md5]['fold_id'] = target_fold_id
                                    upload_cache[f_md5]['session_key'] = target_session_key
                                    upload_cache[f_md5]['prefix'] = target_prefix
                                    upload_cache[f_md5]['mount'] = target_mount
                                
                                items = client.get_family_items(target_fam_id, target_fold_id, target_session_key)
                                fam_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                                if not fam_fid:
                                    fam_fid = client.rapid_upload(target_fam_id, target_fold_id, f_md5, j.get('size') or j.get('fileSize'), s_md5, safe_name, target_session_key)
                                    is_new = True
                            else:
                                raise Exception("卡槽 3 无法恢复，且主卡槽不可用，降级转移彻底失败！")
                        else:
                            raise Exception(f"卡槽 {s_idx+1} 防线自愈失败，无法重建有效凭证！")
                else:
                    raise e
                
            if fam_fid:
                if is_new:
                    if brother_exists: 
                        logger.info(f"✅ 秒传成功: [{safe_name}] (被动预加载，已天生加持长效护盾)")
                    else: 
                        logger.info(f"✅ 秒传成功: [{safe_name}] 已上传！")
                        send_push("▶️ 新剧集入库并播放", f"<b>{safe_name}</b><br>已成功秒传至卡槽 {target_fam_id[-4:]} 并启动播放通道。")
                        
                with cache_lock:
                    upload_cache[f_md5]['fid'] = fam_fid
                
                threading.Thread(target=cleanup_worker, args=(safe_name, f_md5, target_fam_id, target_fold_id, target_session_key), daemon=True).start()

        if is_new:
            time.sleep(1.5)
            try:
                if cfg['openlist_token'] and target_mount: 
                    logger.info(f"🔔 正在呼叫 OpenList 强制刷新缓存: {target_mount}")
                    res = requests.post(f"{cfg['openlist_host']}/api/fs/list", json={"path": target_mount, "refresh": True}, headers={"Authorization": cfg['openlist_token']}, timeout=10)
                    if res.status_code == 200 and res.json().get('code') == 200:
                        logger.info("✅ OpenList 缓存刷新成功！播放通道已通畅。")
                    else:
                        msg = res.json().get('message', '未知错误')
                        logger.error(f"❌ OpenList 拒绝刷新 (错误码:{res.json().get('code')}): {msg}")
                        send_push("⚠️ OpenList 刷新拒绝", f"节点 <b>{target_mount}</b> 刷新失败！<br>报错信息: {msg}<br>请及时登录后台更换最新的 Token。")
                else:
                    logger.warning("⚠️ 未配置 OpenList Token 或挂载路径，跳过刷新。")
            except Exception as e:
                logger.error(f"❌ 无法连接到 OpenList 服务: {e}")

        return redirect(f"{target_prefix.rstrip('/')}/{urllib.parse.quote(safe_name)}", code=302)
        
    except Exception as e:
        logger.error(f"❌ 链路故障: {e}")
        
        # ============================================
        # 🚨 致命死锁修复：报错拦截区瞬间抹除假缓存
        # ============================================
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
                requests.post(f"{cfg['openlist_host']}/api/fs/list", json={"path": current_path, "refresh": True}, headers=headers, timeout=5)
                time.sleep(0.5)
            except: pass

def scan_openlist_recursive(current_path, headers, result_list, cfg):
    logger.info(f"🔎 正在探测目录: {current_path}")
    try:
        req_headers = headers if cfg['openlist_token'] else {}
        res = requests.post(f"{cfg['openlist_host']}/api/fs/list", json={"path": current_path, "refresh": True}, headers=req_headers, timeout=15).json()
        
        if res.get("code") != 200: 
            logger.error(f"❌ OpenList 扫描拒绝 (可能 Token 错误或路径不存在): {res.get('message')}")
            return
            
        for f in res.get("data", {}).get("content", []):
            if f.get("is_dir"): scan_openlist_recursive(f"{current_path}/{f['name']}", headers, result_list, cfg)
            elif f['name'].endswith('.cas'): result_list.append(f"{current_path}/{f['name']}")
    except Exception as e:
        logger.error(f"❌ 探测目录失败 {current_path}: {e}")

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
    if not cas_files:
        logger.info(f"⚠️ 扫描完毕：该区域下未找到任何 .cas 文件")
        return
        
    count = 0
    for full_path in cas_files:
        try:
            if full_path.startswith(base_cas_path):
                rel_path = full_path[len(base_cas_path):].lstrip('/')
            else:
                rel_path = full_path.split('/')[-1]
                
            rel_dir = os.path.dirname(rel_path)
            
            dir_parts = [p for p in rel_dir.split('/') if p]
            show_name = ""
            for part in reversed(dir_parts):
                if not re.match(r'(?i)^(season\s*\d+|specials|电视剧|电影|动漫|纪录片|综艺)$', part):
                    show_name = part
                    break
            
            if not show_name and dir_parts: show_name = dir_parts[-1] 
            if not show_name: show_name = "未知剧集"
            
            show_name = re.sub(r'\s*\(\d{4}\)$', '', show_name).strip()
                
            base_name = os.path.basename(rel_path).rsplit('.', 1)[0]
            
            target_local_dir = os.path.join(cfg['local_strm_dir'], rel_dir)
            os.makedirs(target_local_dir, exist_ok=True)
            strm_path = os.path.join(target_local_dir, f"{base_name}.strm")
            
            if os.path.exists(strm_path): continue
            
            get_res = requests.post(f"{cfg['openlist_host']}/api/fs/get", json={"path": full_path}, headers=headers).json()
            raw_url = get_res.get("data", {}).get("raw_url")
            if not raw_url: continue
            
            cas_content = requests.get(raw_url).text.strip()
            strm_data = f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            
            with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)
            count += 1
        except: pass

    if count > 0: 
        logger.info(f"🎉 同步完毕，成功生成 {count} 个带精准剧名标记的 STRM 文件")
        try: 
            subprocess.Popen(["/data/data/com.termux/files/usr/bin/bash", "/data/data/com.termux/files/home/refresh.sh", scan_root])
            logger.info("🎬 已触发本地 Emby 媒体库刷新指令")
            send_push("🎬 STRM同步成功", f"成功生成并归档了 {count} 个媒体文件，并已触发媒体库刷新。")
        except: pass

@app.route('/api/sync')
def trigger_sync():
    target_path = request.args.get('path') 
    threading.Thread(target=generate_strm_from_openlist_to_local, args=(target_path,), daemon=True).start()
    return "✅ 同步指令下发成功", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
2.pm2启动

```
cd ~/tcloud && pm2 start cas_server.py --name "cas_server" --interpreter python
```