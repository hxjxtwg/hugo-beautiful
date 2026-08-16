---

title: "Termux之cas_server"

author: "xxsky"

type: "posts"

date: 2026-07-01T14:04:33+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

本地计算视频文件分段md5生成cas文件

<!--more-->

cas_server.py:

```
import os
import json
import base64
import hashlib
import time
import argparse
import sys
from flask import Flask, request, jsonify, render_template_string

app = Flask(__name__)

# ==========================================
# 核心配置区
# ==========================================
# Termux 服务器本地 CAS 文件暂存根目录 (分别独立配置)
CAS_BASE_DIRS = {
    "189": "/storage/emulated/0/Download/189cas",
    "139": "/storage/emulated/0/Download/139cas"
}
for d in CAS_BASE_DIRS.values():
    os.makedirs(d, exist_ok=True)

# 28GB 阈值（字节），超过此大小自动切换为 20MB 分片
LARGE_FILE_THRESHOLD = 26 * 1024 * 1024 * 1024  

# 支持批量遍历收割的视频扩展名
VIDEO_EXTENSIONS = ('.mp4', '.mkv', '.avi', '.mov', '.flv', '.wmv', '.rmvb', '.ts')

def get_dynamic_slice_size(file_size):
    """根据文件大小动态计算分片大小（防止大文件片数超 3000 爆限制）"""
    if file_size > LARGE_FILE_THRESHOLD:
        return 20 * 1024 * 1024  # >26G 自动启用 20MB 分片
    return 10 * 1024 * 1024      # 普通文件使用 10MB 分片

def calculate_single_file_cas(file_path, cloud_type="189"):
    """核心底层算法：全速解构单个物理文件并计算标准级联哈希或SHA256"""
    if not os.path.exists(file_path) or os.path.isdir(file_path):
        return None

    file_size = os.path.getsize(file_path)
    file_name = os.path.basename(file_path)
    
    # ======== 139 移动云盘逻辑 ========
    if cloud_type == "139":
        sha256_algo = hashlib.sha256()
        with open(file_path, 'rb') as f:
            while True:
                chunk = f.read(4 * 1024 * 1024)
                if not chunk:
                    break
                sha256_algo.update(chunk)
                
        return {
            "name": file_name,
            "size": file_size,
            "md5": "",
            "sliceMd5": "",
            "sha256": sha256_algo.hexdigest().lower(),
            "create_time": str(int(time.time())),
            "part_size_used": "139无需分片"
        }
    
    # ======== 189 天翼云盘逻辑 ========
    current_slice_size = get_dynamic_slice_size(file_size)
    chunk_size = 4 * 1024 * 1024  # 4MB 流式读取缓冲区

    full_md5_compressor = hashlib.md5()
    slice_md5_hexs = []

    with open(file_path, 'rb') as f:
        off = 0
        while off < file_size:
            slice_end = min(off + current_slice_size, file_size)
            slice_md5_compressor = hashlib.md5()
            inner = off
            
            while inner < slice_end:
                read_len = min(chunk_size, slice_end - inner)
                chunk = f.read(read_len)
                if not chunk:
                    break
                full_md5_compressor.update(chunk)
                slice_md5_compressor.update(chunk)
                inner += read_len

            slice_md5_hexs.append(slice_md5_compressor.hexdigest().upper())
            off = slice_end

    full_md5 = full_md5_compressor.hexdigest().upper()

    if file_size > current_slice_size:
        # 在 Python 端，换行符级联
        joined_text = "\n".join(slice_md5_hexs)
        slice_md5 = hashlib.md5(joined_text.encode('utf-8')).hexdigest().upper()
    else:
        slice_md5 = full_md5

    return {
        "name": file_name,
        "size": file_size,
        "md5": full_md5,
        "sliceMd5": slice_md5,
        "create_time": str(int(time.time())),
        "cloud": "189",
        "part_size_used": f"{current_slice_size / (1024*1024):.0f}MB"
    }

# ==========================================
# 终极一体化网页端 UI (原生防转义架构)
# ==========================================
# 采用 raw 字符串 (r"") 避免转义字符导致前端 JS 崩溃
HTML_TEMPLATE = r"""
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>天翼/移动云盘 CAS 双擎控制中心</title>
<script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/spark-md5@3.0.2/spark-md5.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/js-sha256/0.9.0/sha256.min.js"></script>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
:root { --bg: #0f172a; --card: #1e293b; --border: #334155; --text: #e2e8f0; --dim: #94a3b8; --accent: #3b82f6; --accent-h: #2563eb; --ok: #22c55e; --err: #ef4444; --json: #f59e0b; }
body { font-family: -apple-system, BlinkMacSystemFont, sans-serif; background: var(--bg); color: var(--text); padding: 24px; min-height: 100vh; }
.wrap { max-width: 850px; margin: 0 auto; }
h1 { font-size: 22px; margin-bottom: 4px; color: #facc15; }
.sub { color: var(--dim); font-size: 13px; margin-bottom: 20px; }
.card { background: var(--card); border: 1px solid var(--border); border-radius: 12px; padding: 20px; margin-bottom: 16px; }
.card-t { font-size: 14px; font-weight: 600; margin-bottom: 14px; color: #38bdf8; display:flex; align-items:center; gap:6px; }

.path-builder { display: grid; grid-template-columns: repeat(auto-fit, minmax(130px, 1fr)); gap: 12px; margin-bottom: 12px; }
.pb-group { display: flex; flex-direction: column; gap: 6px; }
.pb-group label { font-size: 12px; color: var(--dim); font-weight: bold; }
.pb-input { padding: 9px 12px; border-radius: 6px; border: 1px solid var(--border); background: var(--bg); color: var(--text); outline: none; font-size: 13px; transition: border 0.2s; }
.pb-input:focus { border-color: var(--accent); }
.final-path-box { background: rgba(59,130,246,0.1); border: 1px dashed var(--accent); padding: 12px; border-radius: 8px; margin-top: 10px; }
.final-path-text { color: #fff; font-family: monospace; font-size: 13px; word-break: break-all; }

/* 运行模式切换 */
.mode-tabs { display: flex; gap: 10px; margin-bottom: 15px; border-bottom: 1px solid var(--border); padding-bottom: 10px; flex-wrap: wrap; }
.tab { padding: 8px 16px; border-radius: 6px; font-size: 13px; cursor: pointer; background: var(--border); color: var(--dim); border: none; font-weight: bold; transition: all 0.2s; }
.tab:hover { background: #475569; }
.tab.active { background: var(--accent); color: #fff; }
.tab-json { border: 1px solid var(--json); color: var(--json); background: transparent; }
.tab-json.active { background: var(--json); color: #fff; }

.drop { border: 2px dashed var(--border); border-radius: 12px; padding: 35px; text-align: center; cursor: pointer; transition: all 0.2s; }
.drop:hover, .drop.over { border-color: var(--accent); background: rgba(59,130,246,.05); }
.drop-text { color: var(--dim); font-size: 13px; margin-top: 5px; }

/* 服务器批量解析区布局 */
.server-zone-grid { display: grid; grid-template-columns: 1fr; gap: 10px; }
.server-path-row { display: flex; flex-direction: column; gap: 6px; }
.server-path-row label { font-size: 12px; font-weight: bold; color: var(--dim); }
.server-path-input { padding: 12px; background: #020617; border: 1px solid var(--border); border-radius: 8px; color: #a7f3d0; font-family: monospace; outline: none; font-size: 13px; width: 100%; transition: border 0.2s; }
.server-path-input:focus { border-color: var(--accent); }

.flist { margin-top: 12px; max-height: 350px; overflow-y: auto; }
.frow { display: flex; align-items: center; gap: 10px; padding: 12px; background: var(--bg); border-radius: 8px; margin-bottom: 6px; font-size: 13px; border-left: 4px solid var(--border); }
.frow.wait { border-left-color: var(--dim); }
.frow.work { border-left-color: var(--accent); }
.frow.done { border-left-color: var(--ok); }
.frow.fail { border-left-color: var(--err); }
.frow .name { flex: 1; word-break: break-all; }
.frow .size { color: var(--dim); white-space: nowrap; font-family: monospace; }
.frow .st { font-size: 12px; white-space: nowrap; min-width: 40px; text-align: right; }
.st.done { color: var(--ok); }
.st.work { color: var(--accent); }
.st.fail { color: var(--err); }
.st.wait { color: var(--dim); }

.btns { display: flex; gap: 10px; flex-wrap: wrap; margin-top: 15px; }
.btn { padding: 10px 20px; border: none; border-radius: 8px; font-size: 13px; font-weight: 500; cursor: pointer; transition: all 0.2s; }
.btn:disabled { opacity: .4; cursor: not-allowed; }
.btn-p { background: var(--accent); color: #fff; }
.btn-p:hover:not(:disabled) { background: var(--accent-h); }
.btn-s { background: var(--border); color: var(--text); }
.btn-s:hover { background: #475569; }
.btn-g { background: var(--ok); color: #fff; }
.btn-g:hover:not(:disabled) { background: #16a34a; }

.out { background: var(--bg); border: 1px solid var(--border); border-radius: 8px; padding: 14px; font-family: monospace; font-size: 12px; line-height: 1.6; white-space: pre-wrap; color: #a7f3d0; max-height: 400px; overflow-y: auto; }
</style>
</head>
<body>
<div class="wrap">
  <h1>⚡ 双擎云盘 CAS 远程集成控制中心</h1>
  <p class="sub">全面支持 189天翼云(分片拼装) & 139移动云盘(SHA256) · 物理目录严格隔离存放</p>

  <!-- 1. Emby 路径配置 -->
  <div class="card" id="cardCategory">
    <div class="card-t">📁 1. 自动归档分类配置 (决定落盘文件夹)</div>
    <div class="path-builder">
      <div class="pb-group">
        <label>主分类</label>
        <select id="catMain" class="pb-input" onchange="buildPath()">
          <option value="华语剧">华语剧</option><option value="欧美剧">欧美剧</option><option value="日韩剧">日韩剧</option>
          <option value="短剧">短剧</option><option value="国漫">国漫</option><option value="日漫">日漫</option>
          <option value="华语电影">华语电影</option><option value="欧美电影">欧美电影</option><option value="日韩电影">日韩电影</option>
          <option value="综艺">综艺</option><option value="纪录片">纪录片</option><option value="演唱会">演唱会</option>
        </select>
      </div>
      <div class="pb-group"><label>影视名称</label><input type="text" id="catName" class="pb-input" placeholder="可手动填 / 拖入文件自动识别" oninput="buildPath()"></div>
      <div class="pb-group"><label>年份</label><input type="text" id="catYear" class="pb-input" placeholder="2026" oninput="buildPath()"></div>
      <div class="pb-group" id="seasonGroup"><label>季数</label><input type="text" id="catSeason" class="pb-input" placeholder="1" oninput="buildPath()"></div>
    </div>
    <div class="final-path-box">
      <label>即将落盘的物理相对目录 (挂载在对应云盘基础目录下)：</label>
      <div class="final-path-text" id="finalPathDisplay"></div>
      <input type="hidden" id="categoryInput">
    </div>
  </div>

  <!-- 2. 操作模式切换与执行域 -->
  <div class="card">
    <div class="card-t">⚙️ 2. 选择工作引擎与目标云盘</div>
    <div class="mode-tabs">
      <select id="cloudType" class="pb-input" style="background:var(--accent); color:white; font-weight:bold; border:none; margin-right:15px; cursor:pointer;" onchange="buildPath()">
          <option value="189">☁️ 天翼云盘 (189)</option>
          <option value="139">☁️ 移动云盘 (139)</option>
      </select>
      <button class="tab active" id="tabLocal" onclick="switchMode('local')">🌐 跨网络计算 </button>
      <button class="tab" id="tabServer" onclick="switchMode('server')">🖥️ 服务器本地 </button>
      <button class="tab tab-json" id="tabJson" onclick="switchMode('json')">🔄 异构 JSON 转换/拆包</button>
    </div>

    <!-- 模式 A：浏览器跨网络选择计算 -->
    <div id="panelLocal">
      <div class="drop" id="dropLocal" onclick="document.getElementById('finput').click()">
        <div style="font-size: 24px;">☁️</div>
        <div class="drop-text">点击或拖拽电脑/手机本地视频到此处<br><span style="color:#64748b;font-size:11px;">(文件添加后处于等待状态，需手动点击开始按钮执行计算)</span></div>
      </div>
      <input type="file" id="finput" multiple style="display: none;" onchange="handleLocalFiles(event)">
    </div>

    <!-- 模式 B：控制服务器本地视频批量收割 -->
    <div id="panelServer" style="display: none;">
      <div class="server-zone-grid">
        <div class="server-path-row">
            <label>1. 服务器基础挂载路径 (如 /sdcard/Download)</label>
            <input type="text" id="serverBasePath" class="server-path-input" value="/data/data/com.termux/files/home/Downloads">
        </div>
        <div class="server-path-row">
            <label>2. ➕ 手动追加【媒体文件夹】(自动扫描该目录下所有视频)</label>
            <input type="text" id="serverSubPath" class="server-path-input" placeholder="例如: 华语剧/剑兰女探社 (2026)/Season 1">
        </div>
        <div class="server-path-row">
            <label>3. ➕ [可选] 锁定具体的单一视频文件 (留空则执行全目录收割)</label>
            <input type="text" id="serverSpecificFile" class="server-path-input" placeholder="例如: S01E01.mp4">
        </div>
      </div>
    </div>

    <!-- 模式 C：异构 JSON 转换 -->
    <div id="panelJson" style="display: none;">
      <div class="drop" id="dropJson" onclick="document.getElementById('fjson').click()" style="border-color:var(--json)">
        <div style="font-size: 24px;">📦</div>
        <div class="drop-text" style="color:var(--json)">点击或拖拽别人分享的批量 JSON 文件到此处<br><span style="font-size:11px;">(自动按 JSON 内的目录结构拆分成单体 .cas 落地服务器)</span></div>
      </div>
      <input type="file" id="fjson" accept=".json" style="display: none;" onchange="handleJsonFile(event)">
    </div>

    <!-- 任务队列面板 -->
    <div class="flist" id="flist"></div>
    
    <!-- 全局操作面板 -->
    <div class="btns">
      <button class="btn btn-g" id="btnGo" onclick="executeTasks()">🚀 执行任务</button>
      <button class="btn btn-s" id="btnCopy" disabled onclick="copyJson()">📋 复制json</button>
      <button class="btn btn-p" id="btnZip" disabled onclick="downloadCasFile()">📥 下载.cas文件</button>
      <button class="btn btn-s" id="btnClear" onclick="resetForm()">🗑️ 清空队列</button>
    </div>
  </div>

  <!-- 3. 输出预览 -->
  <div class="card" id="outCard" style="display:none;">
    <div class="card-t">📋 明文json预览</div>
    <div class="out" id="out"></div>
  </div>
</div>

<script>
const $ = function(id) { return document.getElementById(id); };

// ==== 工作流状态全局变量 ====
let activeMode = 'local';
let localQueue = [];  
let isRunning = false;
let currentPayloads = [];

const LARGE_FILE_THRESHOLD = 26 * 1024 * 1024 * 1024;
const CHUNK_SIZE = 4 * 1024 * 1024;

function getPartSize(fileSize) {
    if (fileSize > LARGE_FILE_THRESHOLD) {
        return 20 * 1024 * 1024;
    }
    return 10 * 1024 * 1024;
}

// ==== 界面与工具库函数 ====
function switchMode(mode) {
    if (isRunning) {
        alert("请等待当前任务结束再切换模式");
        return;
    }
    activeMode = mode;
    
    // UI Tab 切换
    $('tabLocal').className = 'tab';
    $('tabServer').className = 'tab';
    $('tabJson').className = 'tab tab-json';
    
    $('panelLocal').style.display = 'none';
    $('panelServer').style.display = 'none';
    $('panelJson').style.display = 'none';
    
    if (mode === 'local') {
        $('tabLocal').classList.add('active');
        $('panelLocal').style.display = 'block';
        $('cardCategory').style.display = 'block';
    } else if (mode === 'server') {
        $('tabServer').classList.add('active');
        $('panelServer').style.display = 'block';
        $('cardCategory').style.display = 'block';
    } else if (mode === 'json') {
        $('tabJson').classList.add('active');
        $('panelJson').style.display = 'block';
        $('cloudType').value = '139'; // 强制跳到 139 模式
        $('cardCategory').style.display = 'none'; // 隐藏上方手动分类，因为从 json 自动读
    }
    resetForm();
}

function buildPath() {
    try {
        const main = $('catMain').value;
        const name = $('catName').value.trim();
        const year = $('catYear').value.trim();
        let season = $('catSeason').value.trim();
        
        const isMovie = main.indexOf('电影') !== -1 || main.indexOf('演唱会') !== -1;
        $('seasonGroup').style.display = isMovie ? 'none' : 'flex';
        if (isMovie) {
            season = '';
        }

        let path = main;
        if (name) {
            path += '/' + name;
            if (year) {
                path += ' (' + year + ')';
            }
        }
        if (season && !isMovie) {
            if (/^\d+$/.test(season)) {
                path += '/Season ' + season;
            } else {
                path += '/' + season;
            }
        }
        
        // 界面预览路径加上云盘名称以作区分
        const cloudType = $('cloudType').value;
        $('finalPathDisplay').innerText = cloudType + 'cas/' + path + '/';
        $('categoryInput').value = path;
    } catch (e) {
        console.error("路径构建失败:", e);
    }
}

function smartExtract(filename) {
    if ($('catName').value.trim() !== '') return;
    let clean = filename.replace(/[_\s\[\]【】]/g, '.');
    
    let ym = clean.match(/\.(19\d{2}|20\d{2})\./);
    if (ym) $('catYear').value = ym[1];
    
    let sm = clean.match(/\.[Ss](\d{1,2})/);
    if (sm) $('catSeason').value = parseInt(sm[1], 10);
    
    let si = clean.search(/\.(19\d{2}|20\d{2})\.|\.[Ss]\d{1,2}/);
    if (si > 0) {
        $('catName').value = clean.substring(0, si).replace(/\./g, ' ').trim();
    } else {
        $('catName').value = clean.split(/[.-]/)[0].trim();
    }
    buildPath();
}

function fmt(b) {
    if (b < 1048576) return (b/1024).toFixed(1)+'KB';
    if (b < 1073741824) return (b/1048576).toFixed(2)+'MB';
    return (b/1073741824).toFixed(2)+'GB';
}

function esc(s) { 
    return String(s).replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); 
}

// === 拖拽组件事件绑定 ===
const bindDrop = (elId, handler) => {
    const el = $(elId);
    if(!el) return;
    el.addEventListener('dragover', function(e) { e.preventDefault(); el.classList.add('over'); });
    el.addEventListener('dragleave', function() { el.classList.remove('over'); });
    el.addEventListener('drop', function(e) {
        e.preventDefault(); el.classList.remove('over');
        if (e.dataTransfer.files.length) handler({ target: { files: e.dataTransfer.files } });
    });
};
bindDrop('dropLocal', handleLocalFiles);
bindDrop('dropJson', handleJsonFile);

// =====================================
// 模式 C：异构 JSON 转换拆包逻辑
// =====================================
function handleJsonFile(e) {
    const files = e.target.files;
    if (!files.length) return;
    const file = files[0];
    
    const reader = new FileReader();
    reader.onload = function(evt) {
        try {
            const parsedData = JSON.parse(evt.target.result);
            if (!Array.isArray(parsedData)) throw new Error("JSON 不是数组格式");
            
            $('flist').innerHTML = '';
            localQueue = [];
            
            for (let i = 0; i < parsedData.length; i++) {
                const item = parsedData[i];
                if (!item.name || !item.sha256) continue;
                
                // 从长路径中抠出文件名和目录
                // 例如: "动漫/神澜奇域无双珠 (2022)/Season 1/S01E28.mp4"
                const normalizedPath = item.name.replace(/\\/g, '/');
                const lastSlashIdx = normalizedPath.lastIndexOf('/');
                let filename = normalizedPath;
                let categoryPath = "未分类归档";
                
                if (lastSlashIdx !== -1) {
                    filename = normalizedPath.substring(lastSlashIdx + 1);
                    categoryPath = normalizedPath.substring(0, lastSlashIdx);
                }
                
                // 组装成我们的标准 Payload
                const casPayload = {
                    name: filename,
                    size: parseInt(item.size),
                    md5: "",
                    sliceMd5: "",
                    sha256: item.sha256,
                    create_time: String(Math.floor(Date.now() / 1000))
                };
                
                // 推入队列
                localQueue.push({
                    file: { name: filename, size: item.size },
                    state: 'wait',
                    progress: 0,
                    backendObj: null,
                    casObj: casPayload,
                    autoCategory: categoryPath
                });
            }
            
            if(localQueue.length > 0) {
                renderLocalList();
                alert(`成功解析了 ${localQueue.length} 个文件特征，点击执行任务进行落盘。`);
            } else {
                alert("未在 JSON 中找到有效的 name 和 sha256 字段。");
            }
            
        } catch (err) {
            alert("解析 JSON 失败: " + err.message);
        }
    };
    reader.readAsText(file);
    $('fjson').value = '';
}


function handleLocalFiles(e) {
    const files = e.target.files;
    if (!files.length) return;
    
    smartExtract(files[0].name);
    
    for (let i = 0; i < files.length; i++) {
        localQueue.push({ file: files[i], state: 'wait', progress: 0, backendObj: null, casObj: null });
    }
    renderLocalList();
    $('finput').value = '';
}

function renderLocalList() {
    $('flist').innerHTML = '';
    for (let i = 0; i < localQueue.length; i++) {
        const item = localQueue[i];
        const row = document.createElement('div');
        row.className = 'frow ' + item.state;
        
        let stText = '';
        if (item.state === 'wait') stText = '⏳等待触发';
        else if (item.state === 'work') stText = '⚙️处理中';
        else if (item.state === 'done') stText = '✓完成';
        else stText = '✗失败';
        
        const prog = (item.state === 'work' && item.progress > 0) ? ' ' + Math.round(item.progress * 100) + '%' : '';
        
        let badge = '';
        if (item.backendObj) {
            badge = '<span style="color:var(--ok); font-size:12px; margin-left:8px;">[本地安全归档]</span>';
        }
        
        let pathTip = '';
        if (activeMode === 'json' && item.autoCategory) {
            pathTip = `<div style="font-size:11px; color:var(--dim);">自动归档目录: 139cas/${item.autoCategory}</div>`;
        }

        row.innerHTML = '<div class="name">' + esc(item.file.name) + badge + pathTip + '</div>' +
                        '<div class="size">' + fmt(item.file.size) + '</div>' +
                        '<div class="st ' + item.state + '">' + stText + prog + '</div>';
        $('flist').appendChild(row);
    }
}

// ==== 执行统一中控 ====
async function executeTasks() {
    if (isRunning) return;
    
    const ct = $('cloudType').value;
    
    if (activeMode === 'local' || activeMode === 'json') {
        if (localQueue.length === 0) {
            alert("请先添加文件或解析 JSON 到列表！");
            return;
        }
        const pendingTasks = localQueue.filter(q => q.state === 'wait' || q.state === 'fail');
        if (pendingTasks.length === 0) {
            alert("所有文件已处理完毕！");
            return;
        }
        
        isRunning = true;
        $('btnGo').disabled = true;
        $('btnClear').disabled = true;
        
        for (let i = 0; i < localQueue.length; i++) {
            const item = localQueue[i];
            if (item.state === 'done') continue;
            
            item.state = 'work';
            item.progress = 0;
            renderLocalList();
            
            try {
                let casPayload = {};
                
                // 如果是 JSON 拆包模式，不需要计算，直接用现成的 casObj
                if (activeMode === 'json') {
                    casPayload = item.casObj;
                    item.progress = 1;
                } else {
                    // --- 139 移动云盘逻辑 ---
                    if (ct === '139') {
                        const shaAlgo = sha256.create();
                        let off = 0;
                        while (off < item.file.size) {
                            const end = Math.min(off + CHUNK_SIZE, item.file.size);
                            const buf = await item.file.slice(off, end).arrayBuffer();
                            shaAlgo.update(buf);
                            off = end;
                            item.progress = off / item.file.size;
                            renderLocalList();
                            await new Promise(r => setTimeout(r, 0));
                        }
                        casPayload = { 
                            name: item.file.name, 
                            size: item.file.size, 
                            md5: "", 
                            sliceMd5: "", 
                            sha256: shaAlgo.hex().toLowerCase(), 
                            create_time: String(Math.floor(Date.now() / 1000)) 
                        };
                    } 
                    // --- 189 天翼云盘逻辑 ---
                    else {
                        const hashes = await readLocalFileMd5(item.file, function(p) {
                            item.progress = p;
                            renderLocalList();
                        });
                        casPayload = { 
                            name: item.file.name, 
                            size: item.file.size, 
                            md5: hashes.md5, 
                            sliceMd5: hashes.sliceMd5, 
                            create_time: String(Math.floor(Date.now() / 1000)), 
                            cloud: '189' 
                        };
                    }
                }
                
                const b64Str = btoa(unescape(encodeURIComponent(JSON.stringify(casPayload))));
                
                // 决定落盘目录：如果是 json 模式，使用自带的 autoCategory，否则使用 UI 填写的 categoryInput
                const finalCategory = (activeMode === 'json' && item.autoCategory) ? item.autoCategory : $('categoryInput').value;

                // 向后端发起写入指令，传递 cloud_type 进行物理隔离
                const res = await fetch('/api/save_cas_only', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ 
                        cas_data: casPayload, 
                        b64: b64Str, 
                        category: finalCategory, 
                        cloud_type: (activeMode === 'json' ? '139' : ct) // json模式强制落盘为139
                    })
                });
                const data = await res.json();
                item.backendObj = data;
                item.casObj = casPayload; // 保存起来供展示
                item.state = 'done';
            } catch (e) {
                item.state = 'fail';
                console.error(e);
            }
            renderLocalList();
        }
        
        currentPayloads = localQueue.filter(q => q.state === 'done').map(q => q.casObj);
        
        isRunning = false;
        $('btnGo').disabled = false;
        $('btnClear').disabled = false;
        showOutput();

    } else if (activeMode === 'server') {
        const bp = $('serverBasePath').value.trim();
        const sp = $('serverSubPath').value.trim();
        if (!bp) {
            alert("服务器基础挂载路径不能为空！");
            return;
        }
        
        $('flist').innerHTML = '<div class="frow work">🖥️ 服务器核心正在扫描并流式解析目录内所有视频文件，大文件可能需要数十秒，请耐心等待...</div>';
        $('btnGo').disabled = true;
        isRunning = true;
        
        try {
            const res = await fetch('/api/process_server_folder', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ 
                    base_path: bp, 
                    sub_path: sp, 
                    specific_file: $('serverSpecificFile').value.trim(), 
                    category: $('categoryInput').value,
                    cloud_type: ct
                })
            });
            const data = await res.json();
            $('flist').innerHTML = '';
            
            if (data.error) {
                $('flist').innerHTML = '<div class="frow fail"><span class="name">❌ 服务器报错: ' + esc(data.error) + '</span></div>';
            } else if (data.results && data.results.length > 0) {
                for (let i = 0; i < data.results.length; i++) {
                    const r = data.results[i];
                    if (r.error) {
                        $('flist').innerHTML += '<div class="frow fail"><span class="name">❌ 解析失败: ' + esc(r.file_name) + ' - ' + esc(r.error) + '</span></div>';
                    } else {
                        $('flist').innerHTML += '<div class="frow done"><span class="name">✓ [服务器高速解析完成] ' + esc(r.cas_data.name) + ' (' + r.cas_data.part_size_used + ')</span><span style="color:var(--ok)">[已物理归档]</span></div>';
                    }
                }
                currentPayloads = data.results.filter(r => !r.error).map(r => r.cas_data);
                showOutput();
            } else {
                $('flist').innerHTML = '<div class="frow wait"><span class="name">⚠️ 目标目录下没有扫描到任何常见的视频文件。</span></div>';
            }
        } catch (e) {
            $('flist').innerHTML = '<div class="frow fail"><span class="name">❌ 与服务器通信异常</span></div>';
        }
        $('btnGo').disabled = false;
        isRunning = false;
    }
}

// ==== 浏览器纯前端哈希计算底层算法 (189 专用) ====
async function readLocalFileMd5(file, onProgress) {
    const fileSize = file.size;
    const currentSliceSize = getPartSize(fileSize);
    
    const fileSpark = new SparkMD5.ArrayBuffer();
    const sliceMd5Hexs = [];
    let off = 0;

    while (off < fileSize) {
        const sliceEnd = Math.min(off + currentSliceSize, fileSize);
        const sliceSpark = new SparkMD5.ArrayBuffer();
        let inner = off;
        while (inner < sliceEnd) {
            const end = Math.min(inner + CHUNK_SIZE, sliceEnd);
            const buf = await file.slice(inner, end).arrayBuffer();
            fileSpark.append(buf);
            sliceSpark.append(buf);
            inner = end;
            if (onProgress) onProgress(inner / fileSize);
            await new Promise(resolve => setTimeout(resolve, 0));
        }
        sliceMd5Hexs.push(sliceSpark.end().toUpperCase());
        off = sliceEnd;
    }

    const fullMd5 = fileSpark.end().toUpperCase();
    let sliceMd5 = fullMd5;
    if (fileSize > currentSliceSize) {
        // 原生换行符无损拼接
        sliceMd5 = SparkMD5.hash(sliceMd5Hexs.join('\n')).toUpperCase();
    }
    return { md5: fullMd5, sliceMd5: sliceMd5 };
}

// ==== 结果与输出面板控制 ====
function showOutput() {
    if (currentPayloads.length === 0) return;
    $('out').textContent = JSON.stringify(currentPayloads, null, 2);
    $('outCard').style.display = 'block';
    $('btnCopy').disabled = false;
    $('btnZip').disabled = false;
    
    if (currentPayloads.length > 1) {
        $('btnZip').textContent = '📥 下载 .cas 压缩包 (Base64)';
    } else {
        $('btnZip').textContent = '📥 下载 .cas 文件 (Base64)';
    }
}

function copyJson() {
    navigator.clipboard.writeText($('out').textContent);
    alert("明文 JSON 已成功复制到剪贴板！");
}

function downloadCasFile() {
    if (currentPayloads.length === 0) return;
    
    const files = [];
    for (let i = 0; i < currentPayloads.length; i++) {
        const q = currentPayloads[i];
        const b64Str = btoa(unescape(encodeURIComponent(JSON.stringify(q))));
        files.push({ name: q.name + '.cas', data: b64Str });
    }

    if (files.length === 1) {
        const blob = new Blob([files[0].data], { type: 'text/plain' });
        const a = document.createElement('a');
        a.href = URL.createObjectURL(blob);
        a.download = files[0].name;
        a.click();
        URL.revokeObjectURL(a.href);
        return;
    }

    const zip = new JSZip();
    for (let i = 0; i < files.length; i++) {
        zip.file(files[i].name, files[i].data);
    }
    zip.generateAsync({ type: 'blob', compression: 'STORE' }).then(function(blob) {
        const a = document.createElement('a');
        a.href = URL.createObjectURL(blob);
        
        // 文件名动态跟随所选云盘
        const ct = (activeMode === 'json') ? '139' : $('cloudType').value;
        a.download = 'cas_' + ct + '_' + new Date().getTime() + '.zip';
        
        a.click();
        URL.revokeObjectURL(a.href);
    });
}

function resetForm() {
    if (isRunning) return;
    $('flist').innerHTML = '';
    $('outCard').style.display = 'none';
    $('btnCopy').disabled = true;
    $('btnZip').disabled = true;
    localQueue = [];
    currentPayloads = [];
    
    $('catName').value = ''; 
    $('catYear').value = ''; 
    $('catSeason').value = '';
    
    if (activeMode === 'server') {
        $('serverSubPath').value = '';
        $('serverSpecificFile').value = '';
    } else if (activeMode === 'local') {
        $('finput').value = '';
    } else {
        $('fjson').value = '';
    }
    buildPath();
}

// 首次启动时确保状态全部强绑定
buildPath();
switchMode('local');
</script>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE)

# API 1：处理服务器本地媒体文件（增强版：支持文件夹批量遍历收割）
@app.route('/api/process_server_folder', methods=['POST'])
def api_process_server_folder():
    req = request.get_json()
    base_path = req.get('base_path', '').strip()
    sub_path = req.get('sub_path', '').strip()
    specific_file = req.get('specific_file', '').strip()
    category_path = req.get('category', 'default')
    cloud_type = req.get('cloud_type', '189')
    
    if not base_path:
        return jsonify({"error": "基础路径不可为空"}), 400
        
    full_target_dir = os.path.join(base_path, sub_path)
    
    # 建立即将处理的任务列表
    target_files = []
    
    if specific_file:
        # 如果指定了具体文件
        exact_file = os.path.join(full_target_dir, specific_file)
        if os.path.isfile(exact_file):
            target_files.append(exact_file)
        else:
            return jsonify({"error": f"找不到指定的具体文件: {exact_file}"}), 400
    else:
        # 如果只指派了文件夹，遍历文件夹下的所有视频文件
        if not os.path.isdir(full_target_dir):
            return jsonify({"error": f"目标文件夹不存在: {full_target_dir}"}), 400
            
        for f in os.listdir(full_target_dir):
            f_path = os.path.join(full_target_dir, f)
            if os.path.isfile(f_path) and f.lower().endswith(VIDEO_EXTENSIONS):
                target_files.append(f_path)
                
    if not target_files:
        return jsonify({"results": []})
        
    # 根据云盘类型获取对应的基础落盘路径
    base_out_dir = CAS_BASE_DIRS.get(cloud_type, CAS_BASE_DIRS["189"])
    category_dir = os.path.join(base_out_dir, category_path)
    os.makedirs(category_dir, exist_ok=True)
    
    results = []
    for f_path in target_files:
        try:
            cas_data = calculate_single_file_cas(f_path, cloud_type)
            if cas_data:
                # 转换 Base64 落盘
                json_str = json.dumps(cas_data, ensure_ascii=False)
                b64_str = base64.b64encode(json_str.encode('utf-8')).decode('utf-8')
                
                saved_cas_path = os.path.join(category_dir, f"{cas_data['name']}.cas")
                with open(saved_cas_path, 'w', encoding='utf-8') as fs:
                    fs.write(b64_str)
                    
                results.append({
                    "file_name": cas_data['name'],
                    "cas_data": cas_data,
                    "saved_path": saved_cas_path
                })
        except Exception as e:
            results.append({"file_name": os.path.basename(f_path), "error": str(e)})
            
    return jsonify({"results": results})


# API 2：保存前端传来的 CAS 特征（本地浏览器跨网络模式 & JSON 拆包模式共用）
@app.route('/api/save_cas_only', methods=['POST'])
def api_save_cas_only():
    req = request.get_json()
    cas_data = req.get('cas_data', {})
    b64_str = req.get('b64', '')
    category_path = req.get('category', 'default')
    cloud_type = req.get('cloud_type', '189')
    
    try:
        file_name = cas_data.get('name', 'unknown')
        # 根据云盘类型获取对应的基础落盘路径
        base_out_dir = CAS_BASE_DIRS.get(cloud_type, CAS_BASE_DIRS["189"])
        category_dir = os.path.join(base_out_dir, category_path)
        os.makedirs(category_dir, exist_ok=True)
        
        saved_cas_path = os.path.join(category_dir, f"{file_name}.cas")
        with open(saved_cas_path, 'w', encoding='utf-8') as f:
            f.write(b64_str)
        return jsonify({"status": "success"})
    except Exception as e:
        return jsonify({"error": str(e)}), 500

# ==========================================
# 命令行 (CLI) 入口配置
# ==========================================
if __name__ == '__main__':
    parser = argparse.ArgumentParser(description="双擎云盘 CAS 控制中心")
    parser.add_argument('--cli', action='store_true', help='启用纯命令行模式（不在后台启动Web服务器）')
    parser.add_argument('--file', type=str, help='媒体文件的绝对路径（必须与 --cli 配合使用）')
    parser.add_argument('--cloud', type=str, default='139', choices=['189', '139'], help='目标云盘类型 (189 默认 或 139)')
    parser.add_argument('--category', type=str, default='default', help='落盘分类路径 (如 华语剧/剧名/Season 1)')
    
    args = parser.parse_args()

    # 当 TG 脚本等外部程序传入 --cli 参数时，不启动 Web 服务，直接纯命令流式运算
    if args.cli:
        if not args.file or not os.path.exists(args.file):
            print(f"❌ 错误: 找不到指定的物理文件 -> {args.file}")
            sys.exit(1)
            
        print(f"\n🚀 开始纯命令行流式计算 CAS (目标云盘: {args.cloud})")
        print(f"📄 目标文件: {args.file}")
        
        cas_data = calculate_single_file_cas(args.file, args.cloud)
        
        if cas_data:
            json_str = json.dumps(cas_data, ensure_ascii=False)
            b64_str = base64.b64encode(json_str.encode('utf-8')).decode('utf-8')
            
            # 获取基础输出目录并建立完整路径
            base_out_dir = CAS_BASE_DIRS.get(args.cloud, CAS_BASE_DIRS["189"])
            category_dir = os.path.join(base_out_dir, args.category)
            os.makedirs(category_dir, exist_ok=True)
            
            saved_cas_path = os.path.join(category_dir, f"{cas_data['name']}.cas")
            with open(saved_cas_path, 'w', encoding='utf-8') as f:
                f.write(b64_str)
                
            print("\n✨ CAS 锻造闭环完成 ✨")
            print(f"📦 物理文件已安全归档至: {saved_cas_path}")
        else:
            print("\n❌ 计算异常，未能生成特征负载。")
            sys.exit(1)
    else:
        # 默认模式：正常启动网页服务端
        print("启动 Web 远程管理平台 (端口 5050)...")
        app.run(host='0.0.0.0', port=5050)
```