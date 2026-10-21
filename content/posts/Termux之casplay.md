---

title: "Termux之casplay"

author: "xxsky"

type: "posts"

date: 2026-08-03T16:08:57+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

秒传cas、189/139双端播放

<!--more-->
1.pm2管理
```
cd ~/189py && pm2 start auto189.py --name "auto189" --interpreter python

cd ~/189py && pm2 start casplay.py --name "casplay" --interpreter python

cd ~/189py && pm2 start autotg.py --name "autotg" --interpreter python

cd ~/189py && pm2 start cas_server.py --name "casplay" --interpreter python
```

2.脚本
```
pip install pycdlib
```
```
import base64, json, time, random, hashlib, hmac, urllib.parse, threading, uuid, os, requests, logging, subprocess, math
import socket, re, functools
import urllib3
import urllib.request, urllib.error, http.cookiejar, string
urllib3.util.connection.HAS_IPV6 = False
from collections import deque
from flask import Flask, request, redirect, render_template_string, jsonify, Response
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import pad
import io
import pycdlib
import requests

def get_iso_main_video_offset(iso_url, known_total_size=0):
    """军用级 X 光透视：只在 2048 扇区边界扫描，彻底杜绝假阳性偏移"""
    try:
        total_size = int(known_total_size)
        if total_size <= 0:
            head_r = requests.head(iso_url, allow_redirects=True, timeout=10)
            total_size = int(head_r.headers.get('content-length', 0))

        # 拉取前 16MB 目录区进行 X光透视
        r = requests.get(iso_url, headers={'Range': 'bytes=0-16777215', 'User-Agent': 'Mozilla/5.0'}, timeout=20)
        data = r.content
        if not data: return 0, total_size, "unknown"

        sector_size = 2048
        max_sectors = len(data) // sector_size

        for s in range(0, max_sectors):
            base = s * sector_size
            
            # 🔍 1. 严格在扇区起始处探测 MKV 签名 (杜绝 Invalid data 乱码碰撞)
            if data[base:base+4] == b'\x1A\x45\xDF\xA3':
                return base, total_size - base, "mkv"
                
            # 🔍 2. 严格在扇区起始处探测 MP4 签名 (ftyp)
            if base + 8 <= len(data) and data[base+4:base+8] == b'ftyp':
                return base, total_size - base, "mp4"
                
            # 🔍 3. 探测真正的蓝光 M2TS (0x47 间隔 192)
            # 光盘头部不可能小于 100KB，跳过前 50 个扇区防碰撞
            if s >= 50 and base + 960 <= len(data):
                if (data[base + 4] == 0x47 and data[base + 196] == 0x47 and 
                    data[base + 388] == 0x47 and data[base + 580] == 0x47):
                    return base, total_size - base, "m2ts"

        return 0, total_size, "unknown"
    except Exception as e:
        logger.error(f"[ISO解析] 异常: {e}")
        return 0, known_total_size, "unknown"

# ==========================================
# 🏠 局域网探针与辅助函数
# ==========================================
old_getaddrinfo = socket.getaddrinfo
def new_getaddrinfo(host, port, family=0, type=0, proto=0, flags=0):
    responses = old_getaddrinfo(host, port, family, type, proto, flags)
    if host == '::': return responses
    return [res for res in responses if res[0] == socket.AF_INET]
socket.getaddrinfo = new_getaddrinfo

def get_lan_server_ip(req):
    host = req.headers.get('X-Forwarded-Host', req.host).split(':')[0]
    if re.match(r'^(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[0-1])\.|127\.0\.0\.1)', host):
        client_ip = req.headers.get('X-Forwarded-For', req.remote_addr)
        if client_ip:
            client_ip = client_ip.split(',')[0].strip()
            if not re.match(r'^(192\.168\.|10\.|172\.(1[6-9]|2[0-9]|3[0-1])\.|127\.0\.0\.1)', client_ip):
                return None 
        return host
    return None

def format_size(size_in_bytes):
    try:
        size = float(size_in_bytes)
        if size < 1024 * 1024 * 1024: return f"{size / (1024 * 1024):.2f} MB"
        else: return f"{size / (1024 * 1024 * 1024):.2f} GB"
    except: return "未知大小"

def truncate_url(url):
    return url[:80] + '...[已折叠]' if url and len(url) > 80 else url

# ==========================================
# 🧩 终极 CAS 解析引擎 (严谨防错版：囊括所有老文件变体)
# ==========================================
def parse_cas_content(cas_str):
    if not cas_str: return {}
    try:
        # 第一步：解除 URL 编码，并暴烈剥离首尾的空格、换行以及不可见的 BOM 头
        cas_str = urllib.parse.unquote(cas_str).strip().lstrip('\ufeff')
        
        # 🚨 终极拦截：如果内容其实是个 HTML 报错网页（当初生成文件时网盘就死机了）
        if cas_str.startswith('<'):
            logger.error(f"❌ [数据拦截]: 发现死胎文件！这不是数据，而是一张网页报错单！截取: {cas_str[:80]}...")
            return {}
            
        # 第二步：【绝对真理判定】别管长啥样，先试着当 JSON 解码
        try:
            return json.loads(cas_str)
        except:
            pass # 如果不是 JSON，说明是老版本的 Base64 密文，继续往下走
        
        # 第三步：【情况 B - Base64 密文处理】
        # 1. 必须先把 URL 传输中丢失变成“空格”的加号还原！
        cas_b64 = cas_str.replace(' ', '+')
        # 2. 格式兼容：把 URL 安全模式的字符转回标准 Base64 字符
        cas_b64 = cas_b64.replace('-', '+').replace('_', '/')
        # 3. 精准杀毒：剔除所有混入的换行符(\n)、回车符(\r)等乱码杂质
        cas_b64 = re.sub(r'[^A-Za-z0-9+/=]', '', cas_b64)
        # 4. 完美补齐：靠严格的模运算自己把 '=' 补到完美倍数
        cas_b64 = cas_b64.rstrip('=')
        cas_b64 += "=" * ((4 - len(cas_b64) % 4) % 4)
        
        # 第四步：解密成 UTF-8 文本并读取为字典
        return json.loads(base64.b64decode(cas_b64).decode('utf-8'))
            
    except Exception as e:
        # 加上 repr()，让任何隐形导致崩溃的字符(如 \n, \r, \xe0)在日志里现出原形！
        logger.error(f"❌ [数据解析]: {e} | 异常数据原形: {repr(cas_str[:80])}")
        return {}
# ==========================================
# 🛡️ 智能防刷墙 (严格并发锁)
# ==========================================
anti_scan_history = {}
anti_scan_lock = threading.Lock()

def is_allowed_by_anti_scan(client_ip, f_md5):
    if not client_ip: return True
    now = time.time()
    with anti_scan_lock:
        if client_ip not in anti_scan_history: anti_scan_history[client_ip] = []
        history = [req for req in anti_scan_history[client_ip] if now - req[0] < 5]
        unique_md5s = set(req[1] for req in history)
        if f_md5 not in unique_md5s:
            if len(unique_md5s) >= 1:
                anti_scan_history[client_ip] = history
                return False 
            else: history.append((now, f_md5))
        else: history.append((now, f_md5))
        anti_scan_history[client_ip] = history
        return True

# ==========================================
# ⚙️ 默认系统配置 (主服务)
# ==========================================
DEFAULT_CONFIG = {
    "server_host": "https://play.363689.xyz",
    "delete_delay": 600,
    "shield_delay": 2700,
    "cloud_strategy": "hash", 
    "force_mode_b": "false",
    "openlist_host": "http://127.0.0.1:5244",
    "openlist_token": "", 
    
    # === 🌟 V10 终极融合：4大统一战车卡槽 ===
    "accounts": [{}, {}, {}, {}],
    
    "mode_a_channel": "mix_f2p", 
    
    "local_cas_source_dir": "/storage/emulated/0/Download/cas_source",
    
    "network_cas_path": "/177/177-秒传",
    "local_strm_dir": "/storage/emulated/0/Download/cas_strm_modeA",
    
    "network_cas_path_native": "/177/177-原生直连",
    "local_strm_dir_native": "/storage/emulated/0/Download/cas_strm_modeB",
    
    "network_media_path": "/177/177-常规视频",
    "local_strm_dir_media": "/storage/emulated/0/Download/cas_strm_media",

    "network_cas_path_139": "/139/139-秒传",
    "local_strm_dir_139": "/storage/emulated/0/Download/cas_strm_139",
    "yun139_auth": "",
    "yun139_host": "https://caiyun.feixin.10086.cn:7071",
    "openlist_host_139": "http://127.0.0.1:5255",
    "openlist_token_139": "",
    "local_cas_source_dir_139": "/storage/emulated/0/Download/cas_source_139",
    "yun139_phone": "",
    "yun139_password": "",
    "yun139_token": "",
    "yun139_mail_cookie": "",
    "yun139_phone_2": "",
    "yun139_password_2": "",
    "yun139_token_2": "",
    "yun139_mail_cookie_2": "",  
    "local_strm_dir_139_native": "/storage/emulated/0/Download/cas_strm_139_native",
    "yun139_control_host": "https://personal-kd-njs.yun.139.com", 
    "yun139_link_expire": 7200, 
    "delete_delay_139": 14400,  
    "yun139_temp_folder_id": "",
    "yun139_temp_folder_id_2": "",
    "yun139_strategy": "slot1",  # 默认优先1号卡槽  
    "pushplus_token": "",
    "tg_bot_token": "",
    "tg_chat_id": "",
    "tg_proxy": "http://127.0.0.1:7890"
}

EMBY_HOST = "http://127.0.0.1:8096"
API_KEY_LINUX = "751c095055f8493d8e63eb755369b9aa"
API_KEY_APP = "66644805d4bc45ea91b2a5e5eca22105"

app_main = Flask('cas_server_5000')
app_302 = Flask('nginx_302_5001')

upload_cache = {}
native_link_cache = {}
iso_meta_cache = {}     
iso_scan_locks = {}     
media_resolve_locks = {}  
cache_lock = threading.Lock()
print_throttle_cache = {} 

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_DIR = os.path.join(BASE_DIR, "db")
os.makedirs(DB_DIR, exist_ok=True)

def get_db_path(): return os.path.join(DB_DIR, "config.json")

# ==========================================
# 🔔 统一看板日志与推送系统 (严谨双屏版)
# ==========================================
log_buffer_sys = deque(maxlen=150)   # 上窗口：系统监控、生成、外部推送、洗码、巡逻
log_buffer_play = deque(maxlen=150)  # 下窗口：仅 139/189 真实播放取链、劫持放行、销毁

def dispatch_log(log_entry):
    msg = log_entry.get('msg', '')

    # 1. 严格锁定【播放链路专属动作】（下窗口）
    play_features = [
        "链路日志",
        "[播放放行]",
        "[劫持放行]",
        "[网络嗅探]",
        "[原生直通]",
        "[状态请求]",
        "秒传命中]",
        "秒传成功]",
        "秒传就绪]",
        "开始调度]",
        "版权拦截]",
        "凭证失效]",
        "遭遇拦截]",
        "重试依然失败]",
        "[获取底层]",
        "嗅探完成]",
        "嗅探警告]",
        "[虚空造物]",
        "[云盘容灾]",
        "[阵亡警报]",
        "清理预约]",
        "移入纸篓]",
        "空间释放]",
        "[脱壳直链]",
        "[防和谐扫]",
        "[透视成功]",
        "[伪装放行]",
        "[ISO",
        "[直链缓存]",
        "[自愈成功]",
        "[自愈启动]",
        "[自愈失败]",
        "[票据过期]"
    ]

    # 2. 强行排除【STRM 生成与洗码操作】（防止被带进下窗口）
    exclude_features = [
        "生成 STRM",
        "接口调用",
        "洗码",
        "投递箱",
        "后续触发",
        "CAS嗅探启动"
    ]

    if any(k in msg for k in play_features) and not any(k in msg for k in exclude_features):
        log_buffer_play.append(log_entry)
    else:
        log_buffer_sys.append(log_entry)

class MemoryHandler(logging.Handler):
    def emit(self, record):
        msg = self.format(record)
        log_entry = {'time': time.strftime("%H:%M:%S"), 'level': record.levelname, 'msg': f"● {msg}"}
        dispatch_log(log_entry)

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
        msg_text = data.get('msg', '')
        log_entry = {'time': time.strftime("%H:%M:%S"), 'level': data.get('level', 'INFO'), 'msg': f"● [远程] {msg_text}"}
        dispatch_log(log_entry)
        return "OK", 200
    except: return "Error", 400

def send_push(title, content):
    def _do_push():
        cfg = read_config()
        if cfg.get('pushplus_token'):
            try: requests.get(f"http://www.pushplus.plus/send?token={cfg['pushplus_token']}&title={urllib.parse.quote(title)}&content={urllib.parse.quote(content)}&template=html", timeout=5)
            except: pass
        if cfg.get('tg_bot_token') and cfg.get('tg_chat_id'):
            tg_proxy = cfg.get('tg_proxy', '').strip()
            proxy_config = {"http": tg_proxy, "https": tg_proxy} if tg_proxy else None
            try: 
                requests.post(f"https://api.telegram.org/bot{cfg['tg_bot_token']}/sendMessage", data={"chat_id": cfg['tg_chat_id'], "text": f"🚨 <b>{title}</b>\n\n{content.replace('<br>', '\n')}", "parse_mode": "HTML"}, proxies=proxy_config, timeout=5)
            except Exception as e: 
                logger.error(f"❌ TG推送失败: {e} (当前代理: {tg_proxy or '直连'})")
    threading.Thread(target=_do_push, daemon=True).start()

# ==========================================
# 🔑 天翼云独立鉴权引擎 (统一双轨版)
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
        if sk: logger.info(f"[凭证更新] 成功获取 SESSION_KEY ({sk[-4:]})")
        return sk
    except Exception as e:
        logger.error(f"提取 sessionKey 失败: {e}")
        return None

class Cloud189AuthEngine:
    def __init__(self):
        self.session = requests.session()
        self.session.headers = {'User-Agent': "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36", "Accept": "application/json;charset=UTF-8"}

    def getEncrypt(self): return self.session.post("https://open.e.189.cn/api/logbox/config/encryptConf.do", data={'appId': 'cloud'}, timeout=15).json()['data']['pubKey']

    def getRedirectURL(self):
        rsp = self.session.get('https://cloud.189.cn/api/portal/loginUrl.action?redirectURL=https://cloud.189.cn/web/redirect.html?returnURL=/main.action', timeout=15)
        return urllib.parse.parse_qs(urllib.parse.urlparse(rsp.url).query)

    def do_login_and_get_key(self, username, password, slot_name="卡槽自愈"):
        encryptKey = self.getEncrypt()
        query = self.getRedirectURL()
        resData = self.session.post('https://open.e.189.cn/api/logbox/oauth2/appConf.do', data={"version": '2.0', "appKey": 'cloud'}, headers={"Referer": 'https://open.e.189.cn/', "lt": query["lt"][0], "REQID": query["reqId"][0]}, timeout=15).json()
        keyData = f"-----BEGIN PUBLIC KEY-----\n{encryptKey}\n-----END PUBLIC KEY-----"
        data = {"appKey": 'cloud', "version": '2.0', "accountType": '01', "mailSuffix": '@189.cn', "returnUrl": resData['data']['returnUrl'], "paramId": resData['data']['paramId'], "clientType": '1', "isOauth2": "false", "userName": f"{{NRP}}{rsaEncrpt(username, keyData)}", "password": f"{{NRP}}{rsaEncrpt(password, keyData)}"}
        result = self.session.post('https://open.e.189.cn/api/logbox/oauth2/loginSubmit.do', data=data, headers={'Referer': 'https://open.e.189.cn/', 'lt': query["lt"][0], 'REQID': query["reqId"][0]}, timeout=15).json()
        if result['result'] == 0:
            self.session.get(result['toUrl'], headers={"Host": 'cloud.189.cn'}, timeout=15)
            sk = get_session_key_via_api(self.session, slot_name)
            if sk: 
                cookie_str = "; ".join([f"{c.name}={c.value}" for c in self.session.cookies])
                return sk, cookie_str
            else: raise Exception("接口未返回 sessionKey")
        else: raise Exception(result['msg'])

# 🌟 全局缓存：记录文件最后修改时间和凭证
cookie_cache = {"sk": "", "cookie_str": "", "mtime": 0}

def get_auto189_credentials():
    global cookie_cache
    cookie_file = os.path.join(DB_DIR, "cookies.json")
    if not os.path.exists(cookie_file): 
        return "", ""
    
    current_mtime = os.path.getmtime(cookie_file)
    
    if cookie_cache["mtime"] == current_mtime and cookie_cache["sk"]:
        return cookie_cache["sk"], cookie_cache["cookie_str"]
        
    session = requests.Session()
    try:
        with open(cookie_file, 'r', encoding='utf-8') as f:
            cookie_dict = json.load(f)
            session.cookies.update(cookie_dict)
            
        sk = get_session_key_via_api(session, "外部同步大号") or ""
        
        if sk:
            cookie_str = "; ".join([f"{k}={v}" for k, v in cookie_dict.items()])
            cookie_cache["sk"] = sk
            cookie_cache["cookie_str"] = cookie_str
            cookie_cache["mtime"] = current_mtime
            return sk, cookie_str
        else:
            if os.path.exists(cookie_file): os.remove(cookie_file)
            logger.warning("❌ [外部凭证] 新 Cookie 无效已被天翼云拒绝，已粉碎文件等待外部脚本重建！")
            return "", ""
    except Exception as e: 
        logger.error(f"❌ [外部凭证] 读取异常: {e}")
        return "", ""

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
    return cfg

def refresh_account_logic(slot_idx, cfg):
    if slot_idx < len(cfg.get('accounts', [])):
        acc = cfg['accounts'][slot_idx]
        user, pwd = acc.get('username'), acc.get('password')

        if slot_idx == 3 and not pwd:
            logger.info("[自愈启动] 卡槽 4 尝试从外部读取...")
            sk, cookie = get_auto189_credentials()
            if sk and cookie:
                acc['session_key'] = sk
                acc['cookie'] = cookie
                save_config(cfg)
                logger.info("[自愈成功] 卡槽 4 满血复活！")
                send_push("✅ 大号自愈成功", "卡槽 4 成功从外部读取到最新 Cookie 凭证！")
                return sk, cookie
            else:
                logger.error("[自愈失败] 卡槽 4 提取失败！")
                send_push("❌ 大号自愈失败", "卡槽 4 外部 Cookie 提取失败或已过期！")
                return None, None

        if user and pwd:
            logger.info(f"[自愈启动] 卡槽 {slot_idx+1} 开始账号重登...")
            send_push("🔄 双轨自愈启动", f"检测到卡槽 {slot_idx+1} 凭证失效，正在执行自动重登...")
            try:
                auth = Cloud189AuthEngine()
                sk, cookie = auth.do_login_and_get_key(user, pwd, f"卡槽{slot_idx+1}")
                if sk and cookie:
                    acc['session_key'] = sk
                    acc['cookie'] = cookie
                    save_config(cfg)
                    logger.info(f"[自愈成功] 卡槽 {slot_idx+1} 满血复活！")
                    send_push("✅ 自愈成功", f"统一卡槽 {slot_idx+1} 账号重登成功，双轨满血复活！")
                    return sk, cookie
            except Exception as e:
                logger.error(f"[自愈失败] 卡槽 {slot_idx+1}: {e}")
                send_push("❌ 自愈失败", f"统一卡槽 {slot_idx+1} 自动重登失败: {e}")
    return None, None

# ==========================================
# 🧹 全量清空逻辑
# ==========================================
def force_clear_all_worker():
    logger.info("[全量清理] 开始横扫天翼云矩阵...")
    cfg = read_config()
    with cache_lock: upload_cache.clear()
    
    for i, acc in enumerate(cfg.get('accounts', [])):
        fam_id = acc.get('family_id')
        fold_id = acc.get('family_folder_id')
        per_id = acc.get('personal_folder_id')
        sk = acc.get('session_key')
        cookie = acc.get('cookie')
        
        if not sk or not cookie:
            sk, cookie = refresh_account_logic(i, cfg)
            
        if sk and fam_id and fold_id:
            try:
                items = family_client.get_family_items(fam_id, fold_id, sk)
                del_count = sum([family_client.delete_item(fam_id, item['fileId'], sk) for item in items])
                if del_count > 0: family_client.empty_family_recycle(fam_id, sk)
            except: pass
            
        if sk and cookie and per_id:
            try:
                items = personal_client.get_personal_items(per_id, cookie)
                del_count = sum([personal_client.delete_item(item['fileId'], cookie) for item in items])
                if del_count > 0: personal_client.empty_personal_recycle(sk, cookie)
            except: pass
            
    logger.info("[清理完毕] 全区垃圾回收作业圆满完成！")

@app_main.route('/api/clear_all', methods=['POST'])
def api_clear_all():
    threading.Thread(target=force_clear_all_worker, daemon=True).start()
    return "✅ 清空指令下发成功", 200

@app_main.route('/api/clear_139', methods=['POST'])
def api_clear_139():
    threading.Thread(target=cloud139_native.empty_recycle_bin, daemon=True).start()
    return "✅ 139清空指令下发成功", 200

# ==========================================
# 🖥️ ADMIN 界面与配置路由
# ==========================================
@app_main.route('/admin/config', methods=['POST'])
def update_global_config():
    old_cfg = read_config()
    cfg = DEFAULT_CONFIG.copy()
    for k, v in old_cfg.items(): cfg[k] = v
    
    accounts = []
    for i in range(4):
        user = request.form.get(f'acc_user_{i}', '').strip()
        pwd = request.form.get(f'acc_pwd_{i}', '').strip()
        fam_id = request.form.get(f'acc_fam_id_{i}', '').strip()
        fam_fd = request.form.get(f'acc_fam_fd_{i}', '').strip()
        per_fd = request.form.get(f'acc_per_fd_{i}', '').strip()
        
        old_acc = old_cfg.get('accounts', [{},{},{},{}])[i] if i < len(old_cfg.get('accounts', [])) else {}
        sk, cookie = old_acc.get('session_key', ''), old_acc.get('cookie', '')
        
        if user != old_acc.get('username', '') or pwd != old_acc.get('password', ''):
            sk, cookie = "", ""

        accounts.append({
            "username": user, "password": pwd, "family_id": fam_id, 
            "family_folder_id": fam_fd, "personal_folder_id": per_fd,
            "session_key": sk, "cookie": cookie
        })
            
    cfg['accounts'] = accounts
    cfg['cloud_strategy'] = request.form.get('cloud_strategy', 'hash')
    cfg['mode_a_channel'] = request.form.get('mode_a_channel', 'mix_f2p')
    cfg['force_mode_b'] = request.form.get('force_mode_b', 'false') 
    cfg['tg_proxy'] = request.form.get('tg_proxy', '').strip() 
    cfg['yun139_phone_2'] = request.form.get('yun139_phone_2', '').strip()
    cfg['yun139_password_2'] = request.form.get('yun139_password_2', '').strip()
    cfg['yun139_token_2'] = request.form.get('yun139_token_2', '').strip()
    cfg['yun139_mail_cookie_2'] = request.form.get('yun139_mail_cookie_2', '').strip()
    cfg['yun139_temp_folder_id_2'] = request.form.get('yun139_temp_folder_id_2', '').strip()  # 👈 新增保存
    cfg['yun139_strategy'] = request.form.get('yun139_strategy', 'slot1')
    
    for k in cfg.keys():
        if k not in ['accounts', 'cloud_strategy', 'mode_a_channel', 'force_mode_b', 'tg_proxy'] and k in request.form:
            val = request.form.get(k, '').strip()
            if k in ['delete_delay', 'shield_delay', 'delete_delay_139', 'yun139_link_expire']: 
                cfg[k] = int(val) if val else 600
            else: 
                cfg[k] = val
            
    save_config(cfg)
    
    try:
        if 'family_client' in globals():
            family_client.rsa_keys.clear() 
    except: pass
    
    try:
        if 'personal_client' in globals():
            personal_client.rsa_keys.clear()
    except: pass
    
    logger.info(f"[系统配置] V10 矩阵重组！四核心配置均已保存。")
    return redirect("/admin?msg=矩阵重组完成！配置实时生效")

ADMIN_HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>💖 追剧管家 V10 (真融合版)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #f4f6f9; margin: 0; padding: 20px; color: #333; }
        .container { max-width: 950px; margin: 0 auto; }
        .header { background: #fff; padding: 20px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.03); margin-bottom: 20px; display: flex; justify-content: space-between; align-items: center; border-left: 5px solid #2563eb; }
        .card { background: #fff; padding: 25px; border-radius: 12px; box-shadow: 0 4px 15px rgba(0,0,0,0.03); margin-bottom: 20px; }
        h2 { margin: 0; color: #1e293b; } h3 { margin-top: 0; color: #334155; font-size: 1.1rem; border-bottom: 1px solid #e2e8f0; padding-bottom: 10px; margin-bottom: 15px; }
        h4 { color: #475569; margin-bottom: 10px; padding-bottom: 5px; border-bottom: 1px dashed #cbd5e1; }
        .badge { background: #2563eb; color: white; padding: 5px 12px; border-radius: 20px; font-size: 12px; font-weight: bold; letter-spacing: 1px; }
        label { display: block; margin-bottom: 6px; font-weight: 600; color: #64748b; font-size: 12px; }
        input, select { width: 100%; padding: 10px; margin-bottom: 15px; border: 1px solid #cbd5e1; border-radius: 6px; box-sizing: border-box; background: #f8fafc; transition: all 0.3s; font-size: 13px; }
        input:focus, select:focus { border-color: #2563eb; outline: none; box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1); background: #fff; }
        button { background: #2563eb; color: white; border: none; padding: 10px 18px; border-radius: 6px; cursor: pointer; font-weight: bold; transition: 0.2s; }
        button:hover { background: #1d4ed8; transform: translateY(-1px); }
        .btn-purple { background: #8b5cf6; width: 100%; } .btn-purple:hover { background: #7c3aed; }
        .btn-green { background: #10b981; width: 100%; } .btn-green:hover { background: #059669; }
        .btn-blue { background: #0ea5e9; width: 100%; } .btn-blue:hover { background: #0284c7; }
        .btn-orange { background: #f97316; width: 100%; } .btn-orange:hover { background: #ea580c; }
        .grid { display: grid; grid-template-columns: 1.1fr 0.9fr; gap: 20px; }
        .status-grid { display: grid; grid-template-columns: 0.8fr 1.2fr 1fr; gap: 20px; align-items: center; }
        .status-msg { padding: 12px; border-radius: 6px; margin-bottom: 20px; background: #d1fae5; color: #065f46; border: 1px solid #a7f3d0; text-align: center; font-weight: bold; }
        .cloud-box { border: 1px solid #bfdbfe; padding: 15px; border-radius: 8px; margin-bottom: 15px; background: #eff6ff;}
        .cloud-title { font-size: 13px; font-weight: bold; color: #2563eb; margin-bottom: 10px;}
        .path-box { background: #f1f5f9; padding: 15px; border-radius: 8px; margin-bottom: 15px; border: 1px solid #e2e8f0; }
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
            <h2>💖 追剧管家 V10 <span style="font-size:12px; color:#2563eb;">统一滑点矩阵版</span></h2>
            <span class="badge">SYSTEM ONLINE</span>
        </div>
        {% if msg %}<div class="status-msg">{{ msg }}</div>{% endif %}
        <!-- ================= 上方：系统主控台 ================= -->
        <div class="mac-window" style="margin-bottom: 20px;">
            <div class="mac-header">
                <div class="mac-btn btn-close"></div><div class="mac-btn btn-min"></div><div class="mac-btn btn-max"></div>
                <div class="mac-title">系统与心跳控制台 (常规日志)</div>
            </div>
            <div class="console" id="logBoxSys" style="height: 260px;">Loading system terminal...</div>
        </div>

        <!-- ================= 下方：播放分流台 ================= -->
        <div class="mac-window" style="margin-bottom: 25px;">
            <div class="mac-header">
                <div class="mac-btn btn-close" style="background: #8b5cf6;"></div><div class="mac-btn btn-min" style="background: #10b981;"></div><div class="mac-btn btn-max" style="background: #0ea5e9;"></div>
                <div class="mac-title">播放链路解析台 (专属日志)</div>
            </div>
            <div class="console" id="logBoxPlay" style="height: 320px;">Loading playback terminal...</div>
        </div>
        <div class="card">
            <h3>📊 核心控制 & 凭证监控</h3>
            <div class="status-grid">
                <p style="color:#64748b; margin:0;">秒传库：<br><b style="color:#1e293b; font-size:1.8rem;">{{ cache_count }}</b> <span style="font-size:12px;">部剧集</span></p>
                <div style="color:#64748b; margin:0; font-size: 13px; line-height: 1.8; border-left: 2px solid #e2e8f0; padding-left: 15px;">
                    <b>🔑 四核驱动引擎凭证状态：</b><br>
                    {% for i in range(4) %}
                        {% set acc = cfg.accounts[i] if cfg.get('accounts') and i < cfg.accounts|length else {} %}
                        {% set sk = acc.get('session_key', '') %}
                        统一卡槽 {{ i+1 }}: 
                        {% if sk %}<span style="color:#10b981; font-weight:bold;">已就绪 (尾号{{ sk[-4:] }})</span>
                        {% else %}<span style="color:#f43f5e; font-weight:bold;">等待唤醒...</span>{% endif %}<br>
                    {% endfor %}
                    <br>
                    <b>🟠 移动云139:</b>
                    {% if cfg.openlist_host_139 %}<span style="color:#10b981; font-weight:bold;">配置已连接</span>
                    {% else %}<span style="color:#f43f5e; font-weight:bold;">未配置接口</span>{% endif %}
                </div>
                <div style="display:flex; flex-direction:column; gap:8px;">
                    <button type="button" onclick="syncOpenList('189', 'cas')" class="btn-purple" style="height:38px;">🔄 云端同步 (模式A: 稳健秒传)</button>
                    <button type="button" onclick="syncOpenList('189', 'cas_native')" class="btn-green" style="height:38px;">⚡ 云端同步 (模式B: 虚空直通)</button>
                    <button type="button" onclick="syncOpenList('139', 'cas')" class="btn-orange" style="height:38px;">🟠 云端同步 (移动云139 老模式)</button>
                    <button type="button" onclick="syncOpenList('139', 'cas_native')" style="background:#f59e0b; color:white; border:none; border-radius:6px; height:38px; cursor:pointer; font-weight:bold; transition: 0.2s;">🟠 云端同步 (139 原生直连)</button>
                    <button type="button" onclick="syncOpenList('direct', 'media')" class="btn-blue" style="height:38px;">🎬 云端同步 (常规真实视频)</button>
                    <button type="button" onclick="syncLocalCas()" style="background:#8b5cf6; color:white; border:none; border-radius:6px; height:38px; cursor:pointer; font-weight:bold; transition: 0.2s;" onmouseover="this.style.background='#7c3aed'" onmouseout="this.style.background='#8b5cf6'">🗂️ 批量扫描本地 CAS</button>
                    <button type="button" onclick="clearAllCache()" style="background:#ef4444; color:white; border:none; border-radius:6px; height:38px; cursor:pointer; font-weight:bold; transition: 0.2s;">🗑️ 一键清空189家庭/个人回收站</button>
                    <button type="button" onclick="clear139Recycle()" style="background:#ea580c; color:white; border:none; border-radius:6px; height:38px; cursor:pointer; font-weight:bold; transition: 0.2s; margin-top:5px;">🟠 一键清空139回收站</button>
                </div>
            </div>
        </div>
        <div class="card">
            <h3>⚙️ 综合配置中心</h3>
            <form method="POST" action="/admin/config">
            
                <div style="margin-bottom: 20px; padding: 15px; background: #fffbeb; border: 1px solid #fde68a; border-radius: 8px;">
                    <h4 style="color:#d97706; margin-top:0; border-bottom:none;">⚡ 模式 A：四核矩阵攻击队列 (容灾滑点)</h4>
                    <p style="font-size:12px; color:#b45309; margin-top:0; margin-bottom:10px;">当某个账号家庭云报错(如流量超标/null)，系统将根据此队列无缝退守至下一个可用云盘！</p>
                    <select name="mode_a_channel" style="border-color:#fcd34d;">
                        <option value="mix_f2p" {% if cfg.get('mode_a_channel', 'mix_f2p') == 'mix_f2p' %}selected{% endif %}>🔵🟣 混合双打：优先打空所有家庭云，无缝退守个人云</option>
                        <option value="mix_p2f" {% if cfg.get('mode_a_channel') == 'mix_p2f' %}selected{% endif %}>🟣🔵 混合双打：优先打空所有个人云，无缝退守家庭云</option>
                        <option value="family" {% if cfg.get('mode_a_channel') == 'family' %}selected{% endif %}>🔵 纯净模式：仅在 4 个家庭云间穿梭</option>
                        <option value="personal" {% if cfg.get('mode_a_channel') == 'personal' %}selected{% endif %}>🟣 纯净模式：仅在 4 个个人云间穿梭</option>
                    </select>
                </div>

                <div style="margin-bottom: 20px; padding: 15px; background: #f0f9ff; border: 1px solid #bae6fd; border-radius: 8px;">
                    <h4 style="color:#0369a1; margin-top:0; border-bottom:none;">⚡ 全局强制原生直通 (无视播放器选择)</h4>
                    <label style="display:flex; align-items:center; font-size:14px; color:#0c4a6e; cursor:pointer;">
                        <input type="checkbox" name="force_mode_b" value="true" {% if cfg.get('force_mode_b', '') | string | lower == 'true' %}checked{% endif %} style="width:20px; height:20px; margin-right:10px; margin-bottom:0;">
                        勾选此项，强行将所有189播放请求跃迁为原生直连 (模式B)！(关闭则遵循STRM参数)
                    </label>
                </div>
                
                <div style="margin-bottom: 20px;">
                    <label>首发节点分配策略 (Hash推荐)</label>
                    <select name="cloud_strategy">
                        <option value="hash" {% if cfg.cloud_strategy == 'hash' %}selected{% endif %}>🔗 剧名哈希绑定 (多卡槽容灾滑点)</option>
                        <option value="random" {% if cfg.cloud_strategy == 'random' %}selected{% endif %}>🎲 完全随机散列 (多卡槽容灾滑点)</option>
                        <option value="slot1" {% if cfg.cloud_strategy == 'slot1' %}selected{% endif %}>🥇 优先【卡槽 1】(失败无缝退守其他号)</option>
                        <option value="slot2" {% if cfg.cloud_strategy == 'slot2' %}selected{% endif %}>🥈 优先【卡槽 2】(失败无缝退守其他号)</option>
                        <option value="slot3" {% if cfg.cloud_strategy == 'slot3' %}selected{% endif %}>🥉 优先【卡槽 3】(失败无缝退守其他号)</option>
                        <option value="slot4" {% if cfg.cloud_strategy == 'slot4' %}selected{% endif %}>💎 优先【卡槽 4 大号】(失败无缝退守其他号)</option>
                    </select>
                </div>

                <div class="grid">
                    <!-- ================= 左侧栏 ================= -->
                    <div>
                        <h3 style="margin-top:0; border:none;">❇️ 189卡槽区</h3>
                        {% for i in range(4) %}
                        {% set acc = cfg.accounts[i] if cfg.get('accounts') and i < cfg.accounts|length else {} %}
                        <div class="cloud-box">
                            <div class="cloud-title">📌 核心装甲槽位 {{ i + 1 }} {% if i == 3 %}(支持外部Cookie免密注入大号){% endif %}</div>
                            <div style="display:flex; gap:10px;">
                                <input type="text" name="acc_user_{{ i }}" value="{{ acc.get('username', '') }}" placeholder="天翼云账号">
                                <input type="password" name="acc_pwd_{{ i }}" value="{{ acc.get('password', '') }}" placeholder="天翼云密码{% if i == 3 %} (留空则读取 cookies.json){% endif %}">
                            </div>
                            <div style="display:flex; gap:10px;">
                                <input type="text" name="acc_fam_id_{{ i }}" value="{{ acc.get('family_id', '') }}" placeholder="家庭云 Family ID">
                                <input type="text" name="acc_fam_fd_{{ i }}" value="{{ acc.get('family_folder_id', '') }}" placeholder="家庭云 目标目录 ID">
                            </div>
                            <input type="text" name="acc_per_fd_{{ i }}" value="{{ acc.get('personal_folder_id', '') }}" placeholder="个人云 目标目录 ID (不需要则留空)" style="margin-bottom:0;">
                            <input type="hidden" name="acc_sk_{{ i }}" value="{{ acc.get('session_key', '') }}">
                            <input type="hidden" name="acc_cookie_{{ i }}" value="{{ acc.get('cookie', '') }}">
                        </div>
                        {% endfor %}

                        <h3 style="margin-top:20px; border:none;">📁 绝对物理隔离路径设置</h3>
                        
                        <div class="path-box" style="background:#fffbeb; border-color:#fde68a;">
                            <h4 style="color:#d97706;">📁 本地原始 CAS 库 (其他脚本下载存放处)</h4>
                            <label>本地 CAS 源目录</label>
                            <input type="text" name="local_cas_source_dir" value="{{ cfg.local_cas_source_dir }}" required style="margin-bottom:0;">
                        </div>

                        <div class="path-box" style="background:#f0fdf4; border-color:#bbf7d0;">
                            <h4 style="color:#166534;">🌐 189 模式A (家庭云稳健秒传)</h4>
                            <label>云端 CAS 库扫描源目录</label>
                            <input type="text" name="network_cas_path" value="{{ cfg.network_cas_path }}" required>
                            <label>模式A 本地 STRM 独立保存路径</label>
                            <input type="text" name="local_strm_dir" value="{{ cfg.local_strm_dir }}" required style="margin-bottom:0;">
                        </div>

                        <div class="path-box" style="border-color: #10b981; background: #ecfdf5;">
                            <h4 style="color:#059669;">⚡ 189 模式B (极速虚空直通)</h4>
                            <label>云端临时挂载目录 (虚空造物目标路径)</label>
                            <input type="text" name="network_cas_path_native" value="{{ cfg.network_cas_path_native }}" required>
                            <label>模式B 本地 STRM 独立保存路径</label>
                            <input type="text" name="local_strm_dir_native" value="{{ cfg.local_strm_dir_native }}" required style="margin-bottom:0;">
                        </div>

                        <div class="path-box" style="border-color: #0ea5e9; background: #f0f9ff;">
                            <h4 style="color:#0284c7;">🎬 189 常规媒体 (真实视频解析)</h4>
                            <label>云端常规视频库 扫描源目录</label>
                            <input type="text" name="network_media_path" value="{{ cfg.network_media_path }}" required>
                            <label>常规媒体 本地 STRM 独立保存路径</label>
                            <input type="text" name="local_strm_dir_media" value="{{ cfg.local_strm_dir_media }}" required style="margin-bottom:0;">
                        </div>

                        <!-- 💡 【移到左侧】全局基础与 API -->
                        <h3 style="margin-top:20px; border:none;">🌐 全局基础与 API</h3>
                        <div class="path-box">
                            <label>基础外网域名 (Server Host)</label>
                            <input type="text" name="server_host" value="{{ cfg.server_host }}" required>
                            <label>189 OpenList 接口地址 (例如 5244)</label>
                            <input type="text" name="openlist_host" value="{{ cfg.openlist_host }}" required>
                            <label>189 OpenList 授权 Token</label>
                            <input type="password" name="openlist_token" value="{{ cfg.openlist_token }}" placeholder="填入189 OpenList Token">
                            <div style="display: flex; gap: 10px; margin-bottom: 15px;">
                                <div style="flex: 1;"><label>绝对销毁倒计时 (秒)</label><input type="number" name="delete_delay" value="{{ cfg.delete_delay }}" style="margin-bottom:0;" required></div>
                                <div style="flex: 1;"><label>预加载长效护盾 (秒)</label><input type="number" name="shield_delay" value="{{ cfg.shield_delay }}" style="margin-bottom:0;" required></div>
                            </div>
                            <div style="margin-bottom: 0;">
                                <label style="display:inline-block; font-weight:bold;">🛡️ 网关直链保鲜期 (秒):</label>
                                <input type="number" name="link_expire" value="{{ cfg.get('link_expire', 120) }}" style="width:80px; padding:3px;">
                            </div>
                        </div>
                    </div>

                    <!-- ================= 右侧栏 ================= -->
                    <div>
                        <!-- 💡 139 移动云盘顶到最上面 -->
                        <h3 style="margin-top:0; border:none;"><span style="color:#ea580c;">🟠</span> 139 移动云盘设置</h3>
                        <div class="path-box" style="border-color: #f97316; background: #fff7ed;">
                            <label style="color:#2563eb;"><b>【139 独立原生直连】</b></label>
                            
                            <!-- 👇 下面这块是新增的下拉选择框，缩进完全对齐 👇 -->
                            <div style="margin-top: 10px; margin-bottom: 15px; padding: 10px; background: #fff; border: 1px solid #fdba74; border-radius: 6px;">
                                <label style="color:#ea580c; font-weight:bold; margin-bottom: 6px;">🎯 139 首发账号分配策略 (失败自动滑点兜底)</label>
                                <select name="yun139_strategy" style="border-color:#f97316; margin-bottom: 0;">
                                    <option value="slot1" {% if cfg.get('yun139_strategy', 'slot1') == 'slot1' %}selected{% endif %}>🥇 优先【卡槽 1 主号】(报错自动切换到卡槽 2)</option>
                                    <option value="slot2" {% if cfg.get('yun139_strategy', 'slot1') == 'slot2' %}selected{% endif %}>🥈 优先【卡槽 2 备用号】(直接测试2号 / 报错自动切回1号)</option>
                                </select>
                            </div>

                            <label>🌐 139 控制面网关域名 (用于彻底删除/清空回收站)</label>
                            <input type="text" name="yun139_control_host" value="{{ cfg.yun139_control_host }}" placeholder="例如: https://personal-kd-njs.yun.139.com" style="margin-bottom:10px; border: 1px solid #f97316;">
                            
                            <label>139 本地原始 CAS 库 (上游下载的 cas 存放处)</label>
                            <input type="text" name="local_cas_source_dir_139" value="{{ cfg.local_cas_source_dir_139 }}" style="margin-bottom:10px; border: 1px solid #f59e0b;">
                            
                            <label>💣 139 后台物理销毁延时 (秒，默认 14400)</label>
                            <input type="number" name="delete_delay_139" value="{{ cfg.delete_delay_139 }}" placeholder="例如：14400" style="margin-bottom:10px; border: 1px solid #ef4444;">
                            
                            <label>🛡️ 139 直链保鲜期 (秒，原生与老模式通用)</label>
                            <input type="number" name="yun139_link_expire" value="{{ cfg.get('yun139_link_expire', 7200) }}" placeholder="例如：7200" style="margin-bottom:10px; border: 1px solid #10b981;">
                            
                            <!-- ================= 🥇 卡槽 1 (主号) ================= -->
                            <div style="background: #f8fafc; border: 1px solid #e2e8f0; border-radius: 8px; padding: 15px; margin-bottom: 15px;">
                                <label style="color:#10b981; font-size:14px; border-bottom:1px solid #10b981; padding-bottom:5px; margin-bottom:10px;">🥇 139 主账号 (卡槽 1)</label>
                                
                                <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px;">
                                    <label style="margin-bottom: 0; color: #475569;">🍪 主号邮箱 Cookie</label>
                                    <span onclick="const t = document.getElementById('cookie_textarea_1'); t.style.display = t.style.display === 'none' ? 'block' : 'none';" style="cursor: pointer; color: #10b981; font-size: 12px; font-weight: bold; padding: 3px 8px; background: #d1fae5; border: 1px solid #6ee7b7; border-radius: 4px; transition: 0.2s;">[ 展开 / 折叠 ]</span>
                                </div>
                                <span style="font-size:12px; color:#666; display:block; margin-bottom:5px;">登录 mail.10086.cn 后抓取 (包含 Os_SSo_Sid 和 RMKEY)</span>
                                <textarea id="cookie_textarea_1" name="yun139_mail_cookie" rows="3" placeholder="主号 Cookie: key1=value1;" style="width:100%; padding:10px; border: 1px solid #10b981; border-radius:6px; margin-bottom:10px; box-sizing:border-box; resize: vertical; display: {{ 'none' if cfg.yun139_mail_cookie else 'block' }};">{{ cfg.yun139_mail_cookie }}</textarea>
                                
                                <input type="text" name="yun139_phone" value="{{ cfg.yun139_phone }}" placeholder="主号 139 手机号" style="border-color:#10b981;">
                                <input type="password" name="yun139_password" value="{{ cfg.yun139_password }}" placeholder="主号 139 密码" style="border-color:#10b981;">
                                <input type="password" name="yun139_token" value="{{ cfg.yun139_token }}" placeholder="139 Base64 Token (老版授权备用，留空即可)" style="border-color:#10b981;">
                                
                                <!-- 👈 移到这里的专属目录 ID -->
                                <label style="color:#10b981; font-size:12px;">📁 主号专属临时缓存隔离目录 ID</label>
                                <input type="text" name="yun139_temp_folder_id" value="{{ cfg.yun139_temp_folder_id }}" placeholder="留空则默认存放根目录" style="border-color:#10b981; margin-bottom:0;">
                            </div>

                            <!-- ================= 🥈 卡槽 2 (备用号) ================= -->
                            <div style="background: #f5f3ff; border: 1px solid #ede9fe; border-radius: 8px; padding: 15px; margin-bottom: 10px;">
                                <label style="color:#8b5cf6; font-size:14px; border-bottom:1px solid #8b5cf6; padding-bottom:5px; margin-bottom:10px;">🥈 139 备用容灾号 (卡槽 2 - 主号报错时无缝接管)</label>
                                
                                <div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 5px;">
                                    <label style="margin-bottom: 0; color: #475569;">🍪 备用号邮箱 Cookie</label>
                                    <span onclick="const t = document.getElementById('cookie_textarea_2'); t.style.display = t.style.display === 'none' ? 'block' : 'none';" style="cursor: pointer; color: #8b5cf6; font-size: 12px; font-weight: bold; padding: 3px 8px; background: #ede9fe; border: 1px solid #c4b5fd; border-radius: 4px; transition: 0.2s;">[ 展开 / 折叠 ]</span>
                                </div>
                                <span style="font-size:12px; color:#666; display:block; margin-bottom:5px;">如不启用容灾则直接留空即可</span>
                                <textarea id="cookie_textarea_2" name="yun139_mail_cookie_2" rows="3" placeholder="备用号 Cookie: key1=value1;" style="width:100%; padding:10px; border: 1px solid #8b5cf6; border-radius:6px; margin-bottom:10px; box-sizing:border-box; resize: vertical; display: {{ 'none' if cfg.get('yun139_mail_cookie_2') else 'block' }};">{{ cfg.get('yun139_mail_cookie_2', '') }}</textarea>
                                
                                <input type="text" name="yun139_phone_2" value="{{ cfg.get('yun139_phone_2', '') }}" placeholder="备用号 139 手机号" style="border-color:#8b5cf6;">
                                <input type="password" name="yun139_password_2" value="{{ cfg.get('yun139_password_2', '') }}" placeholder="备用号 139 密码" style="border-color:#8b5cf6;">
                                <input type="password" name="yun139_token_2" value="{{ cfg.get('yun139_token_2', '') }}" placeholder="备用号 139 Base64 Token (备用，留空即可)" style="border-color:#8b5cf6;">
                                
                                <!-- 👈 备用号的专属目录 ID -->
                                <label style="color:#8b5cf6; font-size:12px;">📁 备用号专属临时缓存隔离目录 ID</label>
                                <input type="text" name="yun139_temp_folder_id_2" value="{{ cfg.get('yun139_temp_folder_id_2', '') }}" placeholder="留空则默认存放根目录" style="border-color:#8b5cf6; margin-bottom:0;">
                            </div>
                            
                            <label>139 原生直连本地 STRM 独立保存路径</label>
                            <input type="text" name="local_strm_dir_139_native" value="{{ cfg.local_strm_dir_139_native }}" style="margin-bottom:15px; border: 1px solid #f59e0b;">
                            
                            <hr style="border:0; border-top:1px dashed #fdba74; margin-bottom:15px;">
                            
                            <label>139 OpenList 接口 (老模式用)</label>
                            <input type="text" name="openlist_host_139" value="{{ cfg.openlist_host_139 }}">
                            <label>139 OpenList 授权 Token</label>
                            <input type="password" name="openlist_token_139" value="{{ cfg.openlist_token_139 }}">
                            <label>云端 139 CAS 扫描源目录</label>
                            <input type="text" name="network_cas_path_139" value="{{ cfg.network_cas_path_139 }}">
                            <label>139 老模式本地 STRM 独立保存路径</label>
                            <input type="text" name="local_strm_dir_139" value="{{ cfg.local_strm_dir_139 }}" style="margin-bottom:0;">
                        </div>

                        <!-- 💡 【移到右侧底部】消息推送 -->
                        <h3 style="margin-top:20px; border:none;">📱 消息推送</h3>
                        <div class="path-box">
                            <label>PushPlus Token (微信推送)</label><input type="text" name="pushplus_token" value="{{ cfg.pushplus_token }}" placeholder="留空则不推送">
                            <label>Telegram Bot Token</label><input type="text" name="tg_bot_token" value="{{ cfg.tg_bot_token }}">
                            <label>Telegram Chat ID</label><input type="text" name="tg_chat_id" value="{{ cfg.tg_chat_id }}">
                            <label>Telegram 代理地址 (Nekobox / Clash 等)</label><input type="text" name="tg_proxy" value="{{ cfg.get('tg_proxy', '') }}" placeholder="例如: http://127.0.0.1:7890 (留空则直连)" style="margin-bottom:0;">
                        </div>
                    </div>
                </div>
                <button type="submit" style="width:100%; margin-top:15px; font-size:16px;">💾 写入配置并重启四核矩阵引擎</button>
            </form>
        </div><div style="height: 40px;"></div>
    </div>
    <script>
        function syncOpenList(driveType, fileType='cas') { 
            let url = '/api/sync?drive=' + driveType + '&type=' + fileType;
            fetch(url).then(r => alert('同步指令已下发！请看上方日志。')); 
        }
        function syncLocalCas() { 
            if(confirm("将扫描本地配置的 CAS 目录，生成双轨 STRM，确认执行？")) {
                fetch('/api/sync_local').then(r => alert('本地扫描指令已下发！请看上方日志。')); 
            }
        }
        function clear139Recycle() { 
            if(confirm('⚠️ 确定要清空 139 移动云盘回收站吗？')) { 
                fetch('/api/clear_139', {method: 'POST'}).then(r => alert('🟠 139 回收站清理指令已下发！请看上方日志。')); 
            } 
        }
        function clearAllCache() { if(confirm('⚠️ 确定要清空吗？')) { fetch('/api/clear_all', {method: 'POST'}).then(r => alert('🚀 核弹已发射！')); } }
        function fetchLogs() {
            fetch('/admin/logs').then(r => r.json()).then(logs => {
                const box = document.getElementById('logBoxSys');
                const oldScrollTop = box.scrollTop, oldScrollHeight = box.scrollHeight, clientHeight = box.clientHeight;
                box.innerHTML = logs.map(l => `<span class="log-time">[${l.time}]</span><span class="log-msg ${l.msg.includes('✅') ? 'log-SUCCESS' : 'log-' + l.level}">${l.msg}</span><br>`).join('');
                if (oldScrollHeight - clientHeight - oldScrollTop < 30) { box.scrollTop = box.scrollHeight; } else { box.scrollTop = oldScrollTop; }
            });
            fetch('/admin/logs_play').then(r => r.json()).then(logs => {
                const box = document.getElementById('logBoxPlay');
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
def get_logs(): return jsonify(list(log_buffer_sys))

@app_main.route('/admin/logs_play')
def get_logs_play(): return jsonify(list(log_buffer_play))

# ==========================================
# ☁️ 天翼云核心功能类 (家庭云)
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

    def get_download_url(self, family_id, file_id, session_key, client_ua):
        url = "https://cloud.189.cn/api/open/family/file/getFileDownloadUrl.action"
        params = {"familyId": family_id, "fileId": file_id, "sessionKey": session_key}
        
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/118.0.0.0 Safari/537.36",
            "Cookie": f"SESSION_KEY={session_key}; cookieUserSession={session_key}",
            "Accept": "application/json;charset=UTF-8"
        }
        
        try:
            res = self.session.get(url, params=params, headers=headers, timeout=10).json()
            if 'fileDownloadUrl' in res:
                api_url = res['fileDownloadUrl'].replace('&amp;', '&')
                
                unwrap_headers = {
                    "User-Agent": client_ua if client_ua else headers["User-Agent"],
                    "Cookie": f"SESSION_KEY={session_key}; cookieUserSession={session_key}",
                    "Accept-Encoding": "identity"
                }
                
                unwrap_res = requests.get(api_url, headers=unwrap_headers, allow_redirects=False, timeout=10)
                status_code = unwrap_res.status_code
                
                if status_code in [301, 302, 303, 307, 308]: return unwrap_res.headers.get('Location')
                elif status_code == 200: return api_url
                else: raise Exception(f"底层破冰失败 (HTTP {status_code})")
                    
            raise Exception(f"提取网关链接失败: {res.get('msg', res)}")
        except Exception as e: raise e

family_client = TianyiFinalUploader()

# ==========================================
# 🛡️ 139 AES Crypto 底层 (对齐 139 官方加密)
# ==========================================
SBOX = (
0x63,0x7c,0x77,0x7b,0xf2,0x6b,0x6f,0xc5,0x30,0x01,0x67,0x2b,0xfe,0xd7,0xab,0x76,0xca,0x82,0xc9,0x7d,0xfa,0x59,0x47,0xf0,0xad,0xd4,0xa2,0xaf,0x9c,0xa4,0x72,0xc0,
0xb7,0xfd,0x93,0x26,0x36,0x3f,0xf7,0xcc,0x34,0xa5,0xe5,0xf1,0x71,0xd8,0x31,0x15,0x04,0xc7,0x23,0xc3,0x18,0x96,0x05,0x9a,0x07,0x12,0x80,0xe2,0xeb,0x27,0xb2,0x75,
0x09,0x83,0x2c,0x1a,0x1b,0x6e,0x5a,0xa0,0x52,0x3b,0xd6,0xb3,0x29,0xe3,0x2f,0x84,0x53,0xd1,0x00,0xed,0x20,0xfc,0xb1,0x5b,0x6a,0xcb,0xbe,0x39,0x4a,0x4c,0x58,0xcf,
0xd0,0xef,0xaa,0xfb,0x43,0x4d,0x33,0x85,0x45,0xf9,0x02,0x7f,0x50,0x3c,0x9f,0xa8,0x51,0xa3,0x40,0x8f,0x92,0x9d,0x38,0xf5,0xbc,0xb6,0xda,0x21,0x10,0xff,0xf3,0xd2,
0xcd,0x0c,0x13,0xec,0x5f,0x97,0x44,0x17,0xc4,0xa7,0x7e,0x3d,0x64,0x5d,0x19,0x73,0x60,0x81,0x4f,0xdc,0x22,0x2a,0x90,0x88,0x46,0xee,0xb8,0x14,0xde,0x5e,0x0b,0xdb,
0xe0,0x32,0x3a,0x0a,0x49,0x06,0x24,0x5c,0xc2,0xd3,0xac,0x62,0x91,0x95,0xe4,0x79,0xe7,0xc8,0x37,0x6d,0x8d,0xd5,0x4e,0xa9,0x6c,0x56,0xf4,0xea,0x65,0x7a,0xae,0x08,
0xba,0x78,0x25,0x2e,0x1c,0xa6,0xb4,0xc6,0xe8,0xdd,0x74,0x1f,0x4b,0xbd,0x8b,0x8a,0x70,0x3e,0xb5,0x66,0x48,0x03,0xf6,0x0e,0x61,0x35,0x57,0xb9,0x86,0xc1,0x1d,0x9e,
0xe1,0xf8,0x98,0x11,0x69,0xd9,0x8e,0x94,0x9b,0x1e,0x87,0xe9,0xce,0x55,0x28,0xdf,0x8c,0xa1,0x89,0x0d,0xbf,0xe6,0x42,0x68,0x41,0x99,0x2d,0x0f,0xb0,0x54,0xbb,0x16)
INV_SBOX = [0]*256
for _i, _v in enumerate(SBOX): INV_SBOX[_v] = _i
INV_SBOX = tuple(INV_SBOX)
RCON = (0x01,0x02,0x04,0x08,0x10,0x20,0x40,0x80,0x1b,0x36,0x6c,0xd8,0xab,0x4d)

def _xtime(a): return (((a << 1) ^ 0x1b) & 0xff) if (a & 0x80) else (a << 1)
def _mul(a, b):
    r = 0
    for _ in range(8):
        if b & 1: r ^= a
        a = _xtime(a)
        b >>= 1
    return r

def _expand_key(key):
    nk = len(key) // 4
    nr = nk + 6
    kc = [list(key[i:i+4]) for i in range(0, len(key), 4)]
    for i in range(nk, 4 * (nr + 1)):
        t = list(kc[i-1])
        if i % nk == 0:
            t = [SBOX[b] for b in (t[1:] + t[:1])]
            t[0] ^= RCON[i//nk - 1]
        elif nk > 6 and i % nk == 4:
            t = [SBOX[b] for b in t]
        kc.append([kc[i-nk][j] ^ t[j] for j in range(4)])
    return [sum(kc[r*4:(r+1)*4], []) for r in range(nr + 1)], nr

def _shift_rows(s):
    s[1],s[5],s[9],s[13] = s[5],s[9],s[13],s[1]
    s[2],s[6],s[10],s[14] = s[10],s[14],s[2],s[6]
    s[3],s[7],s[11],s[15] = s[15],s[3],s[7],s[11]

def _inv_shift_rows(s):
    s[1],s[5],s[9],s[13] = s[13],s[1],s[5],s[9]
    s[2],s[6],s[10],s[14] = s[10],s[14],s[2],s[6]
    s[3],s[7],s[11],s[15] = s[7],s[11],s[15],s[3]

def _mix_columns(s):
    for c in range(4):
        a = s[c*4:c*4+4]
        t = a[0]^a[1]^a[2]^a[3]
        u = a[0]
        s[c*4+0] ^= t ^ _xtime(a[0]^a[1])
        s[c*4+1] ^= t ^ _xtime(a[1]^a[2])
        s[c*4+2] ^= t ^ _xtime(a[2]^a[3])
        s[c*4+3] ^= t ^ _xtime(a[3]^u)

def _inv_mix_columns(s):
    for c in range(4):
        a = s[c*4:c*4+4]
        s[c*4+0] = _mul(a[0],14)^_mul(a[1],11)^_mul(a[2],13)^_mul(a[3],9)
        s[c*4+1] = _mul(a[0],9)^_mul(a[1],14)^_mul(a[2],11)^_mul(a[3],13)
        s[c*4+2] = _mul(a[0],13)^_mul(a[1],9)^_mul(a[2],14)^_mul(a[3],11)
        s[c*4+3] = _mul(a[0],11)^_mul(a[1],13)^_mul(a[2],9)^_mul(a[3],14)

def _aes_block(block, rks, nr, decrypt=False):
    s = list(block)
    if not decrypt:
        for i in range(16): s[i] ^= rks[0][i]
        for r in range(1, nr):
            for i in range(16): s[i] = SBOX[s[i]]
            _shift_rows(s); _mix_columns(s)
            for i in range(16): s[i] ^= rks[r][i]
        for i in range(16): s[i] = SBOX[s[i]]
        _shift_rows(s)
        for i in range(16): s[i] ^= rks[nr][i]
    else:
        for i in range(16): s[i] ^= rks[nr][i]
        for r in range(nr-1, 0, -1):
            _inv_shift_rows(s)
            for i in range(16): s[i] = INV_SBOX[s[i]]
            for i in range(16): s[i] ^= rks[r][i]
            _inv_mix_columns(s)
        _inv_shift_rows(s)
        for i in range(16): s[i] = INV_SBOX[s[i]]
        for i in range(16): s[i] ^= rks[0][i]
    return bytes(s)

def aes_cbc_encrypt(data, key, iv):
    rks, nr = _expand_key(key)
    pad_len = 16 - len(data) % 16
    data += bytes([pad_len]) * pad_len
    out = b""
    prev = iv
    for i in range(0, len(data), 16):
        blk = bytes(a ^ b for a, b in zip(data[i:i+16], prev))
        prev = _aes_block(blk, rks, nr)
        out += prev
    return out

def aes_cbc_decrypt(data, key, iv):
    rks, nr = _expand_key(key)
    out = b""
    prev = iv
    for i in range(0, len(data), 16):
        blk = _aes_block(data[i:i+16], rks, nr, decrypt=True)
        out += bytes(a ^ b for a, b in zip(blk, prev))
        prev = data[i:i+16]
    pad_len = out[-1]
    return out[:-pad_len]

def aes_ecb_decrypt(data, key):
    rks, nr = _expand_key(key)
    out = b"".join(_aes_block(data[i:i+16], rks, nr, decrypt=True) for i in range(0, len(data), 16))
    pad_len = out[-1]
    return out[:-pad_len]

class NoRedirect(urllib.request.HTTPRedirectHandler):
    def redirect_request(self, req, fp, code, msg, headers, newurl): return None

# ==========================================
# 🟠 移动云盘 139 原生直连引擎 (主备双槽容灾升级版)
# ==========================================
class Cloud139NativeEngine:
    KEY1 = bytes.fromhex("73634235495062495331515373756c734e7253306c673d3d")
    KEY2 = bytes.fromhex("7150714477323633586746674c337538")                
    CLIENT_KEY = "l3TryM&Q+X7@dzwk)qP"
    
    HEADERS_BASE = {
        "Accept": "application/json, text/plain, */*",
        "Cms-Device": "default",
        "mcloud-channel": "1000101",
        "mcloud-client": "10701",
        "mcloud-version": "7.14.0",
        "Origin": "https://yun.139.com",
        "Referer": "https://yun.139.com/w/",
        "x-DeviceInfo": "||9|7.14.0|chrome|120.0.0.0|||windows 10||zh-CN|||",
        "x-huawei-channelSrc": "10000034",
        "x-inner-ntwk": "2",
        "x-m4c-caller": "PC",
        "x-m4c-src": "10002",
        "x-SvcType": "1",
        "Inner-Hcy-Router-Https": "1",
    }

    PERSONAL_HEADERS = {
        "Caller": "web",
        "Mcloud-Route": "001",
        "X-Yun-Api-Version": "v1",
        "X-Yun-App-Channel": "10000034",
        "X-Yun-Channel-Source": "10000034",
        "X-Yun-Client-Info": "||9|7.14.0|chrome|120.0.0.0|||windows 10||zh-CN|||dW5kZWZpbmVk||",
        "X-Yun-Module-Type": "100",
        "X-Yun-SvcType": "1",
    }

    def __init__(self):
        self.url_cache = {}
        # 磁盘持久化文件：记录未删除的临时文件 ID
        self.tracking_db_file = os.path.join(DB_DIR, ".139_active_fids.json")
    
    def _add_active_fid(self, slot_idx, fid):
        try:
            data = {1: [], 2: []}
            if os.path.exists(self.tracking_db_file):
                with open(self.tracking_db_file, 'r', encoding='utf-8') as f:
                    data = json.load(f)
            data_slot = data.get(str(slot_idx), [])
            if str(fid) not in data_slot:
                data_slot.append(str(fid))
            data[str(slot_idx)] = data_slot
            with open(self.tracking_db_file, 'w', encoding='utf-8') as f:
                json.dump(data, f)
        except Exception as e:
            logger.error(f"[139持久化] 记录 fid 失败: {e}")

    def _pop_active_fids(self, slot_idx):
        try:
            if not os.path.exists(self.tracking_db_file):
                return []
            with open(self.tracking_db_file, 'r', encoding='utf-8') as f:
                data = json.load(f)
            fids = data.get(str(slot_idx), [])
            # 取出后清空该卡槽的记录
            data[str(slot_idx)] = []
            with open(self.tracking_db_file, 'w', encoding='utf-8') as f:
                json.dump(data, f)
            return fids
        except:
            return []

    def _md5(self, s): return hashlib.md5(s.encode()).hexdigest()
    def _sha1_hex(self, s): return hashlib.sha1(s.encode()).hexdigest()
    def sorted_json(self, obj): return json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False)

    # 【新增】动态获取凭证保存路径，支持多卡槽
    def _get_token_cache_path(self, slot_idx):
        return os.path.join(DB_DIR, f".cas139_token_{slot_idx}.json")

    def cache_token(self, auth, account, slot_idx=1):
        try:
            with open(self._get_token_cache_path(slot_idx), "w") as f: 
                json.dump({"auth": auth, "account": account, "saved": int(time.time())}, f)
        except OSError: pass

    def load_cached_token(self, slot_idx=1):
        try:
            d = json.load(open(self._get_token_cache_path(slot_idx)))
            auth = d["auth"]
            decoded = base64.b64decode(auth).decode()
            parts = decoded.split(":")
            if len(parts) < 3: return None
            toks = parts[2].split("|")
            if len(toks) < 4: return None
            exp = int(toks[3]) // 1000
            if exp > time.time(): return auth
        except: pass
        return None

    def login(self, account, password, cookie_str="", slot_idx=1):
        import requests
        logger.info(f"[139引擎] 开始执行 139 SSO 鉴权流程 (卡槽 {slot_idx} - 账号: {account})...")
        
        cfg = read_config()
        if not cookie_str: 
            suffix = "" if slot_idx == 1 else "_2"
            cookie_str = cfg.get(f"yun139_mail_cookie{suffix}", "").strip()

        sid = ""
        rmkey = ""
        
        if cookie_str:
            for part in cookie_str.split(";"):
                part = part.strip()
                if part.startswith("Os_SSo_Sid=") or part.startswith("Os_SSO_Sid="): sid = part[11:]
                elif part.startswith("RMKEY="): rmkey = part[6:]
            if sid and rmkey:
                logger.info(f"[139引擎] 🍪 成功从邮箱 Cookie 提取极速免签金牌...")

        if not sid or not rmkey:
            logger.warning(f"[139引擎] ⚠️ 未提取到完整 Cookie，尝试账号密码直登 (此方式极易触发滑块拦截)...")
            import http.cookiejar
            jar = http.cookiejar.CookieJar()
            opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar))
            b64_acc = base64.b64encode(account.encode()).decode()
            cguid = str(int(time.time() * 1000))
            default_page = f"https://mail.10086.cn/default.html?s=1&v=0&u={b64_acc}&m=1&ec=S001&resource=indexLogin&clientid=1003&auto=on&cguid={cguid}"
            
            try: opener.open(urllib.request.Request(default_page, headers={"User-Agent": "Mozilla/5.0"}), timeout=30).read()
            except: pass
            
            mail_cookies = "; ".join(f"{c.name}={c.value}" for c in jar)
            form_data = urllib.parse.urlencode({
                "UserName": account, "passOld": "", "auto": "on",
                "Password": self._sha1_hex("fetion.com.cn:" + password),
                "webIndexPagePwdLogin": "1", "pwdType": "1",
                "clientId": "1003", "authType": "2",
            })
            
            req = urllib.request.Request("https://mail.10086.cn/Login/Login.ashx", data=form_data.encode(), method="POST", headers={
                "Content-Type": "application/x-www-form-urlencoded",
                "Cookie": mail_cookies, "Referer": default_page,
                "Origin": "https://mail.10086.cn", "User-Agent": "Mozilla/5.0"
            })
            
            class NoRedirect(urllib.request.HTTPRedirectHandler):
                def redirect_request(self, req, fp, code, msg, headers, newurl): return None
            no_redirect = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(jar), NoRedirect())
            try:
                resp = no_redirect.open(req, timeout=30)
                location = resp.headers.get("Location", "")
            except urllib.error.HTTPError as e:
                location = e.headers.get("Location", "")
                
            sid_m = re.search(r"sid=([^&]+)", location)
            sid = sid_m.group(1) if sid_m else ""
            if not sid:
                for c in jar:
                    if c.name in ("Os_SSo_Sid", "Os_SSO_Sid"): sid = c.value
            if not sid: raise Exception("密码登录被官方风控拦截。请去浏览器抓取最新 Cookie！")
            
            for c in jar:
                if c.name == "RMKEY": rmkey = c.value

        if not sid or not rmkey: raise Exception("鉴权参数缺失：未获取到 sid 或 rmkey。")

        art_url = f"https://smsrebuild1.mail.10086.cn/setting/s?func={urllib.parse.quote('umc:getArtifact')}&sid={sid}&cguid={str(int(time.time() * 1000))}"
        xml_body = b"<object><int name=\"appId\">16</int><int name=\"envAuth\">1</int></object>"

        try:
            art_resp = requests.post(art_url, data=xml_body, headers={
                "Content-Type": "application/xml",
                "Cookie": f"RMKEY={rmkey}",
                "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/120.0.0.0"
            }, timeout=15)
            art_body = art_resp.text
        except Exception as e:
            raise Exception(f"获取 artifact 网络异常: {e}")

        m = re.search(r'"artifact"\s*:\s*"([^"]+)"', art_body) or \
            re.search(r'artifact\s*[:=]\s*"([^"]+)"', art_body) or \
            re.search(r'<string[^>]*name="artifact"[^>]*>\s*([^<]+)\s*</string>', art_body)

        if not m: raise Exception("未取到 artifact，请检查日志。")
        artifact = m.group(1).strip()

        body = {
            "clientkey_decrypt": self.CLIENT_KEY,
            "clienttype": "886",
            "cpid": "507",
            "dycpwd": artifact,
            "extInfo": {"ifOpenAccount": "0"},
            "loginMode": "0",
            "msisdn": str(account).strip(),
            "pintype": "13",
            "secinfo": self._sha1_hex("fetion.com.cn:" + artifact).upper(),
            "version": "20250901",
        }
        
        iv = os.urandom(16)
        payload = base64.b64encode(iv + aes_cbc_encrypt(self.sorted_json(body).encode(), self.KEY1, iv)).decode()

        final_resp = requests.post("https://user-njs.yun.139.com/user/thirdlogin", data=payload.encode(), headers={
            "hcy-cool-flag": "1",
            "x-huawei-channelSrc": "10246600",
            "x-sdk-channelSrc": "",                                               
            "x-MM-Source": "0",                                                  
            "x-UserAgent": "android|23116PN5BC|android15|1.2.6|||1440x3200|10246600",     
            "x-DeviceInfo": "4|127.0.0.1|5|1.2.6|Xiaomi|23116PN5BC||02-00-00-00-00-00|android 15|1440x3200|android|||", 
            "Content-Type": "text/plain;charset=UTF-8",
            "Accept-Encoding": "gzip",
            "User-Agent": "okhttp/3.12.2"
        }, timeout=30)

        txt = final_resp.text.strip()
        if txt.startswith("{"):
            layer1 = json.loads(txt)
        else:
            blob = base64.b64decode(txt)
            layer1 = json.loads(aes_cbc_decrypt(blob[16:], self.KEY1, blob[:16]))
            
        if not layer1.get("data"): raise Exception(f"兑换凭证失败: {layer1.get('msg', layer1)}")

        inner = aes_ecb_decrypt(bytes.fromhex(layer1["data"]), self.KEY2)
        final_data = json.loads(inner)

        auth_token = final_data.get("authToken", "")
        if not auth_token: raise Exception("最后一步兑换 authToken 失败！")

        account_ret = final_data.get("account", account)
        auth = base64.b64encode(f"pc:{account_ret}:{auth_token}".encode()).decode()
        self.cache_token(auth, account_ret, slot_idx)
        logger.info(f"[139引擎] ✅ 鉴权圆满成功！(卡槽 {slot_idx} 已就绪)")
        return auth

    # 100%保留你原来的函数名，外接老接口不出错
    def ensure_auth(self, slot_idx=1):
        cfg = read_config()
        # 1. 优先检查手动 Token
        if slot_idx == 1 and cfg.get("yun139_token"): return cfg["yun139_token"].strip()
        if slot_idx == 2 and cfg.get("yun139_token_2"): return cfg["yun139_token_2"].strip()
        
        token_file = self._get_token_cache_path(slot_idx)
        
        # 2. 检查本地缓存：只要距离过期还有 10 分钟以上，直接复用！
        if os.path.exists(token_file):
            try:
                with open(token_file, 'r', encoding='utf-8') as f:
                    d = json.load(f)
                auth = d.get("auth", "")
                if auth:
                    decoded = base64.b64decode(auth).decode()
                    exp = int(decoded.split(":")[2].split("|")[3]) // 1000
                    if exp - time.time() > 600:
                        return auth
            except:
                pass
        
        # 3. 确实没缓存或真过期了，才发起登录
        suffix = "" if slot_idx == 1 else "_2"
        acc = cfg.get(f"yun139_phone{suffix}")
        pwd = cfg.get(f"yun139_password{suffix}")
        cookie_str = cfg.get(f"yun139_mail_cookie{suffix}")
        
        if not acc and not cookie_str: 
            raise Exception(f"未配置 139 卡槽 {slot_idx}")
            
        logger.info(f"[139凭证] 卡槽 {slot_idx} 凭证已过期或不存在，执行登录...")
        return self.login(acc, pwd, cookie_str, slot_idx)

    def cal_sign(self, body, ts, rand_str):
        enc = urllib.parse.quote(body, safe="~!*'()-._")
        joined = "".join(sorted(enc))
        b64 = base64.b64encode(joined.encode()).decode()
        return self._md5(self._md5(b64) + self._md5(f"{ts}:{rand_str}")).upper()

    def get_account(self, auth):
        return base64.b64decode(auth).decode("utf-8", "replace").split(":")[1]

    def cloud_request(self, url, body_obj, auth, extra_headers=None, timeout=30):
        body = json.dumps(body_obj, separators=(",", ":"), ensure_ascii=False)
        ts = time.strftime("%Y-%m-%d %H:%M:%S")
        rand = "".join(random.choices(string.ascii_letters + string.digits, k=16))
        headers = dict(self.HEADERS_BASE)
        headers["Content-Type"] = "application/json;charset=UTF-8"
        
        auth_clean = auth.strip()
        headers["Authorization"] = auth_clean if auth_clean.startswith("Basic") else "Basic " + auth_clean
        headers["mcloud-sign"] = f"{ts},{rand},{self.cal_sign(body, ts, rand)}"
        
        if extra_headers: headers.update(extra_headers)
            
        req = urllib.request.Request(url, data=body.encode("utf-8"), headers=headers, method="POST")
        try:
            with urllib.request.urlopen(req, timeout=timeout) as r:
                resp = json.loads(r.read().decode("utf-8", "replace"))
        except urllib.error.HTTPError as e:
            raise Exception(f"HTTP {e.code}: {e.read().decode('utf-8', 'replace')[:300]}")
            
        if isinstance(resp, dict) and resp.get("success") is False:
            raise Exception(f"API返回失败: code={resp.get('code')} message={resp.get('message')}")
        return resp

    def get_personal_host(self, auth):
        resp = self.cloud_request("https://user-njs.yun.139.com/user/route/qryRoutePolicy", {
            "userInfo": {"userType": 1, "accountType": 1, "accountName": self.get_account(auth)},
            "modAddrType": 1}, auth)
        data = resp.get("data") or {}
        for pol in data.get("routePolicyList", []):
            if pol.get("modName") == "personal": return pol["httpsUrl"].rstrip("/")
        raise Exception(f"路由策略响应异常")

    def make_part_infos(self, size):
        part_size = 512 * 1024 * 1024 if size // (1024 ** 3) > 30 else 100 * 1024 * 1024
        n = (size + part_size - 1) // part_size
        return [{"partNumber": i + 1, "partSize": min(part_size, size - i * part_size), "parallelHashCtx": {"partOffset": i * part_size}} for i in range(min(n, 100))]

    def _async_delete(self, file_id, delay, name, auth=None):
        import time
        try:
            m, s = divmod(int(delay), 60)
            time_str = f"{m}分{s}秒" if m > 0 else f"{s}秒"
            
            logger.info(f"[清理预约] 139云盘：{name} 将在 {time_str}后 永久销毁")
            time.sleep(delay)
            
            if not auth: auth = self.ensure_auth(1)
            
            cfg = read_config()
            control_host = cfg.get("yun139_control_host", "").strip() or "https://personal-kd-njs.yun.139.com"
            delete_url = control_host.rstrip('/') + "/hcy/file/batchDelete"
            
            body = {"fileIds": [str(file_id)]}
            resp = self.cloud_request(delete_url, body, auth, self.PERSONAL_HEADERS)
                
            if resp.get("success") or resp.get("code") == "0":
                logger.info(f"[空间释放] 💥 139云盘：{name} 已永久销毁，物理空间即时释放！")
            else:
                logger.warning(f"[销毁异常] 139云盘：{name} 失败: {resp}")
        except Exception as e:
            logger.error(f"[销毁报错] 139云盘：{name}: {e}")

    def empty_recycle_bin(self):
        cfg = read_config()
        control_host = cfg.get("yun139_control_host", "").strip() or "https://personal-kd-njs.yun.139.com"
        del_file_url = control_host.rstrip('/') + "/hcy/file/batchDelete"
        clear_url = control_host.rstrip('/') + "/hcy/recyclebin/clear"

        for slot in [1, 2]:
            suffix = "" if slot == 1 else "_2"
            if not cfg.get(f"yun139_phone{suffix}") and not cfg.get(f"yun139_mail_cookie{suffix}"):
                continue

            try:
                auth = self.ensure_auth(slot)
                
                # 1. 从磁盘持久化文件中读取即便重启也保留的 fid 列表
                fids = self._pop_active_fids(slot)
                if fids:
                    logger.info(f"[139全扫] 🧹 发现卡槽 {slot} 记录的 {len(fids)} 个临时文件(含历史跨重启文件)，正在移入回收站...")
                    self.cloud_request(del_file_url, {"fileIds": fids}, auth, self.PERSONAL_HEADERS)
                
                # 2. 彻底清空回收站物理释放空间
                logger.info(f"[清理个人] 🟠 正在彻底清空卡槽 {slot} 回收站...")
                resp = self.cloud_request(clear_url, {}, auth, self.PERSONAL_HEADERS)
                if resp.get("success") or resp.get("code") == "0":
                    logger.info(f"[清理个人] 💥 卡槽 {slot} 回收站已彻底清空，空间全部真实释放！")
                else:
                    logger.warning(f"[清理异常] 卡槽 {slot} 清空失败: {resp}")
            except Exception as e:
                logger.error(f"[清理报错] 卡槽 {slot}: {e}")

    # ==========================
    # 🌟 核心：主备容灾切换逻辑 🌟
    # ==========================
    def get_direct_link(self, sha256_hash, file_size, file_name, parent_id="/"):
        import threading
        now = time.time()
        
        # 1. 查缓存
        if sha256_hash in self.url_cache:
            cached_url, exp = self.url_cache[sha256_hash]
            if now < exp:
                logger.info(f"[直链缓存] ⚡ 命中本地内存直链缓存，免查官方 API: {file_name}")
                return cached_url

        cfg = read_config()
        # 2. 读取后台策略：决定谁首发，谁兜底
        strategy = cfg.get("yun139_strategy", "slot1")
        first_slot = 2 if strategy == "slot2" else 1
        second_slot = 1 if first_slot == 2 else 2

        try:
            # 首发尝试
            return self._do_get_direct_link(first_slot, sha256_hash, file_size, file_name, parent_id, now)
        except Exception as e1:
            logger.warning(f"⚠️ [云盘容灾] 首发账号(卡槽{first_slot})获取失败: {e1}，正在无缝切换至卡槽{second_slot}...")
            try:
                # 兜底尝试
                return self._do_get_direct_link(second_slot, sha256_hash, file_size, file_name, parent_id, now)
            except Exception as e2:
                logger.error(f"❌ [阵亡警报] 两个卡槽均无法承载: {e2}")
                raise Exception(f"139所有卡槽均失效！首选({first_slot})错: {e1} | 备用({second_slot})错: {e2}")

    def _do_get_direct_link(self, slot_idx, sha256_hash, file_size, file_name, parent_id, now):
        cfg = read_config()
        
        try: delay_sec = int(cfg.get('delete_delay_139', 14400))
        except: delay_sec = 14400
        
        try: cache_ttl = int(cfg.get('yun139_link_expire', 7200))
        except: cache_ttl = 7200
        
        # 👈 智能判断：如果是1号槽取原ID，2号槽取专属ID
        folder_key = "yun139_temp_folder_id" if slot_idx == 1 else "yun139_temp_folder_id_2"
        custom_folder_id = cfg.get(folder_key, "").strip()
        if custom_folder_id:
            parent_id = custom_folder_id
            
        auth = self.ensure_auth(slot_idx)
        try: host = self.get_personal_host(auth)
        except:
            # Token如果失效，删掉重取
            token_file = self._get_token_cache_path(slot_idx)
            os.path.exists(token_file) and os.remove(token_file)
            auth = self.ensure_auth(slot_idx)
            host = self.get_personal_host(auth)

        body = {
            "contentHash": sha256_hash, "contentHashAlgorithm": "SHA256", "contentType": "application/octet-stream", "parallelUpload": False,
            "partInfos": self.make_part_infos(file_size), "size": file_size, "parentFileId": parent_id, "name": file_name, "type": "file", "fileRenameMode": "auto_rename",
        }
        
        resp = self.cloud_request(host + "/file/create", body, auth, self.PERSONAL_HEADERS)
        data = resp.get("data", {})
        
        fid = ""
        if data.get("rapidUpload") and not data.get("partInfos"):
            fid = data.get("fileId", "")
        elif data.get("exist"):
            fid = data.get("fileId", "")
            if not fid: raise Exception("云端已存在该文件但未返回 fileId")
            
        if not fid: raise Exception("秒传未命中，CAS已失效或服务端要求真实切片上传")

        dl_resp = self.cloud_request(host + "/file/getDownloadUrl", {"fileId": str(fid)}, auth, self.PERSONAL_HEADERS)
        dl_data = dl_resp.get("data")
        final_url = ""
        
        if isinstance(dl_data, str) and dl_data.startswith("http"): final_url = dl_data
        elif isinstance(dl_data, dict):
            for k in ("downloadUrl", "downloadURL", "url", "fileUrl", "cdnUrl"):
                if dl_data.get(k): 
                    final_url = dl_data[k]
                    break
                    
        if not final_url: raise Exception("秒传成功，但无法解析下载链接")
        
        self.url_cache[sha256_hash] = (final_url, now + cache_ttl)
        self._add_active_fid(slot_idx, str(fid))
        
        # 将明确的 auth 传入异步销毁函数，避免它默认拿卡槽 1
        if delay_sec > 0:
            import threading
            threading.Thread(target=self._async_delete, args=(fid, delay_sec, file_name, auth), daemon=True).start()

        return final_url

cloud139_native = Cloud139NativeEngine()

# ==========================================
# ☁️ 天翼云核心功能类 (个人云专属)
# ==========================================
class TianyiPersonalUploader:
    def __init__(self):
        self.rsa_keys = {}
        self.session = requests.Session()
        self.app_key = '600100422'

    def _md5(self, text): return hashlib.md5(text.encode('utf-8')).hexdigest().upper()
    def _random_string(self, length=16): return ''.join(random.choices('0123456789abcdef', k=length))
    def _get_timestamp(self): return str(int(time.time() * 1000))
    def _encode_uri(self, text): return urllib.parse.quote(text, safe='~()*!.\'-_')
    def _get_slice_size(self, file_size):
        try: size = int(file_size)
        except: return '10485760'  
        D = 10485760
        if size > D * 2 * 999: return str(max(math.ceil(size / 1999 / D), 5) * D)
        elif size > D * 999: return str(D * 2)
        return str(D)

    def get_base_headers(self, cookie):
        return {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36', 'Cookie': cookie, 'Accept': 'application/json;charset=UTF-8'}

    def get_rsa_key(self, session_key, cookie):
        if session_key in self.rsa_keys: return self.rsa_keys[session_key]
        ts = self._get_timestamp()
        sign = self._md5(f"AppKey={self.app_key}&Timestamp={ts}")
        url = f"https://cloud.189.cn/api/security/generateRsaKey.action?sessionKey={urllib.parse.quote(session_key)}"
        h = self.get_base_headers(cookie)
        h.update({'Sign-Type': '1', 'Signature': sign, 'Timestamp': ts, 'AppKey': self.app_key, 'SessionKey': session_key})
        try:
            res = self.session.get(url, headers=h, timeout=10).json()
            if 'pubKey' in res:
                self.rsa_keys[session_key] = res
                return res
            raise Exception(f"获取个人云公钥失败: {res}")
        except Exception as e:
            raise e

    def build_request(self, params, uri, req_id, session_key, cookie):
        rsa = self.get_rsa_key(session_key, cookie)
        ukey, ts = self._random_string(16), self._get_timestamp()
        p_str = '&'.join([f"{k}={v}" for k, v in params.items()])
        enc_p = AES.new(ukey.encode('utf-8'), AES.MODE_ECB).encrypt(pad(p_str.encode('utf-8'), 16)).hex().upper()
        rsa_c = PKCS1_v1_5.new(RSA.import_key(f"-----BEGIN PUBLIC KEY-----\n{rsa['pubKey']}\n-----END PUBLIC KEY-----"))
        enc_t = base64.b64encode(rsa_c.encrypt(ukey.encode('utf-8'))).decode('utf-8')
        sign = hmac.new(ukey.encode('utf-8'), f"SessionKey={session_key}&Operate=GET&RequestURI={uri}&Date={ts}&params={enc_p}".encode('utf-8'), hashlib.sha1).hexdigest().upper()
        h = self.get_base_headers(cookie)
        h.update({'X-Request-Date': ts, 'X-Request-ID': req_id, 'SessionKey': session_key, 'EncryptionText': enc_t, 'PkId': rsa['pkId'], 'Signature': sign})
        return f"https://upload.cloud.189.cn{uri}?params={enc_p}", h

    def get_personal_items(self, folder_id, cookie):
        url = f"https://cloud.189.cn/api/open/file/listFiles.action?folderId={folder_id}&pageNum=1&pageSize=1000"
        try:
            res = self.session.get(url, headers=self.get_base_headers(cookie), timeout=10).json()
            ao = res.get('fileListAO', {})
            return [{'fileName': f['name'], 'fileId': f['id'], 'isFolder': 'id' in f and 'fileId' not in f} for f in ao.get('folderList', []) + ao.get('fileList', [])]
        except: return []

    def delete_item(self, file_id, cookie):
        url = f"https://cloud.189.cn/api/open/file/deleteFile.action?fileId={file_id}"
        try: return self.session.get(url, headers=self.get_base_headers(cookie), timeout=10).status_code == 200
        except: return False
        
    def empty_personal_recycle(self, session_key, cookie):
        url = "https://cloud.189.cn/api/open/batch/createBatchTask.action"
        payload = {"type": "EMPTY_RECYCLE", "taskInfos": "[]", "targetFolderId": "", "sessionKey": session_key}
        try:
            res = self.session.post(url, data=payload, headers=self.get_base_headers(cookie), timeout=10).json()
            if str(res.get("res_code")) == "0": return True
        except: pass
        return False

    def rapid_upload_personal(self, folder_id, md5, size, smd5, safe_name, session_key, cookie):
        f_md5, s_md5 = str(md5).upper(), str(smd5).upper()
        items = self.get_personal_items(folder_id, cookie)
        for i in items:
            if i['fileName'] == safe_name or f_md5 in i['fileName']: return i['fileId']

        req_id = str(uuid.uuid4())
        slice_size = self._get_slice_size(size)
        
        init_p = {'parentFolderId': folder_id, 'fileName': self._encode_uri(safe_name), 'fileSize': str(size), 'sliceSize': slice_size, 'fileMd5': f_md5, 'sliceMd5': s_md5, 'lazyCheck': '1', 'opertype': '3'}
        url, h = self.build_request(init_p, '/person/initMultiUpload', req_id, session_key, cookie)
        res = self.session.get(url, headers=h, timeout=10).json()
        
        if res.get('code') == 'SUCCESS':
            up_id = res['data']['uploadFileId']
            ck_p = {'fileMd5': f_md5, 'sliceMd5': s_md5, 'uploadFileId': up_id}
            url, h = self.build_request(ck_p, '/person/checkTransSecond', req_id, session_key, cookie)
            if self.session.get(url, headers=h, timeout=10).json().get('data', {}).get('fileDataExists'):
                cm_p = {'uploadFileId': up_id, 'fileMd5': f_md5, 'sliceMd5': s_md5, 'lazyCheck': '1', 'opertype': '3'}
                url, h = self.build_request(cm_p, '/person/commitMultiUploadFile', req_id, session_key, cookie)
                cm_res = self.session.get(url, headers=h, timeout=10).json()
                fid = cm_res.get('file', {}).get('id') or cm_res.get('file', {}).get('userFileId')
                if fid: return fid
        raise Exception(f"个人云秒传失败 (响应信息): {res}")

    def get_direct_url(self, file_id, session_key, cookie, client_ua=""):
        url = "https://cloud.189.cn/api/portal/getFileInfo.action"
        params = {"fileId": str(file_id), "noCache": str(random.random())}
        headers = {
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
            "Cookie": cookie, "Accept": "application/json;charset=UTF-8", "Referer": "https://cloud.189.cn/"
        }
        
        try:
            res = self.session.get(url, params=params, headers=headers, timeout=10).json()
            down_url = res.get('downloadUrl') or res.get('fileDownloadUrl')
            
            if down_url:
                api_url = down_url.replace('&amp;', '&')
                if api_url.startswith('//'): api_url = 'https:' + api_url
                elif not api_url.startswith('http'): api_url = 'https://' + api_url
                
                unwrap_headers = {
                    "User-Agent": client_ua if client_ua else headers["User-Agent"],
                    "Cookie": cookie, "Accept-Encoding": "identity"
                }
                
                unwrap_res = requests.get(api_url, headers=unwrap_headers, allow_redirects=False, timeout=10)
                status_code = unwrap_res.status_code
                
                gateway_url = None
                if status_code in [301, 302, 303, 307, 308]: 
                    loc = unwrap_res.headers.get('Location')
                    if loc and loc.startswith("http://"): loc = loc.replace("http://", "https://", 1)
                    gateway_url = loc
                elif status_code == 200: 
                    if api_url.startswith("http://"): api_url = api_url.replace("http://", "https://", 1)
                    gateway_url = api_url
                else: raise Exception(f"底层破冰失败 (HTTP {status_code})")
                
                if gateway_url:
                    try:
                        probe_res = requests.get(gateway_url, headers=unwrap_headers, allow_redirects=False, timeout=5)
                        if probe_res.status_code in [301, 302, 303, 307, 308]:
                            deep_loc = probe_res.headers.get('Location')
                            if deep_loc: logger.info(f"[无痕探测] 个人云已分配物理节点: {urllib.parse.urlparse(deep_loc).netloc}")
                    except: pass
                
                return gateway_url
                    
            raise Exception(f"提取个人云直链失败(OpenList逻辑): {res}")
        except Exception as e: raise e

personal_client = TianyiPersonalUploader()

# ==========================================
# 🧹 虚空造物清理工
# ==========================================
def delayed_delete_openlist_file(host, token, dir_path, file_name, delay=120):
    time.sleep(delay)
    try:
        headers = {"Authorization": token} if token else {}
        payload = {"dir": dir_path, "names": [file_name]}
        requests.post(f"{host}/api/fs/remove", json=payload, headers=headers, timeout=10)
    except: pass

def cleanup_worker(name, f_md5, fam_id, fold_id, session_key):
    with cache_lock:
        if f_md5 not in upload_cache: return
        expire_time = upload_cache[f_md5]['expire']
        
    expire_str = time.strftime("%H:%M:%S", time.localtime(expire_time))
    logger.info(f"[清理预约] 🕰️ 家庭云：预定于 {expire_str} 执行【{name}】清理任务。")
    
    while True:
        with cache_lock:
            if f_md5 not in upload_cache: return
            expire_time = upload_cache[f_md5]['expire']
            
        now = time.time()
        if now >= expire_time: break
        sleep_time = expire_time - now
        if sleep_time > 0: time.sleep(sleep_time + 1)

    try:
        items = family_client.get_family_items(fam_id, fold_id, session_key)
        real_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == name), None)
        
        if real_fid and family_client.delete_item(fam_id, real_fid, session_key):
            logger.info(f"[移入纸篓] 🗑️ 家庭云：【{name}】已成功移入回收站。")
            time.sleep(2) 
            if family_client.empty_family_recycle(fam_id, session_key):
                logger.info(f"[空间释放] ♻️ 家庭云：回收站已清空，物理空间真实释放！")
    except: pass
        
    with cache_lock:
        if f_md5 in upload_cache: del upload_cache[f_md5]

def personal_cleanup_worker(file_id, session_key, cookie, f_md5):
    with cache_lock:
        if f_md5 not in upload_cache: return
        expire_time = upload_cache[f_md5]['expire']
    
    expire_str = time.strftime("%H:%M:%S", time.localtime(expire_time))
    logger.info(f"[清理预约] 🕰️ 个人云：预定于 {expire_str} 执行文件销毁任务。")
    
    while True:
        with cache_lock:
            if f_md5 not in upload_cache: return
            expire_time = upload_cache[f_md5]['expire']
        now = time.time()
        if now >= expire_time: break
        sleep_time = expire_time - now
        if sleep_time > 0: time.sleep(sleep_time + 1)
        
    try:
        if personal_client.delete_item(file_id, cookie):
            logger.info(f"[移入纸篓] 🗑️ 个人云：文件已成功移入回收站。")
            time.sleep(2)
            if personal_client.empty_personal_recycle(session_key, cookie): 
                logger.info(f"[空间释放] ♻️ 个人云：回收站已清空，物理空间真实释放！")
    except: pass
    
    with cache_lock:
        if f_md5 in upload_cache: del upload_cache[f_md5]

@app_main.route('/smart_proxy/<path:filename>', methods=['GET', 'HEAD'])
@app_302.route('/smart_proxy/<path:filename>', methods=['GET', 'HEAD'])
def smart_proxy(filename=""):
    """智能流代理：真蓝光底层清洗防花屏，假蓝光(MKV)原样透传防报错"""
    target_url = request.args.get('url')
    start_offset = int(request.args.get('offset', 0))
    raw_size = int(request.args.get('size', 0))
    vid_type = request.args.get('type', 'm2ts')

    if not target_url: return "Missing URL", 400

    is_m2ts = (vid_type == 'm2ts')
    if is_m2ts:
        ts_total_size = (raw_size // 192) * 188
        content_type = 'video/mp2t'
    else:
        ts_total_size = raw_size
        content_type = 'video/x-matroska' if vid_type == 'mkv' else 'video/mp4'

    range_header = request.headers.get('Range', None)
    if not range_header:
        req_start, req_end = 0, ts_total_size - 1
        status_code = 200
    else:
        byte_range = range_header.replace('bytes=', '').split('-')
        req_start = int(byte_range[0])
        req_end = int(byte_range[1]) if len(byte_range) > 1 and byte_range[1] else ts_total_size - 1
        status_code = 206

    req_start = max(0, min(req_start, ts_total_size - 1))
    req_end = max(req_start, min(req_end, ts_total_size - 1))
    content_len = req_end - req_start + 1

    if is_m2ts:
        aligned_req_start = (req_start // 188) * 188
        packet_start_idx = aligned_req_start // 188
        packet_end_idx = req_end // 188
        upstream_start = start_offset + packet_start_idx * 192
        upstream_end = start_offset + (packet_end_idx + 1) * 192 - 1
        skip_bytes = req_start - aligned_req_start
    else:
        upstream_start = start_offset + req_start
        upstream_end = start_offset + req_end
        skip_bytes = 0

    upstream_resp = requests.get(
        target_url, 
        headers={'Range': f'bytes={upstream_start}-{upstream_end}', 'User-Agent': request.headers.get('User-Agent', 'Mozilla/5.0')}, 
        stream=True, timeout=25
    )

    def generate():
        bytes_sent = 0
        if is_m2ts:
            leftover = b""
            nonlocal skip_bytes
            for chunk in upstream_resp.iter_content(chunk_size=192 * 4096):
                if not chunk: continue
                chunk = leftover + chunk
                num_packets = len(chunk) // 192
                if num_packets > 0:
                    valid_len = num_packets * 192
                    block_data = chunk[:valid_len]
                    leftover = chunk[valid_len:]
                    
                    # 🚀 C 语言底层极速洗流：剥除 4 字节时间戳
                    out_bytes = b"".join(block_data[i+4:i+192] for i in range(0, valid_len, 192))
                    
                    if skip_bytes > 0:
                        if len(out_bytes) > skip_bytes:
                            out_bytes = out_bytes[skip_bytes:]
                            skip_bytes = 0
                        else:
                            skip_bytes -= len(out_bytes)
                            continue
                    
                    remaining = content_len - bytes_sent
                    if len(out_bytes) > remaining: out_bytes = out_bytes[:remaining]
                    if out_bytes:
                        bytes_sent += len(out_bytes)
                        yield out_bytes
                    if bytes_sent >= content_len: break
                else:
                    leftover = chunk
        else:
            # 🚀 假 ISO（真 MKV/MP4）：绝对原样透传
            for chunk in upstream_resp.iter_content(chunk_size=1024 * 1024):
                if not chunk: continue
                remaining = content_len - bytes_sent
                if len(chunk) > remaining: chunk = chunk[:remaining]
                if chunk:
                    bytes_sent += len(chunk)
                    yield chunk
                if bytes_sent >= content_len: break

    resp_headers = {
        'Content-Type': content_type,
        'Accept-Ranges': 'bytes',
        'Content-Range': f'bytes {req_start}-{req_end}/{ts_total_size}',
        'Content-Length': str(content_len),
    }
    return Response(generate(), status=status_code, headers=resp_headers)

@app_main.route('/play', methods=['GET', 'HEAD'])
def play():
    global iso_meta_cache
    global iso_scan_locks
    global media_resolve_locks

    cas = request.args.get('cas')
    drive_type = request.args.get('drive', '189').strip()
    file_path_param = request.args.get('path', '').strip()
    show_name_from_url = request.args.get('show', '').strip()
    
    client_ua = request.headers.get('User-Agent', '')
    client_ip = request.headers.get('X-Forwarded-For', request.remote_addr)
    if client_ip: client_ip = client_ip.split(',')[0].strip()
    
    cfg = read_config()
    
    if str(cfg.get('force_mode_b', 'false')).lower() == 'true' and cas and drive_type == '189':
        drive_type = '189_native'

    range_header = request.headers.get('Range', '')
    if request.method == 'GET' and range_header and drive_type not in ['189_native', '139_native', '139', 'direct']:
        match = re.match(r'bytes=\d+-(\d+)', range_header)
        if match:
            end_byte = int(match.group(1))
            if 1 < end_byte < 2 * 1024 * 1024:
                logger.warning(f"[防刷拦截] 拒绝极小预读嗅探 (Range: {range_header})，保护云盘！")
                return "Sniff Blocked", 403

    # ==========================================
    # ⚡ 模式 B：189 原生虚空直连解析
    # ==========================================
    if drive_type == '189_native':
        if not cas: return "❌ 缺失 cas 核心代码", 400
        ol_host = cfg.get('openlist_host', 'http://127.0.0.1:5244').rstrip('/')
        ol_token = cfg.get('openlist_token', '')
        headers = {"Authorization": ol_token} if ol_token else {}
        
        try:
            j = parse_cas_content(cas)
            if not j: return "❌ CAS 解析失败", 400
                
            f_md5 = j.get('md5') or j.get('fileMd5') or j.get('fileMD5')
            safe_name = j.get('name') or j.get('fileName') or "unknown.mp4"
            cas_payload = base64.b64encode(json.dumps(j, ensure_ascii=False).encode('utf-8')).decode('utf-8')
            
            with cache_lock:
                if f_md5 not in media_resolve_locks:
                    media_resolve_locks[f_md5] = threading.Lock()
                file_lock = media_resolve_locks[f_md5]

            # 🚀 并发门卫：只放第一个请求进去造物，其余的在门外乖乖等直链！
            with file_lock:
                with cache_lock:
                    if f_md5 in native_link_cache:
                        cached_url, expire_time = native_link_cache[f_md5]
                        if time.time() < expire_time: 
                            now_t = time.time()
                            if now_t - print_throttle_cache.get(f"hit_{f_md5}", 0) > 10:
                                logger.info(f"⚡ [直链缓存] 极速下发 189_native 播放地址")
                                print_throttle_cache[f"hit_{f_md5}"] = now_t
                            return redirect(cached_url)
                        else: del native_link_cache[f_md5] 
            
                logger.info(f"========== 🕵️‍♂️ 189原生直连(模式B) 链路日志 START ==========")
                logger.info(f"[原生直通] 请求原生直通: {safe_name}")

                target_dir = cfg.get('network_cas_path_native', '/177/177-原生直连').rstrip('/')
                temp_file_name = f"temp_play_{f_md5}.cas"
                target_path = f"{target_dir}/{temp_file_name}"
                
                put_headers = headers.copy()
                put_headers.update({"File-Path": urllib.parse.quote(target_path, safe='/'), "Content-Length": str(len(cas_payload.encode('utf-8'))), "Content-Type": "application/octet-stream"})
                
                logger.info(f"[虚空造物] 正在向云端动态写入临时伪装凭证...")
                put_res = requests.put(f"{ol_host}/api/fs/put", data=cas_payload.encode('utf-8'), headers=put_headers, timeout=10)
                if put_res.status_code != 200:
                    logger.error(f"❌ [致命错误] OpenList 写入临时文件失败! 状态码: {put_res.status_code}, 响应: {put_res.text}")
                    return "写入OpenList失败", 500
                
                get_res = requests.post(f"{ol_host}/api/fs/get", json={"path": target_path, "password": ""}, headers=headers, timeout=10).json()
                data_obj = get_res.get('data')
                raw_url = data_obj.get('raw_url') if isinstance(data_obj, dict) else None
                
                if not raw_url:
                    logger.error(f"❌ OpenList 获取直链失败, 响应: {get_res}")
                    return "无 raw_url", 500
                
                logger.info(f"[获取底层] 成功获取底层 raw_url, 启动防风控嗅探...")
                unwrap_headers = {k: v for k, v in request.headers if k.lower() not in ['host', 'accept-encoding', 'authorization']}
                unwrap_headers['Accept-Encoding'] = 'identity'
                if ol_token: unwrap_headers["Authorization"] = ol_token
                if not any(k.lower() == 'range' for k in unwrap_headers): unwrap_headers["Range"] = "bytes=0-"
                
                try:
                    unwrap_res = requests.get(raw_url, headers=unwrap_headers, allow_redirects=False, timeout=15, stream=True)
                    status_code = unwrap_res.status_code
                    headers_dict = dict(unwrap_res.headers)
                    unwrap_res.close()

                    if status_code == 500:
                        logger.warning(f"[嗅探警告] 嗅探遇 500，去 Range 破冰...")
                        range_key = next((k for k in unwrap_headers if k.lower() == 'range'), None)
                        if range_key: del unwrap_headers[range_key]
                        unwrap_res = requests.get(raw_url, headers=unwrap_headers, allow_redirects=False, timeout=15, stream=True)
                        status_code = unwrap_res.status_code
                        headers_dict = dict(unwrap_res.headers)
                        unwrap_res.close()

                    logger.info(f"[嗅探完成] 状态码: {status_code}")

                    if not hasattr(delayed_delete_openlist_file, "last_trigger"): delayed_delete_openlist_file.last_trigger = {}
                    now_time = time.time()
                    if now_time - delayed_delete_openlist_file.last_trigger.get(temp_file_name, 0) > 120:
                        delayed_delete_openlist_file.last_trigger[temp_file_name] = now_time
                        threading.Thread(target=delayed_delete_openlist_file, args=(ol_host, ol_token, target_dir, temp_file_name, 120), daemon=True).start()

                    final_return_url = None
                    if status_code in [301, 302, 303, 307, 308]: final_return_url = headers_dict.get('Location')
                    elif status_code in [200, 206]: final_return_url = raw_url
                        
                    if final_return_url:
                        with cache_lock: native_link_cache[f_md5] = (final_return_url, time.time() + 7200)
                        logger.info(f"[脱壳直链] 拿到 189 官方直链！")
                        logger.info(f"[播放放行] 节点: {urllib.parse.urlparse(final_return_url).netloc} | 地址: {truncate_url(final_return_url)}")
                        logger.info(f"========== 🕵️‍♂️ 189原生直连(模式B) 链路日志 END ==========\n")
                        return redirect(final_return_url)
                    else: 
                        logger.error("❌ 原生直连获取直链异常，未找到有效的重定向地址")
                        return "获取直链异常", 500

                except Exception as unwrap_e: 
                    logger.error(f"❌ 原生直连嗅探异常: {unwrap_e}")
                    return "获取直链异常", 500
        except Exception as e: 
            logger.error(f"❌ 模式B 处理全局异常: {e}")
            return "处理异常", 500
            
    # ==========================================
    # 🟠 139 原生直连解析 (终极防断流 & 完美并发锁版)
    # ==========================================
    if drive_type == '139_native':
        sha256_hash = request.args.get('sha256')
        file_size = request.args.get('size')
        file_name = request.args.get('name')
        
        if not sha256_hash or not file_size: 
            return "❌ 缺失特征码参数", 400

        file_size = int(file_size)
        file_name = urllib.parse.unquote(file_name) if file_name else f"unknown_{sha256_hash[:8]}.mp4"

        # 💡 针对普通视频：MKV / MP4 保持极速 302 原生直连
        is_iso = file_name.lower().endswith('.iso')

        # 🚀 1. 终极并发锁：确保针对同一个特征码的并发请求，只有一个能去执行取链和扫描！
        with cache_lock:
            if sha256_hash not in iso_scan_locks:
                iso_scan_locks[sha256_hash] = threading.Lock()
            file_scan_lock = iso_scan_locks[sha256_hash]

        with file_scan_lock:
            # 🚀 2. 先查缓存（直链是否过期 + ISO偏移是否已算）
            cached_url = None
            if sha256_hash in native_link_cache:
                url, expire_time = native_link_cache[sha256_hash]
                if time.time() < expire_time:
                    cached_url = url
                else:
                    del native_link_cache[sha256_hash]

            # 🚨 3. 活体探测 (防断流 403)
            if cached_url:
                try:
                    test_r = requests.get(cached_url, headers={'Range': 'bytes=0-1', 'User-Agent': 'Mozilla/5.0'}, timeout=3)
                    if test_r.status_code in [401, 403, 404, 400]:
                        logger.warning(f"⚠️ [直链过期] 139 CDN 链接已失效(403)，正在重新提取新链接...")
                        cached_url = None
                        if sha256_hash in native_link_cache: del native_link_cache[sha256_hash]
                        # 既然链接断了，ISO 缓存的偏移可能也会因新链接变动，一并清空
                        if is_iso and sha256_hash in iso_meta_cache: del iso_meta_cache[sha256_hash]
                except:
                    cached_url = None

            # 🚀 4. 执行取链与扫盘 (仅限拿到锁的第一个请求，后面的请求全都是直接拿缓存！)
            if not cached_url:
                logger.info(f"=== 🕵️‍♂️ 139 连链路日志 START ===")
                logger.info(f"[状态请求] 139 原生直通: {file_name}")
                
                try:
                    dl_url = cloud139_native.get_direct_link(sha256_hash, file_size, file_name)
                    if dl_url:
                        native_link_cache[sha256_hash] = (dl_url, time.time() + 7200)
                        cached_url = dl_url
                        logger.info(f"[脱壳直链] 拿到全新 139 官方脱壳直链！")
                        logger.info(f"[播放放行] 节点: {urllib.parse.urlparse(cached_url).netloc} | 地址: {truncate_url(cached_url)}")
                        logger.info(f"=== 🕵️‍♂️ 139链路日志 END ===\n")
                    else:
                        logger.error("❌ 139 直连获取直链异常，链接为空")
                        return "获取直链异常", 500
                except Exception as e:
                    logger.error(f"❌ 139 原生直连异常: {e}")
                    return str(e), 500
            
            # --- 以下是 ISO 专属处理 ---
            if is_iso:
                if sha256_hash in iso_meta_cache:
                    start_byte, v_size, vid_type = iso_meta_cache[sha256_hash]
                    now_t = time.time()
                    if now_t - print_throttle_cache.get(f"iso_cache_{sha256_hash}", 0) > 10:
                        logger.info(f"[ISO缓存] 极速提取内嵌类型: {vid_type} | 偏移: {start_byte}")
                        print_throttle_cache[f"iso_cache_{sha256_hash}"] = now_t
                else:
                    logger.info(f"[防和谐扫] 正在透视 ISO 内部真实文件结构: {file_name}")
                    start_byte, v_size, vid_type = get_iso_main_video_offset(cached_url, file_size)
                    if v_size > 0:
                        iso_meta_cache[sha256_hash] = (start_byte, v_size, vid_type)
                    logger.info(f"[透视成功] 偏移位置: {start_byte} | 视频大小: {format_size(v_size)} | 内部类型: {vid_type}")

        # === 离开锁区域，各自放行 ===
        if not is_iso:
            return redirect(cached_url, code=302)

        if vid_type == "unknown":
            logger.warning(f"[伪装放行] 无法识别 ISO 内嵌格式，尝试强制原生直链放行...")
            return redirect(cached_url, code=302)

        lan_ip = get_lan_server_ip(request)
        host_prefix = f"http://{lan_ip}:5001" if lan_ip else cfg.get('server_host', '').rstrip('/')
        
        ext = ".ts" if vid_type == "m2ts" else f".{vid_type}"
        proxy_route = f"{host_prefix}/smart_proxy/play{ext}?url={urllib.parse.quote(cached_url)}&offset={start_byte}&size={v_size}&type={vid_type}"
        return redirect(proxy_route, code=302)

    # ==========================================
    # 🎬 常规媒体直通 / 🟠 移动云 139 通用跳转 (老模式)
    # ==========================================
    if drive_type in ['139', 'direct']:
        if not file_path_param: return "❌ 请求缺少 path 参数", 400

        with cache_lock:
            if file_path_param not in media_resolve_locks:
                media_resolve_locks[file_path_param] = threading.Lock()
            file_lock = media_resolve_locks[file_path_param]

        # 🚀 并发门卫：让第一个请求去 OpenList 走流程，其余的直接白嫖！
        with file_lock:
            with cache_lock:
                if file_path_param in native_link_cache:
                    cached_url, expire_time = native_link_cache[file_path_param]
                    if time.time() < expire_time: 
                        now_t = time.time()
                        if now_t - print_throttle_cache.get(f"hit_{file_path_param}", 0) > 10:
                            logger.info(f"[直链缓存] 极速下发常规媒体播放地址")
                            print_throttle_cache[f"hit_{file_path_param}"] = now_t
                        return redirect(cached_url)
                    else: del native_link_cache[file_path_param] 

            is_139 = (drive_type == '139')
            ol_host = cfg.get('openlist_host_139' if is_139 else 'openlist_host', 'http://127.0.0.1:5244').rstrip('/')
            ol_token = cfg.get('openlist_token_139' if is_139 else 'openlist_token', '')
            headers = {"Authorization": ol_token} if ol_token else {}
            
            log_tag = "139云盘" if is_139 else "常规真视频"
            logger.info(f"========== 🕵️‍♂️ {log_tag} 链路日志 START ==========")
            logger.info(f"[状态请求] 触发请求: {file_path_param}")

            try:
                get_res = requests.post(f"{ol_host}/api/fs/get", json={"path": file_path_param, "password": ""}, headers=headers, timeout=15)
                if get_res.status_code == 200:
                    data_obj = get_res.json().get('data')
                    raw_url = data_obj.get('raw_url') if isinstance(data_obj, dict) else None
                    if raw_url:
                        logger.info(f"[获取底层] 成功获取底层 raw_url...")
                        
                        unwrap_headers = {
                            "User-Agent": request.headers.get('User-Agent', 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'),
                            "Accept-Encoding": "identity"
                        }
                        if ol_token and ol_host in raw_url:
                            unwrap_headers["Authorization"] = ol_token
                            
                        client_range = request.headers.get('Range')
                        unwrap_headers["Range"] = client_range if client_range else "bytes=0-"
                        
                        try:
                            unwrap_res = requests.get(raw_url, headers=unwrap_headers, allow_redirects=False, timeout=15, stream=True)
                            status_code = unwrap_res.status_code
                            headers_dict = dict(unwrap_res.headers)
                            unwrap_res.close()

                            if status_code == 500:
                                range_key = next((k for k in unwrap_headers if k.lower() == 'range'), None)
                                if range_key: del unwrap_headers[range_key]
                                unwrap_res = requests.get(raw_url, headers=unwrap_headers, allow_redirects=False, timeout=15, stream=True)
                                status_code = unwrap_res.status_code
                                headers_dict = dict(unwrap_res.headers)
                                unwrap_res.close()
                            
                            logger.info(f"[嗅探完成] 状态码: {status_code}")
                            
                            if status_code in [301, 302, 303, 307, 308]:
                                final_cdn_url = headers_dict.get('Location')
                                if final_cdn_url:
                                    expire_sec = int(cfg.get('yun139_link_expire', 7200)) if is_139 else int(cfg.get('link_expire', 120))
                                    with cache_lock: native_link_cache[file_path_param] = (final_cdn_url, time.time() + expire_sec)
                                    logger.info(f"[播放放行] 节点: {urllib.parse.urlparse(final_cdn_url).netloc} | 地址: {truncate_url(final_cdn_url)}")
                                    logger.info(f"========== 🕵️‍♂️ {log_tag} 链路日志 END ==========\n")
                                    return redirect(final_cdn_url)
                                else: 
                                    logger.error(f"❌ {log_tag} 缺失直链跳转地址")
                                    return "缺失直链", 500
                            elif status_code in [200, 206]:
                                expire_sec = int(cfg.get('yun139_link_expire', 7200)) if is_139 else int(cfg.get('link_expire', 120))
                                with cache_lock: native_link_cache[file_path_param] = (raw_url, time.time() + expire_sec)
                                logger.info(f"[播放放行] 节点: {urllib.parse.urlparse(raw_url).netloc} | 地址: {truncate_url(raw_url)}")
                                logger.info(f"========== 🕵️‍♂️ {log_tag} 链路日志 END ==========\n")
                                return redirect(raw_url)
                            else: 
                                logger.error(f"❌ {log_tag} 状态码异常: {status_code}")
                                return "状态码异常", 500
                        except Exception as unwrap_e: 
                            logger.error(f"❌ {log_tag} 获取直链嗅探异常: {unwrap_e}")
                            return "获取直链异常", 500
                    else: 
                        logger.error(f"❌ {log_tag} 接口返回数据中没有 raw_url")
                        return "无 raw_url", 500
                else: 
                    logger.error(f"❌ {log_tag} 接口破冰失败，状态码: {get_res.status_code}")
                    return f"破冰失败", 500
            except Exception as e: 
                logger.error(f"❌ {log_tag} 接口通信全局异常: {e}")
                return "接口通信异常", 500

    # ==========================================
    # 🔵 模式 A：四核统一矩阵攻击队列 (容灾滑点)
    # ==========================================
    if cas:
        safe_name = "未知文件"
        try:
            j = parse_cas_content(cas)
            if not j: raise Exception("CAS 数据解析异常或为空")
            
            f_md5 = str(j.get('md5') or j.get('fileMd5') or j.get('fileMD5')).upper()
            
            if not is_allowed_by_anti_scan(client_ip, f_md5):
                logger.warning(f"[防刷拦截] 拒绝播放器密集并发嗅探！(MD5: {f_md5[:8]})")
                return "Sniff Blocked", 429
                
            s_md5 = str(j.get('slice_md5') or j.get('sliceMd5') or j.get('sliceMD5')).upper()
            raw_size = j.get('size') or j.get('fileSize')
            human_size = format_size(raw_size)
            name = j.get('name') or j.get('fileName')
            base_safe_name = "".join(x for x in name if x not in r'\/:*?"<>|')
            
            if show_name_from_url: 
                clean_show = re.sub(r'\s*\(\d{4}\)', '', show_name_from_url)
                show_identifier = re.sub(r'(?i)\s*(HFR|HQ|IQ|HDR|SDR|DV|4K|1080p|720p)\b', '', clean_show).strip()
            else:
                clean_show = re.split(r'(?i)\.S\d+|\.E\d+|-第\d+集', base_safe_name)[0]
                clean_show = re.sub(r'\s*\(\d{4}\)', '', clean_show)
                show_identifier = re.sub(r'(?i)\s*(HFR|HQ|IQ|HDR|SDR|DV|4K|1080p|720p)\b', '', clean_show).strip()
                
            bind_key = show_identifier
            ext = os.path.splitext(base_safe_name)[1]
            if not ext or len(ext) > 6: ext = ".mp4"

            ep_num = None
            for p in [r'(?i)E(?:P)?\s*0*(\d+)', r'第\s*0*(\d+)\s*[集话期]', r'(?:\[|\()0*(\d+)(?:\]|\))', r'(?i)episode\s*0*(\d+)']:
                m = re.search(p, base_safe_name)
                if m: 
                    ep_num = int(m.group(1)); break
            s_match = re.search(r'(?i)S0*(\d+)', base_safe_name)
            s_num = int(s_match.group(1)) if s_match else 1
            year_match = re.search(r'(?<!\d)(19\d{2}|20\d{2})(?!\d)', base_safe_name)
            year_str = f".{year_match.group(1)}" if year_match else ""

            if show_identifier and ep_num is not None: safe_name = f"{show_identifier}.S{s_num:02d}E{ep_num:02d}{year_str}{ext}"
            else: safe_name = f"{show_identifier}{year_str}{ext}" if show_identifier else base_safe_name

            tags = []
            for t in re.findall(r'(?i)\b(1080p|2160p|4K|DV|HQ|HDR|SDR|IQ|HFR|H\.?26[45]|x\.?26[45])\b', base_safe_name + " " + show_name_from_url):
                t_u = t.upper().replace('.', '')
                if t_u == '1080P': t_u = '1080p'
                elif t_u == '2160P': t_u = '2160p'
                elif t_u == 'X264': t_u = 'H264'
                elif t_u == 'X265': t_u = 'H265'
                if t_u not in tags: tags.append(t_u)
                    
            tag_str = "." + ".".join(tags) if tags else ""
            if tag_str:
                if safe_name.endswith(ext): safe_name = safe_name[:-len(ext)] + tag_str + ext
                else: safe_name = safe_name + tag_str + ext
                
            with cache_lock:
                name_collided = any(v.get('fid') != 'processing' and f_md5 != k and safe_name == (v.get('show_name') + ext if 'show_name' in v else "") for k, v in upload_cache.items())
                if name_collided:
                    size_tag = f".{human_size.replace(' ', '')}"
                    safe_name = safe_name[:-len(ext)] + size_tag + ext if safe_name.endswith(ext) else safe_name + size_tag + ext

            current_time = time.time()
            download_url = None
            is_processing_by_others = False
            
            with cache_lock:
                if f_md5 in upload_cache:
                    cached_data = upload_cache[f_md5]
                    if cached_data.get('download_url'):
                        link_expire_sec = int(cfg.get('link_expire', 120))
                        if current_time - cached_data.get('url_time', 0) < link_expire_sec:
                            download_url = cached_data['download_url']
                        else:
                            logger.info(f"♻️ [票据过期] 距离上次获取已超 {link_expire_sec} 秒，强行作废，提取新鲜直链...")
                            upload_cache[f_md5]['download_url'] = None
                    else:
                        is_processing_by_others = True

            if is_processing_by_others and not download_url:
                for _ in range(50):
                    time.sleep(0.2)
                    with cache_lock:
                        if f_md5 in upload_cache and upload_cache[f_md5].get('download_url'):
                            download_url = upload_cache[f_md5]['download_url']
                            break

            if not download_url and not is_processing_by_others:
                with cache_lock: upload_cache[f_md5] = {'fid': 'processing', 'expire': current_time + cfg.get('delete_delay', 600), 'download_url': None}
                
                valid_accs = []
                for i, acc in enumerate(cfg.get('accounts', [])):
                    fam_valid = bool(acc.get('family_id') and acc.get('family_folder_id'))
                    per_valid = bool(acc.get('personal_folder_id'))
                    can_login = bool((acc.get('username') and acc.get('password')) or i == 3)
                    if (fam_valid or per_valid) and (acc.get('session_key') or can_login):
                        valid_accs.append((i, acc))
                
                if not valid_accs: return "未配置任何有效网盘卡槽", 500

                strategy = cfg.get('cloud_strategy', 'hash')
                strategy_to_idx = {'slot1': 0, 'slot2': 1, 'slot3': 2, 'slot4': 3}
                
                if strategy in strategy_to_idx:
                    target_idx = strategy_to_idx[strategy]
                    vanguard = next((a for a in valid_accs if a[0] == target_idx), None)
                    if vanguard:
                        candidates = [vanguard] + [a for a in valid_accs if a[0] != target_idx]
                    else:
                        logger.warning(f"⚠️ [防死锁] 优先卡槽 {target_idx+1} 未就绪，退守全局哈希滑点调度！")
                        hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(valid_accs)
                        candidates = valid_accs[hash_idx:] + valid_accs[:hash_idx]
                
                elif strategy == 'hash':
                    hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(valid_accs)
                    candidates = valid_accs[hash_idx:] + valid_accs[:hash_idx]
                else: 
                    candidates = valid_accs.copy()
                    random.shuffle(candidates)

                mode_a_cfg = cfg.get('mode_a_channel', 'mix_f2p')
                search_sequence = []
                if mode_a_cfg == 'mix_f2p':
                    search_sequence = [('family', i, a) for i, a in candidates] + [('personal', i, a) for i, a in candidates]
                elif mode_a_cfg == 'mix_p2f':
                    search_sequence = [('personal', i, a) for i, a in candidates] + [('family', i, a) for i, a in candidates]
                elif mode_a_cfg == 'personal':
                    search_sequence = [('personal', i, a) for i, a in candidates]
                else:
                    search_sequence = [('family', i, a) for i, a in candidates]

                final_channel_type = None
                
                for c_type, s_idx, acc in search_sequence:
                    fam_id, fam_fd = acc.get('family_id'), acc.get('family_folder_id')
                    per_fd = acc.get('personal_folder_id')

                    if c_type == 'family' and not (fam_id and fam_fd): continue
                    if c_type == 'personal' and not per_fd: continue

                    target_sk = acc.get('session_key')
                    target_cookie = acc.get('cookie')

                    if not target_sk or not target_cookie:
                        target_sk, target_cookie = refresh_account_logic(s_idx, cfg)
                        if not target_sk: continue

                    log_tag = f"卡槽{s_idx+1}({'个人' if c_type=='personal' else '家庭'})"
                    logger.info(f"[{log_tag}开始调度] 开始处理: {safe_name}")

                    try:
                        if c_type == 'personal':
                            real_fid = personal_client.rapid_upload_personal(per_fd, f_md5, raw_size, s_md5, safe_name, target_sk, target_cookie)
                            download_url = personal_client.get_direct_url(real_fid, target_sk, target_cookie, client_ua)
                            logger.info(f"[{log_tag}秒传就绪] {safe_name} ({human_size})")
                        else:
                            items = family_client.get_family_items(fam_id, fam_fd, target_sk)
                            real_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                            
                            if not real_fid:
                                real_fid = family_client.rapid_upload(fam_id, fam_fd, f_md5, raw_size, s_md5, safe_name, target_sk)
                                logger.info(f"[{log_tag}秒传成功] {safe_name} ({human_size})")
                            else:
                                logger.info(f"[{log_tag}秒传命中] {safe_name} ({human_size})")
                                
                            download_url = family_client.get_download_url(fam_id, real_fid, target_sk, client_ua)
                        
                        if download_url:
                            final_channel_type = c_type
                            with cache_lock:
                                upload_cache[f_md5]['fid'] = real_fid
                                upload_cache[f_md5]['download_url'] = download_url
                                upload_cache[f_md5]['url_time'] = current_time
                                upload_cache[f_md5]['is_personal'] = (c_type == 'personal')
                            if c_type == 'personal': threading.Thread(target=personal_cleanup_worker, args=(real_fid, target_sk, target_cookie, f_md5), daemon=True).start()
                            else: threading.Thread(target=cleanup_worker, args=(safe_name, f_md5, fam_id, fam_fd, target_sk), daemon=True).start()
                            break

                    except Exception as e:
                        err_str = str(e).lower()
                        if "black list" in err_str or "security check" in err_str:
                            logger.error(f"[{log_tag}版权拦截] 黑名单限制，强制阻断！")
                            break 
                            
                        elif any(k in err_str for k in ["auth_fail", "session", "111", "notlogin"]):
                            logger.warning(f"[{log_tag}凭证失效] 尝试执行自愈...")

                            target_sk, target_cookie = refresh_account_logic(s_idx, cfg)
                            if target_sk:
                                try: 
                                    if c_type == 'personal':
                                        real_fid = personal_client.rapid_upload_personal(per_fd, f_md5, raw_size, s_md5, safe_name, target_sk, target_cookie)
                                        download_url = personal_client.get_direct_url(real_fid, target_sk, target_cookie, client_ua)
                                        logger.info(f"[{log_tag}秒传就绪] {safe_name} ({human_size})")
                                    else:
                                        items = family_client.get_family_items(fam_id, fam_fd, target_sk)
                                        real_fid = next((i['fileId'] for i in items if f_md5 in i['fileName'] or i['fileName'] == safe_name), None)
                                        
                                        if not real_fid:
                                            real_fid = family_client.rapid_upload(fam_id, fam_fd, f_md5, raw_size, s_md5, safe_name, target_sk)
                                            logger.info(f"[{log_tag}秒传成功] {safe_name} ({human_size})")
                                        else:
                                            logger.info(f"[{log_tag}秒传命中] {safe_name} ({human_size})")
                                            
                                        download_url = family_client.get_download_url(fam_id, real_fid, target_sk, client_ua)
                                        
                                    if download_url:
                                        final_channel_type = c_type
                                        with cache_lock:
                                            upload_cache[f_md5]['fid'] = real_fid
                                            upload_cache[f_md5]['download_url'] = download_url
                                            upload_cache[f_md5]['url_time'] = current_time
                                            upload_cache[f_md5]['is_personal'] = (c_type == 'personal')
                                        if c_type == 'personal': threading.Thread(target=personal_cleanup_worker, args=(real_fid, target_sk, target_cookie, f_md5), daemon=True).start()
                                        else: threading.Thread(target=cleanup_worker, args=(safe_name, f_md5, fam_id, fam_fd, target_sk), daemon=True).start()
                                        break
                                except Exception as e2:
                                    logger.error(f"[{log_tag}自愈后重试依然失败]: {e2}。切换至下一通道...")
                                    continue
                        else:
                            logger.warning(f"[{log_tag}遭遇拦截] 报错: {e}。🚨 触发队列滑点：无缝切换至下一个卡槽通道...")
                            continue

                if not download_url:
                    with cache_lock:
                        if f_md5 in upload_cache: del upload_cache[f_md5]
                    raise Exception("攻击队列执行完毕，所有卡槽节点均无法承载该文件！")
            
            if download_url:                   
                parsed = urllib.parse.urlparse(download_url)
                is_p = False
                with cache_lock:
                    if f_md5 in upload_cache: is_p = upload_cache[f_md5].get('is_personal', False)
                
                now_t = time.time()
                if now_t - print_throttle_cache.get(f"play_{f_md5}", 0) > 15:
                    logger.info(f"[播放放行] 节点({'个人云' if is_p else '家庭云'}): {parsed.netloc} | 地址: {truncate_url(download_url)}")
                    print_throttle_cache[f"play_{f_md5}"] = now_t
                    
                return redirect(download_url, code=302)

        except Exception as e:
            logger.error(f"❌ 模式A 处理异常: {e}")
            with cache_lock:
                if 'f_md5' in locals() and f_md5 in upload_cache and upload_cache[f_md5].get('fid') == 'processing':
                    del upload_cache[f_md5]
            return f"错误: {e}", 500

def warm_up_parent(target_path, headers, api_host):
    if not target_path: return
    cfg = read_config()
    base_path = cfg.get('network_cas_path', '').rstrip('/')
    if target_path.startswith(base_path):
        rel_path = target_path[len(base_path):].strip('/')
        parts = rel_path.split('/')
        current_path = base_path
        for part in parts[:-1]:
            current_path = f"{current_path}/{part}"
            try: requests.post(f"{api_host}/api/fs/list", json={"path": current_path, "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=5)
            except: pass

def scan_openlist_recursive(current_path, headers, result_list, api_host, file_type='cas'):
    try:
        res = requests.post(f"{api_host}/api/fs/list", json={"path": current_path, "page": 1, "per_page": 1000, "refresh": True}, headers=headers, timeout=15).json()
        if res.get("code") != 200: return
        for f in res.get("data", {}).get("content", []):
            if f.get("is_dir"): scan_openlist_recursive(f"{current_path}/{f['name']}", headers, result_list, api_host, file_type)
            else:
                ext = f['name'].lower().split('.')[-1] if '.' in f['name'] else ''
                if file_type in ['cas', 'cas_native', 'cas_native_139'] and ext == 'cas': result_list.append(f"{current_path}/{f['name']}")
                elif file_type == 'media' and ext in ['mp4', 'mkv', 'ts', 'avi', 'mov', 'webm', 'flv', 'iso']: result_list.append(f"{current_path}/{f['name']}")
    except: pass

def generate_strm_from_openlist_to_local(target_path=None, drive_type='189', file_type='cas'):
    cfg = read_config()
    
    if drive_type == '139':
        scan_root = target_path if target_path else cfg.get('network_cas_path_139', '')
        base_cas_path = cfg.get('network_cas_path_139', '')
        local_strm_dir = cfg.get('local_strm_dir_139', '')
        local_strm_dir_native = cfg.get('local_strm_dir_139_native', '')
        api_host = cfg.get('openlist_host_139', 'http://127.0.0.1:5255').rstrip('/')
        api_token = cfg.get('openlist_token_139', '')
    else:
        api_host = cfg.get('openlist_host', 'http://127.0.0.1:5244').rstrip('/')
        api_token = cfg.get('openlist_token', '')
        if file_type == 'cas':
            scan_root = target_path if target_path else cfg.get('network_cas_path', '')
            base_cas_path = cfg.get('network_cas_path', '')
            local_strm_dir = cfg.get('local_strm_dir', '')
        elif file_type == 'cas_native':
            scan_root = target_path if target_path else cfg.get('network_cas_path', '')
            base_cas_path = cfg.get('network_cas_path', '')
            local_strm_dir = cfg.get('local_strm_dir_native', '')
        elif file_type == 'media':
            scan_root = target_path if target_path else cfg.get('network_media_path', '')
            base_cas_path = cfg.get('network_media_path', '')
            local_strm_dir = cfg.get('local_strm_dir_media', '')
        
    os.makedirs(local_strm_dir if local_strm_dir else '/tmp', exist_ok=True)
    headers = {"Authorization": api_token} if api_token else {}
    
    if target_path and drive_type != '139': warm_up_parent(target_path, headers, api_host)
    
    logger.info(f"[扫描启动] OpenList [{drive_type} / {file_type.upper()}] -> 区域: {scan_root}")
    cas_files = []
    search_type = 'cas' if file_type == 'both' else file_type
    scan_openlist_recursive(scan_root, headers, cas_files, api_host, search_type)
    if not cas_files: return logger.info(f"⚠️ 未找到目标文件")
        
    count = 0
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
            
            show_name = re.sub(r'\s*\(\d{4}\)', '', show_name)
            show_name = re.sub(r'(?i)\s*(HQ|IQ|HDR|SDR|DV|4K|1080p|720p)\b', '', show_name)
            show_name = re.sub(r'[《》]', '', show_name).strip()
            
            base_name = os.path.basename(rel_path).rsplit('.', 1)[0]
            
            # 🎬 常规媒体 (Media)
            if file_type == 'media':
                target_local_dir = os.path.join(local_strm_dir, rel_dir)
                os.makedirs(target_local_dir, exist_ok=True)
                strm_path = os.path.join(target_local_dir, f"{base_name}.strm")
                if not os.path.exists(strm_path):
                    with open(strm_path, "w", encoding="utf-8") as f: 
                        f.write(f"{cfg['server_host']}/play?drive=direct&path={urllib.parse.quote(full_path)}&show={urllib.parse.quote(show_name)}")
                    count += 1
                continue

            # ⬇️ 云端内容抓取
            get_res = req_session.post(f"{api_host}/api/fs/get", json={"path": full_path}, headers=headers, timeout=10).json()
            raw_url = get_res.get("data", {}).get("raw_url")
            if not raw_url: continue
            cas_content = req_session.get(raw_url, timeout=10).text.strip()
            # 🚨 终极防污染：如果下载下来的内容是 HTML 网页(说明网盘限流或死机)，绝不生成 STRM！
            if cas_content.startswith('<'):
                logger.warning(f"⚠️ [防污染拦截] OpenList 返回了网页报错，已跳过生成: {base_name}")
                continue
            
            # ================== 139 体系 ==================
            if drive_type == '139':
                # 🎯 【139 模式 B】: 老模式 (OpenList) -> -139B.strm
                if file_type in ['cas', 'both'] and local_strm_dir:
                    target_dir_b = os.path.join(local_strm_dir, rel_dir)
                    os.makedirs(target_dir_b, exist_ok=True)
                    strm_path_b = os.path.join(target_dir_b, f"{base_name}-139B.strm")
                    if not os.path.exists(strm_path_b):
                        fake_net_path = f"{net_cas_path_139}/{rel_path}".replace('\\', '/')
                        with open(strm_path_b, "w", encoding="utf-8") as f_old:
                            f_old.write(f"{cfg['server_host']}/play?drive=139&path={urllib.parse.quote(fake_net_path)}&show={urllib.parse.quote(show_name)}")
                        count += 1
                        
                # 🎯 【139 模式 A】: 官方原生直连 -> -139A.strm
                if file_type in ['cas_native', 'both'] and local_strm_dir_native:
                    try:
                        c_dict = parse_cas_content(cas_content)
                        c_sha = c_dict.get('sha256') or c_dict.get('hash') or c_dict.get('fileMd5') or c_dict.get('md5') or ''
                        c_size = c_dict.get('size') or c_dict.get('fileSize') or ''
                        c_name = c_dict.get('name') or c_dict.get('fileName') or base_name
                        if c_sha and str(c_size):
                            target_dir_a = os.path.join(local_strm_dir_native, rel_dir)
                            os.makedirs(target_dir_a, exist_ok=True)
                            strm_path_a = os.path.join(target_dir_a, f"{base_name}-139A.strm")
                            if not os.path.exists(strm_path_a):
                                strm_data_a = f"{cfg['server_host']}/play?drive=139_native&sha256={c_sha}&size={c_size}&name={urllib.parse.quote(c_name)}&show={urllib.parse.quote(show_name)}"
                                with open(strm_path_a, "w", encoding="utf-8") as f: f.write(strm_data_a)
                                count += 1
                    except Exception as e: 
                        logger.error(f"139解析异常: {e}")
                        
            # ================== 189 体系 ==================
            else:
                # 🎯 【189 模式 A】: 官方秒传 -> .strm (保持兼容无后缀)
                if file_type in ['cas', 'both'] and local_strm_dir:
                    target_dir_a = os.path.join(local_strm_dir, rel_dir)
                    os.makedirs(target_dir_a, exist_ok=True)
                    strm_path_a = os.path.join(target_dir_a, f"{base_name}.strm")
                    if not os.path.exists(strm_path_a):
                        strm_data_a = f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
                        with open(strm_path_a, "w", encoding="utf-8") as f: f.write(strm_data_a)
                        count += 1
                        
                # 🎯 【189 模式 B】: OpenList中转虚空直通 -> -189B.strm
                if file_type in ['cas_native', 'both'] and local_strm_dir_native:
                    target_dir_b = os.path.join(local_strm_dir_native, rel_dir)
                    os.makedirs(target_dir_b, exist_ok=True)
                    strm_path_b = os.path.join(target_dir_b, f"{base_name}-189B.strm")
                    if not os.path.exists(strm_path_b):
                        strm_data_b = f"{cfg['server_host']}/play?drive=189_native&cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
                        with open(strm_path_b, "w", encoding="utf-8") as f: f.write(strm_data_b)
                        count += 1
                        
        except Exception as e: time.sleep(2)

    if count > 0: logger.info(f"[同步完毕] 成功归档 {count} 个 STRM 文件")

@app_main.route('/api/sync')
def trigger_sync():
    target_path = request.args.get('path') 
    drive_type = request.args.get('drive', '189')
    file_type = request.args.get('type', 'cas')
    threading.Thread(target=generate_strm_from_openlist_to_local, args=(target_path, drive_type, file_type), daemon=True).start()
    return "✅ 同步指令下发成功", 200

def local_cas_sync_worker():
    cfg = read_config()
    
    # === 1. 扫描 189 的本地源目录 ===
    source_dir_189 = cfg.get('local_cas_source_dir', '')
    base_dir_a = cfg.get('local_strm_dir', '')
    base_dir_b = cfg.get('local_strm_dir_native', '')
    count_189 = 0
    
    if source_dir_189 and os.path.exists(source_dir_189):
        logger.info(f"[本地扫描] 启动 189 CAS 扫描 -> 目录: {source_dir_189}")
        for root, dirs, files in os.walk(source_dir_189):
            for file in files:
                if file.endswith('.cas'):
                    full_path = os.path.join(root, file)
                    rel_path = full_path[len(source_dir_189):].lstrip('/\\')
                    rel_dir = os.path.dirname(rel_path)
                    
                    dir_parts = [p for p in rel_dir.split('/') if p]
                    show_name = "未知剧集"
                    for part in reversed(dir_parts):
                        if not re.match(r'(?i)^(season\s*\d+|specials|电视剧|电影|动漫|纪录片|综艺)$', part):
                            show_name = part; break
                    if show_name == "未知剧集" and dir_parts: show_name = dir_parts[-1]
                    
                    show_name = re.sub(r'\s*\(\d{4}\)', '', show_name)
                    show_name = re.sub(r'(?i)\s*(HQ|IQ|HDR|SDR|DV|4K|1080p|720p)\b', '', show_name)
                    show_name = re.sub(r'[《》]', '', show_name).strip()
                    base_name = os.path.splitext(file)[0]
                    
                    try:
                        with open(full_path, 'r', encoding='utf-8') as f:
                            cas_content = f.read().strip()
                            
                        # 🎯 189 模式 A (官方秒传) -> .strm
                        if base_dir_a:
                            target_a = os.path.join(base_dir_a, rel_dir)
                            os.makedirs(target_a, exist_ok=True)
                            strm_a = os.path.join(target_a, f"{base_name}.strm")
                            if not os.path.exists(strm_a):
                                with open(strm_a, "w", encoding="utf-8") as fa:
                                    fa.write(f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}")
                                count_189 += 1
                                
                        # 🎯 189 模式 B (OpenList直通) -> -189B.strm
                        if base_dir_b:
                            target_b = os.path.join(base_dir_b, rel_dir)
                            os.makedirs(target_b, exist_ok=True)
                            strm_b = os.path.join(target_b, f"{base_name}-189B.strm")
                            if not os.path.exists(strm_b):
                                with open(strm_b, "w", encoding="utf-8") as fb:
                                    fb.write(f"{cfg['server_host']}/play?drive=189_native&cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}")
                                count_189 += 1
                    except: pass

    # === 2. 扫描 139 的本地源目录 ===
    source_dir_139 = cfg.get('local_cas_source_dir_139', '')
    base_dir_139_old = cfg.get('local_strm_dir_139', '')
    base_dir_139_native = cfg.get('local_strm_dir_139_native', '')
    net_cas_path_139 = cfg.get('network_cas_path_139', '').rstrip('/')
    count_139 = 0
    
    if source_dir_139 and os.path.exists(source_dir_139):
        logger.info(f"[本地扫描] 启动 139 CAS 双轨扫描 -> 目录: {source_dir_139}")
        for root, dirs, files in os.walk(source_dir_139):
            for file in files:
                if file.endswith('.cas'):
                    full_path = os.path.join(root, file)
                    rel_path = full_path[len(source_dir_139):].lstrip('/\\')
                    rel_dir = os.path.dirname(rel_path)
                    
                    dir_parts = [p for p in rel_dir.split('/') if p]
                    show_name = "未知剧集"
                    for part in reversed(dir_parts):
                        if not re.match(r'(?i)^(season\s*\d+|specials|电视剧|电影|动漫|纪录片|综艺)$', part):
                            show_name = part; break
                    if show_name == "未知剧集" and dir_parts: show_name = dir_parts[-1]
                    
                    show_name = re.sub(r'\s*\(\d{4}\)', '', show_name)
                    show_name = re.sub(r'(?i)\s*(HQ|IQ|HDR|SDR|DV|4K|1080p|720p)\b', '', show_name)
                    show_name = re.sub(r'[《》]', '', show_name).strip()
                    base_name = os.path.splitext(file)[0]
                    
                    try:
                        with open(full_path, 'r', encoding='utf-8') as f:
                            cas_content = f.read().strip()
                        
                        # 🎯 139 模式 B (老模式中转) -> -139B.strm
                        if base_dir_139_old:
                            target_139_old = os.path.join(base_dir_139_old, rel_dir)
                            os.makedirs(target_139_old, exist_ok=True)
                            strm_139_old = os.path.join(target_139_old, f"{base_name}-139B.strm")
                            if not os.path.exists(strm_139_old):
                                fake_net_path = f"{net_cas_path_139}/{rel_path}".replace('\\', '/')
                                with open(strm_139_old, "w", encoding="utf-8") as f_old:
                                    f_old.write(f"{cfg['server_host']}/play?drive=139&path={urllib.parse.quote(fake_net_path)}&show={urllib.parse.quote(show_name)}")
                                count_139 += 1
                                
                        # 🎯 139 模式 A (原生直连) -> -139A.strm
                        if base_dir_139_native:
                            target_139_native = os.path.join(base_dir_139_native, rel_dir)
                            os.makedirs(target_139_native, exist_ok=True)
                            strm_139_native = os.path.join(target_139_native, f"{base_name}-139A.strm")
                            if not os.path.exists(strm_139_native):
                                d = parse_cas_content(cas_content)
                                c_sha = d.get('sha256') or d.get('hash') or d.get('fileMd5') or d.get('md5') or ''
                                c_size = d.get('size') or d.get('fileSize') or ''
                                c_name = d.get('name') or d.get('fileName') or base_name
                                if c_sha and str(c_size):
                                    with open(strm_139_native, "w", encoding="utf-8") as f_native:
                                        f_native.write(f"{cfg['server_host']}/play?drive=139_native&sha256={c_sha}&size={c_size}&name={urllib.parse.quote(c_name)}&show={urllib.parse.quote(show_name)}")
                                    count_139 += 1
                    except: pass
                    
    if count_189 > 0 or count_139 > 0: 
        logger.info(f"[扫描完毕] 生成 189 STRM: {count_189} 个, 139 STRM: {count_139} 个")

@app_main.route('/api/sync_local')
def api_sync_local():
    threading.Thread(target=local_cas_sync_worker, daemon=True).start()
    return "✅ 本地扫描指令下发成功", 200

@app_main.route('/api/make_strm', methods=['POST'])
def api_make_strm():
    try:
        data = request.json
        source_cas_path = data.get('source_cas_path')    
        target_local_dir = data.get('target_local_dir') 
        strm_name = data.get('strm_name')                
        show_name = data.get('show_name')                
        mode = data.get('mode', '').lower() 

        if not all([source_cas_path, strm_name, show_name, target_local_dir]): 
            return jsonify({"code": 400, "msg": "指令参数不全"}), 400
        if not os.path.exists(source_cas_path): 
            return jsonify({"code": 404, "msg": f"找不到源文件: {source_cas_path}"}), 404

        cfg = read_config()
        with open(source_cas_path, 'r', encoding='utf-8') as f: cas_content = f.read().strip()
        
        if cas_content.startswith('<'):
            return jsonify({"code": 500, "msg": "源 cas 文件已被污染成 HTML 网页，拒绝生成"}), 500
        
        base_name = os.path.splitext(strm_name)[0]
        
        os.makedirs(target_local_dir, exist_ok=True)
        cas_data = parse_cas_content(cas_content)

        # =======================================================
        # 👑 绝对统一：生成文件后缀标准
        # =======================================================

        # 🎯 【139 模式 A】: 官方原生直连 (139_native)
        if mode in ['mode_a_139', 'cas_139_native', '139_native', 'both_139']:
            c_sha = cas_data.get('sha256') or cas_data.get('hash') or cas_data.get('fileMd5') or cas_data.get('md5') or ''
            c_size = cas_data.get('size') or cas_data.get('fileSize') or ''
            c_name = cas_data.get('name') or cas_data.get('fileName') or base_name
            
            if c_sha and str(c_size):
                out_name = f"{base_name}-139A.strm" if 'both' in mode else strm_name
                strm_path = os.path.join(target_local_dir, out_name)
                strm_data = f"{cfg['server_host']}/play?drive=139_native&sha256={c_sha}&size={c_size}&name={urllib.parse.quote(c_name)}&show={urllib.parse.quote(show_name)}"
                with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)
            else:
                if 'both' not in mode: return jsonify({"code": 500, "msg": "139模式A生成失败: 提取不到 sha256 或 size"}), 500

        # 🎯 【139 模式 B】: OpenList老模式 (139)
        if mode in ['mode_b_139', 'cas_139', '139_old', 'both_139']:
            cloud_path = data.get('cloud_cas_path', '')
            if cloud_path:
                out_name = f"{base_name}-139B.strm" if 'both' in mode else strm_name
                strm_path = os.path.join(target_local_dir, out_name)
                strm_data = f"{cfg['server_host']}/play?drive=139&path={urllib.parse.quote(cloud_path)}&show={urllib.parse.quote(show_name)}"
                with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)
            else:
                if 'both' not in mode: return jsonify({"code": 400, "msg": "139模式B缺少 cloud_cas_path 参数"}), 400

        # 🎯 【189 模式 A】: 官方秒传直通 (189)
        if mode in ['mode_a_189', 'mode_a', 'cas', 'both', 'both_189']:
            out_name = f"{base_name}.strm" if 'both' in mode else strm_name
            strm_path = os.path.join(target_local_dir, out_name)
            strm_data = f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)

        # 🎯 【189 模式 B】: OpenList虚空直通 (189_native)
        if mode in ['mode_b_189', 'mode_b', 'cas_native', 'both', 'both_189']:
            out_name = f"{base_name}-189B.strm" if 'both' in mode else strm_name
            strm_path = os.path.join(target_local_dir, out_name)
            strm_data = f"{cfg['server_host']}/play?drive=189_native&cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)

        logger.info(f"[接口调用] 成功生成 STRM: {strm_name} (策略: {mode})")
        return jsonify({"code": 200, "msg": "success"}), 200
        
    except Exception as e: 
        return jsonify({"code": 500, "msg": str(e)}), 500

emby_session = requests.Session()

@functools.lru_cache(maxsize=256)
def get_emby_item_path(item_id, media_source_id=None):
    clean_media_id = str(media_source_id).replace('mediasource_', '').strip() if media_source_id else None
    query_ids = f"{clean_media_id},{item_id}" if clean_media_id else item_id
    
    def _extract_path(api_key, desc):
        url = f"{EMBY_HOST}/emby/Items?Ids={query_ids}&Fields=Path,MediaSources&api_key={api_key}"
        try:
            res = emby_session.get(url, timeout=3)
            if res.status_code == 200:
                items = res.json().get('Items', [])
                if clean_media_id:
                    for item in items:
                        if str(item.get('Id')) == clean_media_id: return item.get('Path', ''), f"{desc}-精准匹配"
                        for source in item.get('MediaSources', []):
                            if str(source.get('Id')) == clean_media_id: return source.get('Path', item.get('Path', '')), f"{desc}-精准匹配"
                if items:
                    sources = items[0].get('MediaSources', [])
                    if sources: return sources[0].get('Path', ''), f"{desc}-默认版本"
                    return items[0].get('Path', ''), f"{desc}-兜底主路径"
        except: pass
        return None, None

    res_path, res_desc = _extract_path(API_KEY_LINUX, "Linux(主力)")
    if res_path: return res_path, res_desc
    return _extract_path(API_KEY_APP, "APP(备用)")

@app_302.route('/', defaults={'path': ''}, methods=['GET', 'HEAD', 'POST', 'OPTIONS'])
@app_302.route('/<path:full_path>', methods=['GET', 'HEAD', 'POST', 'OPTIONS'])
def catch_all_for_emby(full_path):
    # 🚨 终极死锁切断：交给新的 smart_proxy，绝不串台死循环！
    if request.path.startswith('/smart_proxy/'):
        filename = request.path.split('/smart_proxy/')[-1]
        return smart_proxy(filename)
    if request.path.startswith('/play'):
        return play()

    match = re.search(r'/(?:videos|Items)/(\d+)/(?:stream|original|Download)', request.path, re.IGNORECASE)
    if not match: return redirect(f"{EMBY_HOST}{request.full_path}", code=302)
    
    item_id = match.group(1)
    media_source_id = request.args.get('MediaSourceId') or request.args.get('mediaSourceId') or request.args.get('mediasourceid')
    
    try:
        file_path, version = get_emby_item_path(item_id, media_source_id)
        if not file_path: return redirect(f"{EMBY_HOST}{request.full_path}", code=302)
        
        strm_url = None
        file_name = file_path.split('/')[-1] if file_path else "未知文件"

        if file_path.startswith('http://') or file_path.startswith('https://') or file_path.startswith('play?'):
            strm_url = file_path
        elif file_path.lower().endswith('.strm') and os.path.exists(file_path):
            with open(file_path, 'r', encoding='utf-8') as f: strm_url = f.read().strip()

        if strm_url:
            cfg = read_config()
            lan_ip = get_lan_server_ip(request)
            
            parsed_url = urllib.parse.urlparse(strm_url)
            query_params = urllib.parse.parse_qs(parsed_url.query)
            
            is_direct = False
            if 'direct' in file_path.lower() or 'modeb' in file_path.lower(): is_direct = True
            if str(cfg.get('force_mode_b', 'false')).lower() == 'true': is_direct = True
                
            if is_direct and 'cas' in query_params and 'drive' not in query_params:
                query_params['drive'] = ['189_native']

            new_query = urllib.parse.urlencode(query_params, doseq=True)
            path_str = parsed_url.path if parsed_url.path.startswith('/') else '/' + parsed_url.path
            
            if not parsed_url.netloc:
                host_base = f"http://{lan_ip}:5000" if lan_ip else cfg['server_host']
                strm_url = f"{host_base}{path_str}?{new_query}"
            else:
                if lan_ip: strm_url = f"http://{lan_ip}:5000{path_str}?{new_query}"
                else: strm_url = f"{parsed_url.scheme}://{parsed_url.netloc}{path_str}?{new_query}"
            
            now_t = time.time()
            if now_t - print_throttle_cache.get(f"hijack_{item_id}", 0) > 15:
                if lan_ip: logger.info(f"[网络嗅探] 识别为内网播放，下发路由重定向")
                logger.info(f"[劫持放行] {version} -> {file_name}")
                print_throttle_cache[f"hijack_{item_id}"] = now_t
                
            return redirect(strm_url, code=302)
        else:
            return redirect(f"{EMBY_HOST}{request.full_path}", code=302)
    except:
        return redirect(f"{EMBY_HOST}{request.full_path}", code=302)

def run_main(): app_main.run(host='0.0.0.0', port=5000, use_reloader=False)
def run_302(): app_302.run(host='0.0.0.0', port=5001, use_reloader=False)

def keep_alive_worker():
    time.sleep(60) 
    while True:
        try:
            cfg = read_config()
            has_checked = False
            
            for i, acc in enumerate(cfg.get('accounts', [])):
                fam_id, fold_id = acc.get('family_id'), acc.get('family_folder_id')
                per_id = acc.get('personal_folder_id')
                sk, cookie = acc.get('session_key'), acc.get('cookie')
                
                if not (acc.get('username') and acc.get('password')) and i != 3: continue
                
                if not has_checked:
                    logger.info("📡 [巡逻雷达] 正在对矩阵卡槽进行后台健康体检...")
                    has_checked = True
                
                is_alive = True
                if (fam_id and fold_id and not sk) or (per_id and not cookie) or (i == 3 and not sk):
                    is_alive = False
                
                if is_alive and fam_id and fold_id and sk:
                    try: family_client.get_family_items(fam_id, fold_id, sk)
                    except Exception as e:
                        if any(k in str(e).lower() for k in ["auth_fail", "111", "session"]): is_alive = False 
                
                if is_alive and per_id and sk and cookie:
                    try: personal_client.get_personal_items(per_id, cookie)
                    except Exception as e:
                        if any(k in str(e).lower() for k in ["auth_fail", "111", "session", "notlogin"]): is_alive = False
                
                if not is_alive: 
                    logger.warning(f"⚠️ [巡逻预警] 发现卡槽 {i+1} 凭证异常或为空，立即激活自愈程序！")
                    refresh_account_logic(i, cfg)
                
                time.sleep(random.randint(5, 10))
            
            if has_checked:
                logger.info("✅ [巡逻完毕] 本轮健康体检结束，所有卡槽运转正常。")
                
        except Exception as e: 
            logger.error(f"❌ [巡逻异常]: {e}")
            
        time.sleep(3600) 

if __name__ == '__main__':
    logger.info("[管家启动] 双头蛇引擎 V10 四核矩阵滑点版 启动完毕！")
    threading.Thread(target=keep_alive_worker, daemon=True).start()
    t1 = threading.Thread(target=run_main)
    t2 = threading.Thread(target=run_302)
    t1.start()
    t2.start()
    t1.join()
    t2.join()
```