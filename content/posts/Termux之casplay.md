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
import base64, json, time, random, hashlib, hmac, urllib.parse, threading, uuid, os, requests, logging, subprocess, math
import socket, re, functools
import urllib3
urllib3.util.connection.HAS_IPV6 = False
from collections import deque
from flask import Flask, request, redirect, render_template_string, jsonify
from Crypto.Cipher import AES, PKCS1_v1_5
from Crypto.PublicKey import RSA
from Crypto.Util.Padding import pad

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
cache_lock = threading.Lock()
print_throttle_cache = {}  # 🌟 新增：日志视觉防刷屏字典

BASE_DIR = os.path.dirname(os.path.abspath(__file__))
DB_DIR = os.path.join(BASE_DIR, "db")
os.makedirs(DB_DIR, exist_ok=True)

def get_db_path(): return os.path.join(DB_DIR, "config.json")

# ==========================================
# 🔔 统一看板日志与推送系统
# ==========================================
log_buffer = deque(maxlen=150) 
class MemoryHandler(logging.Handler):
    def emit(self, record):
        msg = self.format(record)
        log_buffer.append({'time': time.strftime("%H:%M:%S"), 'level': record.levelname, 'msg': f"● {msg}"})

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
        log_buffer.append({'time': time.strftime("%H:%M:%S"), 'level': data.get('level', 'INFO'), 'msg': f"● [远程] {data.get('msg', '')}"})
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
    
    # 核心恢复：检查文件最后修改时间
    current_mtime = os.path.getmtime(cookie_file)
    
    # 如果文件没被别的脚本修改过，直接返回内存里的 Key，绝不去骚扰天翼云！
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
            # 存入记忆缓存
            cookie_cache["sk"] = sk
            cookie_cache["cookie_str"] = cookie_str
            cookie_cache["mtime"] = current_mtime
            return sk, cookie_str
        else:
            # 只有当天翼云明确拒绝了这个新文件，且提取不到 Key 时，才销毁它逼迫外部重新生成
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

        # 🌟 修复: 卡槽 4 凭证自动提取与 TG 推送
        if slot_idx == 3 and not pwd:
            logger.info("[凭证获取] 卡槽 4 尝试从外部 cookies.json 读取最高权限...")
            sk, cookie = get_auto189_credentials()
            if sk and cookie:
                acc['session_key'] = sk
                acc['cookie'] = cookie
                save_config(cfg)
                logger.info("[自愈成功] 卡槽 4 从外部读取 Cookie 满血复活！")
                send_push("✅ 大号自愈成功", "卡槽 4 成功从外部读取到最新 Cookie 凭证！")
                return sk, cookie
            else:
                logger.error("[凭证失败] 卡槽 4 外部 Cookie 提取失败或已过期！")
                send_push("❌ 大号自愈失败", "卡槽 4 外部 Cookie 提取失败或已过期！")
                return None, None

        # 🌟 修复: 统一 1~4 号密码重登时的 TG 推送全覆盖
        if user and pwd:
            logger.info(f"[自愈启动] 统一卡槽 {slot_idx+1} 开始账号密码重登...")
            send_push("🔄 双轨自愈启动", f"检测到卡槽 {slot_idx+1} 凭证失效，正在执行自动重登...")
            try:
                auth = Cloud189AuthEngine()
                sk, cookie = auth.do_login_and_get_key(user, pwd, f"卡槽{slot_idx+1}")
                if sk and cookie:
                    acc['session_key'] = sk
                    acc['cookie'] = cookie
                    save_config(cfg)
                    logger.info(f"[自愈成功] 统一卡槽 {slot_idx+1} 满血复活！")
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
    cfg['tg_proxy'] = request.form.get('tg_proxy', '').strip()  # 新增：保存代理配置
    
    for k in cfg.keys():
        if k not in ['accounts', 'cloud_strategy', 'mode_a_channel', 'force_mode_b', 'tg_proxy'] and k in request.form:
            val = request.form.get(k, '').strip()
            if k in ['delete_delay', 'shield_delay']: cfg[k] = int(val) if val else (600 if k == 'delete_delay' else 2700)
            else: cfg[k] = val
            
    save_config(cfg)
    family_client.rsa_keys.clear() 
    personal_client.rsa_keys.clear()
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
        <div class="mac-window">
            <div class="mac-header">
                <div class="mac-btn btn-close"></div><div class="mac-btn btn-min"></div><div class="mac-btn btn-max"></div>
                <div class="mac-title">追剧控制台 - 实时运行日志</div>
            </div>
            <div class="console" id="logBox">Loading terminal...</div>
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
                    <button type="button" onclick="syncOpenList('139', 'cas')" class="btn-orange" style="height:38px;">🟠 云端同步 (移动云139)</button>
                    <button type="button" onclick="syncOpenList('direct', 'media')" class="btn-blue" style="height:38px;">🎬 云端同步 (常规真实视频)</button>
                    <button type="button" onclick="syncLocalCas()" style="background:#8b5cf6; color:white; border:none; border-radius:6px; height:38px; cursor:pointer; font-weight:bold; transition: 0.2s;" onmouseover="this.style.background='#7c3aed'" onmouseout="this.style.background='#8b5cf6'">🗂️ 批量扫描本地 CAS</button>
                    <button type="button" onclick="clearAllCache()" style="background:#ef4444; color:white; border:none; border-radius:6px; height:38px; cursor:pointer; font-weight:bold; transition: 0.2s;" onmouseover="this.style.background='#dc2626'" onmouseout="this.style.background='#ef4444'">🗑️ 一键清空全区回收站</button>
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
                        <!-- 🌟 修复: 恢复首发卡槽锁定，支持失败滑点 -->
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

                        <!-- 移到左侧的全局 API 区域 -->
                        <h3 style="margin-top:10px; border:none;">🌐 全局基础与 API</h3>
                        <label>基础外网域名 (Server Host)</label>
                        <input type="text" name="server_host" value="{{ cfg.server_host }}" required>
                        <label>189 OpenList 接口地址 (例如 5244)</label>
                        <input type="text" name="openlist_host" value="{{ cfg.openlist_host }}" required>
                        <label>189 OpenList 授权 Token</label>
                        <input type="password" name="openlist_token" value="{{ cfg.openlist_token }}" placeholder="填入189 OpenList Token">
                        <div style="display: flex; gap: 10px; margin-bottom: 15px;">
                            <div style="flex: 1;"><label>绝对销毁倒计时</label><input type="number" name="delete_delay" value="{{ cfg.delete_delay }}" style="margin-bottom:0;" required></div>
                            <div style="flex: 1;"><label>预加载长效护盾</label><input type="number" name="shield_delay" value="{{ cfg.shield_delay }}" style="margin-bottom:0;" required></div>
                        </div>
                        <div style="margin-bottom: 10px;">
                            <label style="display:inline-block; width:220px; font-weight:bold;">🛡️ 网关直链保鲜期 (秒):</label>
                            <input type="number" name="link_expire" value="{{ cfg.get('link_expire', 120) }}" style="width:80px; padding:3px;">
                        </div>

                        <!-- 移到左侧的消息推送区域 -->
                        <h3 style="margin-top:20px; border:none;">📱 消息推送</h3>
                        <label>PushPlus Token (微信推送)</label><input type="text" name="pushplus_token" value="{{ cfg.pushplus_token }}" placeholder="留空则不推送">
                        <label>Telegram Bot Token</label><input type="text" name="tg_bot_token" value="{{ cfg.tg_bot_token }}">
                        <label>Telegram Chat ID</label><input type="text" name="tg_chat_id" value="{{ cfg.tg_chat_id }}">
                        <label>Telegram 代理地址 (Nekobox / Clash 等)</label><input type="text" name="tg_proxy" value="{{ cfg.get('tg_proxy', '') }}" placeholder="例如: http://127.0.0.1:7890 (留空则直连)">
                    </div>

                    <!-- ================= 右侧栏 ================= -->
                    <div>
                        <h3 style="margin-top:0; border:none;">📁 绝对物理隔离路径设置</h3>
                        
                        <div class="path-box">
                            <h4>📁 本地原始 CAS 库 (其他脚本下载存放处)</h4>
                            <label>本地 CAS 源目录</label>
                            <input type="text" name="local_cas_source_dir" value="{{ cfg.local_cas_source_dir }}" required style="margin-bottom:0;">
                        </div>

                        <div class="path-box">
                            <h4>🌐 189 模式A (家庭云稳健秒传)</h4>
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

                        <div class="path-box" style="border-color: #f97316; background: #fff7ed;">
                            <h4 style="color:#ea580c;">🟠 139 移动云盘设置</h4>
                            <label>139 OpenList 接口 (例如 5255)</label>
                            <input type="text" name="openlist_host_139" value="{{ cfg.openlist_host_139 }}">
                            <label>139 OpenList 授权 Token</label>
                            <input type="password" name="openlist_token_139" value="{{ cfg.openlist_token_139 }}">
                            <label>云端 139 CAS 扫描源目录</label>
                            <input type="text" name="network_cas_path_139" value="{{ cfg.network_cas_path_139 }}">
                            <label>139 本地 STRM 独立保存路径</label>
                            <input type="text" name="local_strm_dir_139" value="{{ cfg.local_strm_dir_139 }}" style="margin-bottom:0;">
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
    logger.info(f"[定时销毁] 预定于 {expire_str} 执行家庭云清理任务。")
    
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
            time.sleep(2) 
            if family_client.empty_family_recycle(fam_id, session_key):
                logger.info(f"[执行销毁] 家庭云文件已清除。")
    except: pass
        
    with cache_lock:
        if f_md5 in upload_cache: del upload_cache[f_md5]

