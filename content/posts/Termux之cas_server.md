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
from flask import Flask, request, jsonify, render_template_string

app = Flask(__name__)

# ==========================================
# 核心配置区
# ==========================================
# Termux 服务器本地 CAS 文件暂存根目录
CAS_OUTPUT_DIR = "/storage/emulated/0/Download/189cas"
os.makedirs(CAS_OUTPUT_DIR, exist_ok=True)

# 28GB 阈值（字节），超过此大小自动切换为 20MB 分片
LARGE_FILE_THRESHOLD = 26 * 1024 * 1024 * 1024  

# 支持批量遍历收割的视频扩展名
VIDEO_EXTENSIONS = ('.mp4', '.mkv', '.avi', '.mov', '.flv', '.wmv', '.rmvb', '.ts')

def get_dynamic_slice_size(file_size):
    """根据文件大小动态计算分片大小（防止大文件片数超 3000 爆限制）"""
    if file_size > LARGE_FILE_THRESHOLD:
        return 20 * 1024 * 1024  # >26G 自动启用 20MB 分片
    return 10 * 1024 * 1024      # 普通文件使用 10MB 分片

def calculate_single_file_cas(file_path):
    """核心底层算法：全速解构单个物理文件并计算标准级联哈希"""
    if not os.path.exists(file_path) or os.path.isdir(file_path):
        return None

    file_size = os.path.getsize(file_path)
    file_name = os.path.basename(file_path)
    
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
<title>天翼云盘 CAS 远程集成控制中心</title>
<script src="https://cdn.jsdelivr.net/npm/jszip@3.10.1/dist/jszip.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/spark-md5@3.0.2/spark-md5.min.js"></script>
<style>
* { margin: 0; padding: 0; box-sizing: border-box; }
:root { --bg: #0f172a; --card: #1e293b; --border: #334155; --text: #e2e8f0; --dim: #94a3b8; --accent: #3b82f6; --accent-h: #2563eb; --ok: #22c55e; --err: #ef4444; }
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
.mode-tabs { display: flex; gap: 10px; margin-bottom: 15px; border-bottom: 1px solid var(--border); padding-bottom: 10px; }
.tab { padding: 8px 16px; border-radius: 6px; font-size: 13px; cursor: pointer; background: var(--border); color: var(--dim); border: none; font-weight: bold; transition: all 0.2s; }
.tab:hover { background: #475569; }
.tab.active { background: var(--accent); color: #fff; }

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
  <h1>⚡ 天翼CAS服务器控制中心</h1>
  <p class="sub">支持大文件（>28G）自动切换 20M 分片逻辑 · 完美保留明文 JSON 结构与 Base64 统一输出</p>

  <!-- 1. Emby 路径配置 -->
  <div class="card">
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
      <label>即将落盘的物理相对目录：</label>
      <div class="final-path-text" id="finalPathDisplay"></div>
      <input type="hidden" id="categoryInput">
    </div>
  </div>

  <!-- 2. 操作模式切换与执行域 -->
  <div class="card">
    <div class="card-t">⚙️ 2. 选择工作引擎与提取源</div>
    <div class="mode-tabs">
      <button class="tab active" id="tabLocal" onclick="switchMode('local')">🌐 跨网络计算 </button>
      <button class="tab" id="tabServer" onclick="switchMode('server')">🖥️ 服务器本地 </button>
    </div>

    <!-- 模式 A：浏览器跨网络选择计算 -->
    <div id="panelLocal">
      <div class="drop" id="drop" onclick="document.getElementById('finput').click()">
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

    <!-- 任务队列面板 -->
    <div class="flist" id="flist"></div>
    
    <!-- 全局操作面板 -->
    <div class="btns">
      <button class="btn btn-g" id="btnGo" onclick="executeTasks()">🚀 计算&归档</button>
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

// 动态分片计算（对齐天翼云28G防爆机制）
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
    if (mode === 'server') {
        $('tabServer').classList.add('active');
        $('tabLocal').classList.remove('active');
        $('panelServer').style.display = 'block';
        $('panelLocal').style.display = 'none';
    } else {
        $('tabServer').classList.remove('active');
        $('tabLocal').classList.add('active');
        $('panelServer').style.display = 'none';
        $('panelLocal').style.display = 'block';
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
        
        $('finalPathDisplay').innerText = path + '/';
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
    return s.replace(/&/g,'&amp;').replace(/</g,'&lt;').replace(/>/g,'&gt;'); 
}

// 拖拽组件事件绑定
const dropEl = $('drop');
dropEl.addEventListener('dragover', function(e) { 
    e.preventDefault(); 
    dropEl.classList.add('over'); 
});
dropEl.addEventListener('dragleave', function() { 
    dropEl.classList.remove('over'); 
});
dropEl.addEventListener('drop', function(e) {
    e.preventDefault(); 
    dropEl.classList.remove('over');
    if (e.dataTransfer.files.length) {
        handleLocalFiles({ target: { files: e.dataTransfer.files } });
    }
});

function handleLocalFiles(e) {
    const files = e.target.files;
    if (!files.length) return;
    
    smartExtract(files[0].name);
    
    for (let i = 0; i < files.length; i++) {
        localQueue.push({ file: files[i], state: 'wait', progress: 0, md5: '', sliceMd5: '', backendObj: null });
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
        else if (item.state === 'work') stText = '⚙️计算中';
        else if (item.state === 'done') stText = '✓完成';
        else stText = '✗失败';
        
        const prog = (item.state === 'work' && item.progress > 0) ? ' ' + Math.round(item.progress * 100) + '%' : '';
        
        let badge = '';
        if (item.backendObj) {
            badge = '<span style="color:var(--ok); font-size:12px; margin-left:8px;">[本地安全归档]</span>';
        }

        row.innerHTML = '<div class="name">' + esc(item.file.name) + badge + '</div>' +
                        '<div class="size">' + fmt(item.file.size) + '</div>' +
                        '<div class="st ' + item.state + '">' + stText + prog + '</div>';
        $('flist').appendChild(row);
    }
}

// ==== 执行统一中控 ====
async function executeTasks() {
    if (isRunning) return;
    
    if (activeMode === 'local') {
        if (localQueue.length === 0) {
            alert("请先添加视频文件到列表！");
            return;
        }
        const pendingTasks = localQueue.filter(q => q.state === 'wait' || q.state === 'fail');
        if (pendingTasks.length === 0) {
            alert("所有文件已计算完毕！");
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
                const hashes = await readLocalFileMd5(item.file, function(p) {
                    item.progress = p;
                    renderLocalList();
                });
                item.md5 = hashes.md5;
                item.sliceMd5 = hashes.sliceMd5;
                
                const casPayload = { name: item.file.name, size: item.file.size, md5: item.md5, sliceMd5: item.sliceMd5, create_time: String(Math.floor(Date.now() / 1000)), cloud: '189' };
                const b64Str = btoa(unescape(encodeURIComponent(JSON.stringify(casPayload))));

                // 向后端发起写入指令
                const res = await fetch('/api/save_cas_only', {
                    method: 'POST',
                    headers: { 'Content-Type': 'application/json' },
                    body: JSON.stringify({ cas_data: casPayload, b64: b64Str, category: $('categoryInput').value })
                });
                const data = await res.json();
                item.backendObj = data;
                item.state = 'done';
            } catch (e) {
                item.state = 'fail';
                console.error(e);
            }
            renderLocalList();
        }
        
        currentPayloads = localQueue.filter(q => q.state === 'done').map(q => ({
            name: q.file.name, size: q.file.size, md5: q.md5, sliceMd5: q.sliceMd5, create_time: String(Math.floor(Date.now() / 1000)), cloud: '189'
        }));
        
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
                    category: $('categoryInput').value 
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
                        $('flist').innerHTML += '<div class="frow done"><span class="name">✓ [服务器高速解析完成] ' + esc(r.cas_data.name) + ' (单片:' + r.cas_data.part_size_used + ')</span><span style="color:var(--ok)">[已物理归档]</span></div>';
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

// ==== 浏览器纯前端哈希计算底层算法 (精准保留\n拼接) ====
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
        a.download = 'cas_files_' + new Date().getTime() + '.zip';
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
    
    // 不重置基础类别，只清除名称和季数
    $('catName').value = ''; 
    $('catYear').value = ''; 
    $('catSeason').value = '';
    if (activeMode === 'server') {
        $('serverSubPath').value = '';
        $('serverSpecificFile').value = '';
    } else {
        $('finput').value = '';
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
        
    # 执行批量计算与归档
    category_dir = os.path.join(CAS_OUTPUT_DIR, category_path)
    os.makedirs(category_dir, exist_ok=True)
    
    results = []
    for f_path in target_files:
        try:
            cas_data = calculate_single_file_cas(f_path)
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


# API 2：保存前端传来的 CAS 特征（本地浏览器跨网络模式）
@app.route('/api/save_cas_only', methods=['POST'])
def api_save_cas_only():
    req = request.get_json()
    cas_data = req.get('cas_data', {})
    b64_str = req.get('b64', '')
    category_path = req.get('category', 'default')
    
    try:
        file_name = cas_data.get('name', 'unknown')
        category_dir = os.path.join(CAS_OUTPUT_DIR, category_path)
        os.makedirs(category_dir, exist_ok=True)
        
        saved_cas_path = os.path.join(category_dir, f"{file_name}.cas")
        with open(saved_cas_path, 'w', encoding='utf-8') as f:
            f.write(b64_str)
        return jsonify({"status": "success"})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


if __name__ == '__main__':
    # 允许局域网远程操控
    app.run(host='0.0.0.0', port=5050)
```