def personal_cleanup_worker(file_id, session_key, cookie, f_md5):
    with cache_lock:
        if f_md5 not in upload_cache: return
        expire_time = upload_cache[f_md5]['expire']
    
    expire_str = time.strftime("%H:%M:%S", time.localtime(expire_time))
    logger.info(f"[定时销毁] 预定于 {expire_str} 执行个人云清理任务。")
    
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
            time.sleep(2)
            if personal_client.empty_personal_recycle(session_key, cookie): logger.info(f"[执行销毁] 个人云文件已清除。")
    except: pass
    
    with cache_lock:
        if f_md5 in upload_cache: del upload_cache[f_md5]

@app_main.route('/play', methods=['GET', 'HEAD'])
def play():
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
    if request.method == 'GET' and range_header and drive_type not in ['189_native', '139', 'direct']:
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
            cas_str = urllib.parse.unquote(cas.strip())
            if cas_str.startswith('{'): j = json.loads(cas_str)
            else:
                cas_b64 = cas_str.replace(' ', '+')
                cas_b64 += "=" * ((4 - len(cas_b64) % 4) % 4)
                j = json.loads(base64.b64decode(cas_b64).decode('utf-8'))
                
            f_md5 = j.get('md5') or j.get('fileMd5') or j.get('fileMD5')
            safe_name = j.get('name') or j.get('fileName') or "unknown.mp4"
            cas_payload = base64.b64encode(json.dumps(j, ensure_ascii=False).encode('utf-8')).decode('utf-8')
            
            with cache_lock:
                if f_md5 in native_link_cache:
                    cached_url, expire_time = native_link_cache[f_md5]
                    if time.time() < expire_time: return redirect(cached_url)
                    else: del native_link_cache[f_md5] 
            
            logger.info(f"========== 🕵️‍♂️ 原生直连(模式B) 链路日志 START ==========")
            logger.info(f"▶️ [1] 请求原生直通: {safe_name}")

            target_dir = cfg.get('network_cas_path_native', '/177/177-原生直连').rstrip('/')
            temp_file_name = f"temp_play_{f_md5}.cas"
            target_path = f"{target_dir}/{temp_file_name}"
            
            put_headers = headers.copy()
            put_headers.update({"File-Path": urllib.parse.quote(target_path, safe='/'), "Content-Length": str(len(cas_payload.encode('utf-8'))), "Content-Type": "application/octet-stream"})
            
            logger.info(f"☁️ [2] 虚空造物：正在向云端动态写入临时伪装凭证...")
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
            
            logger.info(f"📥 [3] 成功获取底层 raw_url, 启动防风控嗅探...")
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
                    logger.warning(f"⚠️ [警告] 嗅探遇 500，去 Range 破冰...")
                    range_key = next((k for k in unwrap_headers if k.lower() == 'range'), None)
                    if range_key: del unwrap_headers[range_key]
                    unwrap_res = requests.get(raw_url, headers=unwrap_headers, allow_redirects=False, timeout=15, stream=True)
                    status_code = unwrap_res.status_code
                    headers_dict = dict(unwrap_res.headers)
                    unwrap_res.close()

                logger.info(f"🔍 [4] 嗅探完成！状态码: {status_code}")

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
                    logger.info(f"✅ [5] 完美触发！拿到 189 官方直链！")
                    logger.info(f"[播放放行] 节点: {urllib.parse.urlparse(final_return_url).netloc} | 地址: {truncate_url(final_return_url)}")
                    logger.info(f"========== 🕵️‍♂️ 原生直连(模式B) 链路日志 END ==========\n")
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
    # 🎬 常规媒体直通 / 🟠 移动云 139 通用跳转
    # ==========================================
    if drive_type in ['139', 'direct']:
        if not file_path_param: return "❌ 请求缺少 path 参数", 400

        with cache_lock:
            if file_path_param in native_link_cache:
                cached_url, expire_time = native_link_cache[file_path_param]
                if time.time() < expire_time: return redirect(cached_url)
                else: del native_link_cache[file_path_param] 

        is_139 = (drive_type == '139')
        ol_host = cfg.get('openlist_host_139' if is_139 else 'openlist_host', 'http://127.0.0.1:5244').rstrip('/')
        ol_token = cfg.get('openlist_token_139' if is_139 else 'openlist_token', '')
        headers = {"Authorization": ol_token} if ol_token else {}
        
        log_tag = "139云盘" if is_139 else "常规真视频"
        logger.info(f"========== 🕵️‍♂️ {log_tag} 链路日志 START ==========")
        logger.info(f"▶️ [1] 触发请求: {file_path_param}")

        try:
            get_res = requests.post(f"{ol_host}/api/fs/get", json={"path": file_path_param, "password": ""}, headers=headers, timeout=15)
            if get_res.status_code == 200:
                data_obj = get_res.json().get('data')
                raw_url = data_obj.get('raw_url') if isinstance(data_obj, dict) else None
                if raw_url:
                    logger.info(f"📥 [2] 成功获取底层 raw_url...")
                    
                    # 🌟 核心修复：极致净化请求头！剥离 Vidhub 专属复杂参数，并防止 OpenList Token 污染 139 CDN 导致 400 报错
                    unwrap_headers = {
                        "User-Agent": request.headers.get('User-Agent', 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'),
                        "Accept-Encoding": "identity"
                    }
                    # 只有当 raw_url 真的指向你自己的 OpenList 代理时，才允许塞入 Token
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
                        
                        logger.info(f"🔍 [3] 嗅探完成！状态码: {status_code}")
                        
                        if status_code in [301, 302, 303, 307, 308]:
                            final_cdn_url = headers_dict.get('Location')
                            if final_cdn_url:
                                with cache_lock: native_link_cache[file_path_param] = (final_cdn_url, time.time() + 120)
                                logger.info(f"[播放放行] 节点: {urllib.parse.urlparse(final_cdn_url).netloc} | 地址: {truncate_url(final_cdn_url)}")
                                logger.info(f"========== 🕵️‍♂️ {log_tag} 链路日志 END ==========\n")
                                return redirect(final_cdn_url)
                            else: 
                                logger.error(f"❌ {log_tag} 缺失直链跳转地址")
                                return "缺失直链", 500
                        elif status_code in [200, 206]:
                            with cache_lock: native_link_cache[file_path_param] = (raw_url, time.time() + 120)
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
            cas_str = urllib.parse.unquote(cas.strip())
            if cas_str.startswith('{'): j = json.loads(cas_str)
            else:
                cas_b64 = cas_str.replace(' ', '+')
                cas_b64 += "=" * ((4 - len(cas_b64) % 4) % 4)
                j = json.loads(base64.b64decode(cas_b64).decode('utf-8'))
            
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
            
            # 阶段 1：极速查缓存
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

            # 阶段 2：等待并发线程完成
            if is_processing_by_others and not download_url:
                for _ in range(50):
                    time.sleep(0.2)
                    with cache_lock:
                        if f_md5 in upload_cache and upload_cache[f_md5].get('download_url'):
                            download_url = upload_cache[f_md5]['download_url']
                            break

            # 阶段 3：全新生成 -> 建立四核矩阵攻击队列 (跨账号无缝滑点)
            if not download_url and not is_processing_by_others:
                with cache_lock: upload_cache[f_md5] = {'fid': 'processing', 'expire': current_time + cfg.get('delete_delay', 600), 'download_url': None}
                
                # 过滤出存在有效数据的账号
                valid_accs = []
                for i, acc in enumerate(cfg.get('accounts', [])):
                    fam_valid = bool(acc.get('family_id') and acc.get('family_folder_id'))
                    per_valid = bool(acc.get('personal_folder_id'))
                    can_login = bool((acc.get('username') and acc.get('password')) or i == 3)
                    if (fam_valid or per_valid) and (acc.get('session_key') or can_login):
                        valid_accs.append((i, acc))
                
                if not valid_accs: return "未配置任何有效网盘卡槽", 500

                # ======== 🌟 终极版：打头阵优先 + 矩阵兜底策略 ========
                strategy = cfg.get('cloud_strategy', 'hash')
                strategy_to_idx = {'slot1': 0, 'slot2': 1, 'slot3': 2, 'slot4': 3}
                
                if strategy in strategy_to_idx:
                    target_idx = strategy_to_idx[strategy]
                    # 寻找被钦定打头阵的卡槽
                    vanguard = next((a for a in valid_accs if a[0] == target_idx), None)
                    if vanguard:
                        # 战术阵型：钦定号排绝对第一，其他存活的号跟在后面排队接盘！
                        candidates = [vanguard] + [a for a in valid_accs if a[0] != target_idx]
                    else:
                        # 钦定的号没填账号或失效了，退守哈希阵型
                        logger.warning(f"⚠️ [防死锁] 优先卡槽 {target_idx+1} 未就绪，退守全局哈希滑点调度！")
                        hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(valid_accs)
                        candidates = valid_accs[hash_idx:] + valid_accs[:hash_idx]
                
                elif strategy == 'hash':
                    hash_idx = int(hashlib.md5(bind_key.encode('utf-8')).hexdigest(), 16) % len(valid_accs)
                    candidates = valid_accs[hash_idx:] + valid_accs[:hash_idx]
                else: # random
                    candidates = valid_accs.copy()
                    random.shuffle(candidates)
                # ===================================================

                mode_a_cfg = cfg.get('mode_a_channel', 'mix_f2p')
                search_sequence = []
                # 构建扁平化的瀑布流攻击序列 (例如：家1->家2->家3->家4->个1->个2...)
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
                    logger.info(f"[{log_tag}调度] 开始处理: {safe_name}")

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
                            # +++ 完美成功！更新缓存并跳出攻击队列 +++
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
                            break # 黑名单没救，直接中止
                            
                        elif any(k in err_str for k in ["auth_fail", "session", "111", "notlogin"]):
                            logger.warning(f"[{log_tag}凭证失效] 尝试执行自愈...")

                            target_sk, target_cookie = refresh_account_logic(s_idx, cfg)
                            if target_sk:
                                try: # 自愈后重试一次
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
                            # 🎯 核心解决：如果是空间不足、体积超限、Null 报错
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
                
                # 🌟 修复：播放放行 15 秒视觉消音
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
                if file_type in ['cas', 'cas_native'] and ext == 'cas': result_list.append(f"{current_path}/{f['name']}")
                elif file_type == 'media' and ext in ['mp4', 'mkv', 'ts', 'avi', 'mov', 'webm', 'flv', 'iso']: result_list.append(f"{current_path}/{f['name']}")
    except: pass

def generate_strm_from_openlist_to_local(target_path=None, drive_type='189', file_type='cas'):
    cfg = read_config()
    
    if drive_type == '139':
        scan_root = target_path if target_path else cfg.get('network_cas_path_139', '')
        base_cas_path = cfg.get('network_cas_path_139', '')
        local_strm_dir = cfg.get('local_strm_dir_139', '')
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
        
    os.makedirs(local_strm_dir, exist_ok=True)
    headers = {"Authorization": api_token} if api_token else {}
    
    if target_path and drive_type != '139': warm_up_parent(target_path, headers, api_host)
    
    logger.info(f"[扫描启动] OpenList [{drive_type} / {file_type.upper()}] -> 区域: {scan_root}")
    cas_files = []
    scan_openlist_recursive(scan_root, headers, cas_files, api_host, file_type)
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
            
            target_local_dir = os.path.join(local_strm_dir, rel_dir)
            os.makedirs(target_local_dir, exist_ok=True)
            
            if file_type == 'cas_native': strm_path = os.path.join(target_local_dir, f"{base_name}-189.strm")
            elif drive_type == '139': strm_path = os.path.join(target_local_dir, f"{base_name}-139.strm")
            else: strm_path = os.path.join(target_local_dir, f"{base_name}.strm")
                
            if os.path.exists(strm_path): continue
            
            if count > 0 and count % 50 == 0: time.sleep(3)
                
            if file_type == 'media':
                with open(strm_path, "w", encoding="utf-8") as f: f.write(f"{cfg['server_host']}/play?drive=direct&path={urllib.parse.quote(full_path)}&show={urllib.parse.quote(show_name)}")
                count += 1
                continue
                
            if drive_type == '139':
                with open(strm_path, "w", encoding="utf-8") as f: f.write(f"{cfg['server_host']}/play?drive=139&path={urllib.parse.quote(full_path)}&show={urllib.parse.quote(show_name)}")
                count += 1
                continue
            
            get_res = req_session.post(f"{api_host}/api/fs/get", json={"path": full_path}, headers=headers, timeout=10).json()
            raw_url = get_res.get("data", {}).get("raw_url")
            if not raw_url: continue
            
            cas_content = req_session.get(raw_url, timeout=10).text.strip()
            
            if file_type == 'cas_native': strm_data = f"{cfg['server_host']}/play?drive=189_native&cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            else: strm_data = f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            
            with open(strm_path, "w", encoding="utf-8") as f: f.write(strm_data)
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
    source_dir = cfg.get('local_cas_source_dir', '')
    if not source_dir or not os.path.exists(source_dir): return

    logger.info(f"[本地扫描] 启动本地 CAS 双轨扫描 -> 目录: {source_dir}")
    base_dir_a = cfg.get('local_strm_dir', '')
    base_dir_b = cfg.get('local_strm_dir_native', '')
    
    count = 0
    for root, dirs, files in os.walk(source_dir):
        for file in files:
            if file.endswith('.cas'):
                full_path = os.path.join(root, file)
                rel_path = full_path[len(source_dir):].lstrip('/\\')
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
                        
                    if base_dir_a:
                        target_a = os.path.join(base_dir_a, rel_dir)
                        os.makedirs(target_a, exist_ok=True)
                        strm_a = os.path.join(target_a, f"{base_name}.strm")
                        if not os.path.exists(strm_a):
                            with open(strm_a, "w", encoding="utf-8") as fa:
                                fa.write(f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}")
                            count += 1
                            
                    if base_dir_b:
                        target_b = os.path.join(base_dir_b, rel_dir)
                        os.makedirs(target_b, exist_ok=True)
                        strm_b = os.path.join(target_b, f"{base_name}-189.strm")
                        if not os.path.exists(strm_b):
                            with open(strm_b, "w", encoding="utf-8") as fb:
                                fb.write(f"{cfg['server_host']}/play?drive=189_native&cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}")
                            count += 1
                except: pass
                    
    if count > 0: logger.info(f"[扫描完毕] 共增量生成 {count} 个 STRM 文件")

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
        target_local_dir_native = data.get('target_local_dir_native')
        strm_name = data.get('strm_name')                
        show_name = data.get('show_name')                
        mode = data.get('mode', 'cas') 

        if not all([source_cas_path, target_local_dir, strm_name, show_name]): return jsonify({"code": 400, "msg": "指令参数不全"}), 400
        if not os.path.exists(source_cas_path): return jsonify({"code": 404, "msg": f"找不到源文件: {source_cas_path}"}), 404

        cfg = read_config()
        with open(source_cas_path, 'r', encoding='utf-8') as f: cas_content = f.read().strip()
        base_name = os.path.splitext(strm_name)[0]

        if mode in ['cas', 'both']:
            os.makedirs(target_local_dir, exist_ok=True)
            strm_path_a = os.path.join(target_local_dir, strm_name)
            strm_data_a = f"{cfg['server_host']}/play?cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            with open(strm_path_a, "w", encoding="utf-8") as f: f.write(strm_data_a)

        if mode in ['cas_native', 'both']:
            out_dir_b = target_local_dir_native if target_local_dir_native else target_local_dir
            os.makedirs(out_dir_b, exist_ok=True)
            strm_path_b = os.path.join(out_dir_b, f"{base_name}-189.strm")
            strm_data_b = f"{cfg['server_host']}/play?drive=189_native&cas={urllib.parse.quote(cas_content)}&show={urllib.parse.quote(show_name)}"
            with open(strm_path_b, "w", encoding="utf-8") as f: f.write(strm_data_b)

        return jsonify({"code": 200, "msg": "success"}), 200
    except Exception as e: return jsonify({"code": 500, "msg": str(e)}), 500

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
            
            if lan_ip: logger.info(f"[网络嗅探] 识别为内网播放，下发路由重定向")
            
            # 🌟 修复：劫持放行 15 秒视觉消音
            now_t = time.time()
            if now_t - print_throttle_cache.get(f"hijack_{item_id}", 0) > 15:
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
    time.sleep(60) # ⚡ 开机60秒后启动第一次巡逻
    while True:
        try:
            cfg = read_config()
            has_checked = False
            
            for i, acc in enumerate(cfg.get('accounts', [])):
                fam_id, fold_id = acc.get('family_id'), acc.get('family_folder_id')
                per_id = acc.get('personal_folder_id')
                sk, cookie = acc.get('session_key'), acc.get('cookie')
                
                # 如果这个卡槽没填账号（且不是大号），直接跳过
                if not (acc.get('username') and acc.get('password')) and i != 3: continue
                
                if not has_checked:
                    logger.info("📡 [巡逻雷达] 正在对矩阵卡槽进行后台健康体检...")
                    has_checked = True
                
                is_alive = True
                # 🌟 只要填了目录 ID 但凭证是空的，当场判定阵亡，强制激活自愈！
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
                
                time.sleep(random.randint(5, 10)) # 查完一个号歇几秒防风控
            
            if has_checked:
                logger.info("✅ [巡逻完毕] 本轮健康体检结束，所有卡槽运转正常。")
                
        except Exception as e: 
            logger.error(f"❌ [巡逻异常]: {e}")
            
        time.sleep(3600) # 查完一轮，睡 1 小时后再次巡逻

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