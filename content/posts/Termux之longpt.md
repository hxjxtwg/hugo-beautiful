---

title: "Termux之longpt"

author: "xxsky"

type: "posts"

date: 2026-09-08T16:59:48+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

#### 一、longpt签到脚本

<!--more-->
1.抓取Cookie

2.longpt.py

```
import requests
import time

# 如果走 Cookie 路线
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Cookie': '这里替换成你抓取到的完整 Cookie 字符串'
}

# 签到接口地址，根据你 F12 抓到的实际地址填
SIGN_URL = "https://longpt.org/attendance.php"

def pt_signin():
    try:
        # 根据实际抓包情况选择 get 或 post
        response = requests.get(SIGN_URL, headers=HEADERS, timeout=10)
        response.raise_for_status()
        
        # 简单判断是否签到成功，具体关键字看返回的 HTML 内容
        if "签到成功" in response.text or "已签到" in response.text:
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] 签到成功！")
        else:
            print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] 状态未知，返回内容：{response.text[:100]}")
    except Exception as e:
        print(f"签到出错: {e}")

if __name__ == "__main__":
    pt_signin()
```
3.PM2启动管理

```
cd ~/189py && pm2 start longpt.py --name "longpt" --cron-restart="0 9 * * *" --no-autorestart
```
每天上午 9 点会自动运行：你在命令里写的 --cron-restart="0 9 * * *" 就是标准的 Cron 表达式，代表每天的 09:00:00 触发。它会按照你运行 Termux 设备的本地系统时间来准时执行

关于执行状态的补充说明：你带上了 --no-autorestart 这个参数，逻辑非常完美。因为这个签到脚本属于“跑完就结束”的一次性任务，并非需要长期驻留后台的服务。

执行完这次签到后，这个进程稍后在 PM2 里的状态会自动变成 stopped。

不用担心它停了，这属于正常现象。到了明天上午 9 点整，PM2 会按照 Cron 规则自动把它重新唤醒拉起，执行完毕后再次停止，每天周而复始。

你可以放心地把它挂在后台了。如果你后续想确认它到底有没有按时打卡，随时可以用 pm2 logs longpt 查看未来的签到日志

#### 二、H5网页端
1.安装强大的 HTML 解析库 BeautifulSoup：
```
pip install beautifulsoup4
```
2.安装 Flask 框架
```
pip install flask
```
3.pm2管理
启动管理
```
cd 189py && pm2 start h5long.py --name "h5long"
```
定时签到
```
pm2 start "curl http://127.0.0.1:5051/api/signin" --name "as-long" --cron-restart="0 10 * * *" --no-autorestart
```
4.h5long.py
```
from flask import Flask, render_template_string, Response, request, jsonify
import requests
from bs4 import BeautifulSoup
import urllib.parse
import re

app = Flask(__name__)

# ================= 必须配置的区域 =================
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Cookie': 'SITE_TOTAL_ID=b06a69304be7b561a043e0a5854d8e72; c_secure_pass=eyJ1c2VyX2lkIjoiMzQyODMiLCJleHBpcmVzIjoxODEzNDk1MjQyfS5lZjAyNTU4NDlkMzlkYjZhNzM4NmM5NTUzZTNlMWFjM2Q4M2VlYjE3NTUwNjI1ZjE3NjRkMzAwNjlmZTlmM2Zj; server_name_session=8d0f548489f4150c8755cd0497ba1583; XSRF-TOKEN=eyJpdiI6IjZjTVpMTkc0dmRWQk5hbzdEeTJGZVE9PSIsInZhbHVlIjoiTXEycXdQK3cxSVhCbGRLN01sM0lWUzRlb1k3MExsUzJXU082NkZzWFNCUE1TQkJzLytLR3dXTG5IczI1Zm9CWDVTN3c2dXdXL0d3YWxuQ21hZWlyN0Q3V1RFeEgwcG5SZFAwUEppaVd6Njg5YnZ2SEtaa2ZSYUxuWGZoaEJmbGQiLCJtYWMiOiJlNWY5ZGM0MmIxMzhkNzFlMWZhZmQyMGFkZWQwNDNlMjQ0N2EyZTlhZTY3MTVlZjc3MWQ3YTM1ZGExNWNhYmY1IiwidGFnIjoiIn0%3D; nexusphp_session=eyJpdiI6InV6d29tOTlJTE5pU2NmUzU2RDQwWWc9PSIsInZhbHVlIjoiVXc1dUVncTI5RVRCU2dUL3c3bS91OTU4ajN0blFOVHNhRHBsY2swUlNIMnduNVRqTW1NVjBlaHVUa3VyZnFXRC9oNGwxdGlyM0NFdCtkUkVXVm9DdlBuSlFWOStuNE9yeFhOelBtS3lad0VZQzg5ekxVWEE4MXNPRFc3ZHIvYnciLCJtYWMiOiJmZDE5MGQ3Nzk3MjlmZjcwY2Q4YTU3NWI1ZGE0YjQ0ZTRiMmY1ODM3OWYzNGNlZTYxMGNlNTA3NmU0M2QzNmE1IiwidGFnIjoiIn0%3D; cf_clearance=vFfWiQs3D8QlbbS5.Y8gP.qI9eg_zkm3NDUFKMFRFkM-1789555622-1.2.1.1-XUcX7UQU1ZLQo_GsH57MbFmsC29A0klq_4K_D4EQmj_q1HY0oMVdS16NDkVBUuBzKPozx0BVCDvulNXSE25EgEXeXVIBv._FIRIatr0IjUdj0Nzy7Rs67Pt2Pg7Z_rmBNfZFuwz.GBb_UYVy0SWkOn7amlHl.ZXKc26K_P3ABYeHfV4LJaqKdgBtTiAY9Vz.xuWD7N48sxr08kvKMTppmmlh8YHoc9wyEwphpla_GQ0K4tDTtD55v.ew5e.4BI.TD5Gp7Bwh3yY9o1BUJzULAqeLFZXYduYWLuShQ.2NSrYNHvRyK85j5vuYoWrOflx4dJW.15_4Ns421lyItYWmJU5ZH0DYS.VUW0sLE8umB_5lH2wMxNk6ABq3oEMxX9yhOkxuFLrwYNHu756bR5FycUaEOWgcOJXUdqN20oKw3OgZk_kUtbMgGSdLIijF.p9h' 
}

OFFICIAL_URL = "https://longpt.org/torrents.php"
THIRD_PARTY_URL = "https://longpt.org/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search=&search_area=0&search_mode=0&author="
INDEX_URL = "https://longpt.org/index.php"
SIGNIN_URL = "https://longpt.org/attendance.php"
# =================================================

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>LongPT 手机端</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background-color: #f2f2f6; margin: 0; padding: 0; color: #333; padding-top: 55px; }
        
        .nav-bar { position: fixed; top: 0; left: 0; right: 0; height: 45px; background: #fff; display: flex; box-shadow: 0 1px 3px rgba(0,0,0,0.1); z-index: 100; }
        .nav-item { flex: 1; text-align: center; line-height: 45px; font-size: 13px; font-weight: bold; color: #666; text-decoration: none; position: relative; }
        .nav-item.active { color: #007aff; }
        .nav-item.active::after { content: ''; position: absolute; bottom: 0; left: 15%; right: 15%; height: 3px; background: #007aff; border-radius: 3px 3px 0 0; }
        
        .container { padding: 10px; }
        .card { background: #fff; border-radius: 10px; padding: 10px; margin-bottom: 12px; box-shadow: 0 2px 5px rgba(0,0,0,0.05); }
        .content { display: flex; gap: 10px; }
        .poster-wrapper { flex-shrink: 0; width: 85px; }
        .poster { width: 100%; height: 120px; object-fit: cover; border-radius: 6px; background-color: #e0e0e0; }
        .info { flex-grow: 1; display: flex; flex-direction: column; justify-content: space-between; overflow: hidden; }
        .text-wrap { margin-bottom: 5px; }
        .title { font-size: 14px; font-weight: bold; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; line-height: 1.3; }
        .subtitle { font-size: 11px; color: #666; line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 3; -webkit-box-orient: vertical; overflow: hidden; }
        
        .torrent-stats { display: flex; gap: 12px; font-size: 12px; font-weight: 600; margin-bottom: 8px; background: #f8f9fa; padding: 4px 8px; border-radius: 4px; }
        .stat-seed { color: #34c759; }
        .stat-leech { color: #ff3b30; }
        .stat-size { color: #007aff; margin-left: auto; }

        .actions { display: flex; gap: 8px; margin-top: auto; }
        .btn { flex: 1; text-align: center; padding: 6px 0; border-radius: 4px; text-decoration: none; font-size: 12px; font-weight: 500; cursor: pointer; border: none; }
        .btn-detail { background: #f0f0f5; color: #333; }
        .btn-download { background: #007aff; color: #fff; }
        
        .btn-signin { background: #34c759; color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 20px; transition: opacity 0.2s; }
        .btn-signin:active { opacity: 0.7; }

        .account-header { text-align: center; padding: 20px 0 10px; }
        .avatar { width: 70px; height: 70px; background: #007aff; color: #fff; border-radius: 35px; line-height: 70px; font-size: 24px; margin: 0 auto 10px; }
        .account-name { font-size: 18px; font-weight: bold; }
        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; margin-top: 15px; }
        .stat-box { background: #f9f9f9; padding: 12px; border-radius: 8px; text-align: center; }
        .stat-val { font-size: 16px; font-weight: bold; color: #007aff; margin-bottom: 4px; }
        .stat-lbl { font-size: 11px; color: #666; }

        /* 新增：搜索框样式 */
        .search-container { display: flex; gap: 8px; margin-bottom: 15px; }
        .search-input { flex: 1; padding: 10px 12px; border: 1px solid #ddd; border-radius: 6px; outline: none; font-size: 14px; background: #fff; }
        .search-btn { background: #007aff; color: #fff; border: none; padding: 0 15px; border-radius: 6px; font-weight: bold; cursor: pointer; }

        @media (prefers-color-scheme: dark) {
            body { background-color: #121212; color: #e0e0e0; }
            .nav-bar { background: #1e1e1e; border-bottom: 1px solid #333; }
            .nav-item { color: #999; }
            .nav-item.active { color: #4facfe; }
            .nav-item.active::after { background: #4facfe; }
            .card, .stat-box { background: #1e1e1e; box-shadow: 0 2px 5px rgba(0,0,0,0.3); }
            .subtitle, .stat-lbl { color: #999; }
            .torrent-stats { background: #2c2c2e; }
            .btn-detail { background: #333; color: #ccc; }
            .poster { background-color: #333; }
            .btn-signin { background: #30d158; }
            .search-input { background: #1e1e1e; color: #eee; border: 1px solid #444; }
        }
    </style>
</head>
<body>
    <div class="nav-bar">
        <a href="/?tab=official" class="nav-item {% if tab == 'official' %}active{% endif %}">官方发布</a>
        <a href="/?tab=third" class="nav-item {% if tab == 'third' %}active{% endif %}">最近更新</a>
        <a href="/?tab=search" class="nav-item {% if tab == 'search' %}active{% endif %}">全站搜索</a>
        <a href="/?tab=account" class="nav-item {% if tab == 'account' %}active{% endif %}">我的帐号</a>
    </div>

    <div class="container">
        <!-- 搜索框区域 -->
        {% if tab == 'search' %}
        <form class="search-container" action="/" method="GET">
            <input type="hidden" name="tab" value="search">
            <input type="text" name="q" class="search-input" placeholder="输入你想找的种子..." value="{{ query }}">
            <button type="submit" class="search-btn">🔍 搜索</button>
        </form>
        {% endif %}

        {% if error %}
            <div style="text-align:center; color:#ff4d4f; margin-top:30px; font-weight:bold;">{{ error }}</div>
        {% endif %}

        {% if tab == 'account' and user_info %}
        <div class="card">
            <div class="account-header">
                <div class="avatar">{{ user_info.username[:1] | upper }}</div>
                <div class="account-name">{{ user_info.username }}</div>
            </div>
            <div class="stat-grid">
                <div class="stat-box"><div class="stat-val">{{ user_info.bonus }}</div><div class="stat-lbl">魔力值</div></div>
                <div class="stat-box"><div class="stat-val">{{ user_info.ratio }}</div><div class="stat-lbl">分享率</div></div>
                <div class="stat-box"><div class="stat-val">{{ user_info.upload }}</div><div class="stat-lbl">总上传</div></div>
                <div class="stat-box"><div class="stat-val">{{ user_info.download }}</div><div class="stat-lbl">总下载</div></div>
            </div>
            <button id="signinBtn" class="btn btn-signin" onclick="doSignIn()">✅ 手动签到</button>
        </div>
        
        <script>
            function doSignIn() {
                const btn = document.getElementById('signinBtn');
                btn.innerText = '签到中...';
                btn.disabled = true;
                
                fetch('/api/signin')
                    .then(response => response.json())
                    .then(data => {
                        alert(data.msg);
                        btn.innerText = '✅ 手动签到';
                        btn.disabled = false;
                    })
                    .catch(err => {
                        alert('请求失败，请检查网络');
                        btn.innerText = '✅ 手动签到';
                        btn.disabled = false;
                    });
            }
        </script>
        
        {% elif tab != 'account' %}
            {% for item in torrents %}
            <div class="card">
                <div class="content">
                    <div class="poster-wrapper">
                        <img class="poster" src="{{ item.img }}" referrerpolicy="no-referrer" alt="海报">
                    </div>
                    <div class="info">
                        <div class="text-wrap">
                            <div class="title">{{ item.title }}</div>
                            <div class="subtitle">{{ item.subtitle }}</div>
                        </div>
                        
                        <div class="torrent-stats">
                            <span><span class="stat-seed">⬆️ 做种</span> {{ item.seeders }}</span>
                            <span><span class="stat-leech">⬇️ 下载</span> {{ item.leechers }}</span>
                            {% if item.size %}
                                <span class="stat-size">📦 {{ item.size }}</span>
                            {% endif %}
                        </div>

                        <div class="actions">
                            <a href="{{ item.link }}" target="_blank" class="btn btn-detail">查看详情</a>
                            <a href="/download/{{ item.id }}" class="btn btn-download">📥 下载种子</a>
                        </div>
                    </div>
                </div>
            </div>
            {% endfor %}
            
            {% if tab == 'search' and not torrents and query %}
                <div style="text-align:center; color:#999; margin-top:30px;">没有找到与 "{{ query }}" 相关的种子哦</div>
            {% endif %}
        {% endif %}
    </div>
</body>
</html>
"""

@app.route('/')
def index():
    tab = request.args.get('tab', 'official')
    query = request.args.get('q', '')
    torrents = []
    user_info = {}
    seen_ids = set() 
    error_msg = None
    
    try:
        if tab == 'account':
            resp = requests.get(INDEX_URL, headers=HEADERS, timeout=10)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效，请在代码中更新 Cookie！"
            else:
                soup = BeautifulSoup(resp.text, 'html.parser')
                page_text = soup.get_text()
                
                user_match = re.search(r'欢迎回来,\s*([^\[\s]+)', page_text)
                bonus_match = re.search(r'魔力值.*?:\s*([\d\.,]+)', page_text)
                ratio_match = re.search(r'分享率:\s*([\d\.,]+)', page_text)
                up_match = re.search(r'上传量:\s*([\d\.,\s]+[KMGTP]B)', page_text)
                down_match = re.search(r'下载量:\s*([\d\.,\s]+[KMGTP]B)', page_text)
                
                user_info = {
                    'username': user_match.group(1) if user_match else "未知用户",
                    'bonus': bonus_match.group(1) if bonus_match else "0",
                    'ratio': ratio_match.group(1) if ratio_match else "0",
                    'upload': up_match.group(1) if up_match else "0 GB",
                    'download': down_match.group(1) if down_match else "0 GB"
                }

        # 官方列表与搜索共用同一套抓取逻辑（排版完全一致）
        elif tab in ['official', 'search']:
            if tab == 'search' and not query:
                # 仅处于搜索页但没关键字时，不发请求，直接展示空白页
                pass
            else:
                # 若为搜索状态，将关键字拼入参数发起全站搜索
                if tab == 'search':
                    safe_query = urllib.parse.quote(query)
                    target_url = f"https://longpt.org/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
                else:
                    target_url = OFFICIAL_URL
                    
                resp = requests.get(target_url, headers=HEADERS, timeout=10)
                if "登录" in resp.text and "torrents.php" not in resp.url:
                    error_msg = "⚠️ Cookie 已失效！"
                elif resp.status_code == 200:
                    soup = BeautifulSoup(resp.text, 'html.parser')
                    for row in soup.find_all('tr'):
                        tds = row.find_all('td')
                        if len(tds) < 6: continue
                        title_a = row.find('a', href=lambda h: h and h.startswith('details.php?id='))
                        if not title_a: continue
                        
                        tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                        title = title_a.get('title') or title_a.get_text(strip=True)
                        if not title and title_a.find('b'): title = title_a.find('b').get_text(strip=True)
                        if not title or tid in seen_ids: continue
                        seen_ids.add(tid)
                        
                        link = "https://longpt.org/" + title_a['href']
                        
                        try:
                            size = tds[-5].get_text(strip=True)
                            seeders = tds[-4].get_text(strip=True)
                            leechers = tds[-3].get_text(strip=True)
                        except:
                            size, seeders, leechers = "未知", "-", "-"

                        img_src = ""
                        for img in (title_a.find_parent('table') or row).find_all('img'):
                            src = img.get('data-src') or img.get('src') or ""
                            if src and 'category' not in src.lower() and 'cat' not in src.lower():
                                img_src = src if src.startswith('http') else "https://longpt.org/" + src
                                break
                                
                        subtitle = ""
                        td = title_a.find_parent('td')
                        if td:
                            for icon in td.find_all('img'):
                                alt = icon.get('alt') or icon.get('title')
                                if alt and icon.get('src', '') != img_src:
                                    icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                            subtitle = td.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                            subtitle = subtitle.replace('[ | ]', '').replace('[ | | ]', '')
                            subtitle = re.sub(r'\[(Sticky|poster|置顶)\]', '', subtitle, flags=re.IGNORECASE)
                            subtitle = re.sub(r'\[[\s\|]*\]|热门|剩余时间.*?(?:\||$)', '', subtitle)
                            subtitle = re.sub(r'\|(?:\s*\|)+', '|', subtitle).strip(' | ')
                            
                        torrents.append({
                            'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                            'seeders': seeders, 'leechers': leechers, 'size': size
                        })

        elif tab == 'third':
            resp = requests.get(INDEX_URL, headers=HEADERS, timeout=10)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效！"
            elif resp.status_code == 200:
                soup = BeautifulSoup(resp.text, 'html.parser')
                for title_a in soup.find_all('a', href=lambda h: h and h.startswith('details.php?id=')):
                    tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                    title = title_a.get('title') or title_a.get_text(strip=True)
                    if not title and title_a.find('img'):
                        title = title_a.find('img').get('alt') or title_a.find('img').get('title')
                    if not title or tid in seen_ids: continue
                    seen_ids.add(tid)
                    
                    link = "https://longpt.org/" + title_a['href']
                    container = title_a.find_parent('tr') or title_a.find_parent('div')
                    
                    img_src, subtitle = "", ""
                    seeders, leechers, size = "-", "-", ""

                    if container:
                        if container.name == 'tr':
                            tds = container.find_all('td')
                            if len(tds) >= 3:
                                seeders = tds[-2].get_text(strip=True)
                                leechers = tds[-1].get_text(strip=True)

                        for img in container.find_all('img'):
                            src = img.get('data-src') or img.get('src') or ""
                            if src and 'category' not in src.lower() and 'cat' not in src.lower():
                                img_src = src if src.startswith('http') else "https://longpt.org/" + src
                                break
                        for icon in container.find_all('img'):
                            alt = icon.get('alt') or icon.get('title')
                            if alt and icon.get('src', '') != img_src:
                                icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                        subtitle = container.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                        subtitle = subtitle.replace('[ | ]', '').replace('[ | | ]', '')
                        subtitle = re.sub(r'\[(Sticky|poster|置顶)\]', '', subtitle, flags=re.IGNORECASE)
                        subtitle = re.sub(r'\[[\s\|]*\]|热门|剩余时间.*?(?:\||$)', '', subtitle)
                        subtitle = re.sub(r'\|(?:\s*\|)+', '|', subtitle).strip(' | ')

                        if seeders != "-" and subtitle.endswith(leechers):
                            subtitle = subtitle.rsplit('|', 2)[0].strip(' | ')

                    if not img_src:
                        img_src = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='85' height='120'><rect width='100%' height='100%' fill='%23e6e6e6'/><text x='50%' y='50%' fill='%23999' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>暂无海报</text></svg>"

                    torrents.append({
                        'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                        'seeders': seeders, 'leechers': leechers, 'size': size
                    })

    except Exception as e:
        error_msg = f"服务器内部请求错误: {e}"
        
    return render_template_string(HTML_TEMPLATE, tab=tab, query=query, torrents=torrents, user_info=user_info, error=error_msg)

@app.route('/api/signin')
def api_signin():
    try:
        r = requests.get(SIGNIN_URL, headers=HEADERS, timeout=10)
        if "签到成功" in r.text or "这是您的第" in r.text or "已连续签到" in r.text:
            return jsonify({"msg": "🎉 签到成功！"})
        elif "已经" in r.text or "退下" in r.text:
            return jsonify({"msg": "☕ 今天已经签到过了，明天再来吧。"})
        elif "登录" in r.text:
            return jsonify({"msg": "⚠️ 签到失败：Cookie 已失效，请更新！"})
        else:
            return jsonify({"msg": "🤔 状态未知，可能已签到，请留意积分变化。"})
    except Exception as e:
        return jsonify({"msg": f"❌ 网络请求出错: {str(e)}"})

@app.route('/download/<int:tid>')
def download_torrent(tid):
    dl_url = f"https://longpt.org/download.php?id={tid}"
    try:
        r = requests.get(dl_url, headers=HEADERS, timeout=10)
        if "application/x-bittorrent" in r.headers.get('Content-Type', '') or "octet-stream" in r.headers.get('Content-Type', ''):
            return Response(
                r.content,
                mimetype="application/x-bittorrent",
                headers={"Content-Disposition": f"attachment;filename=LongPT_{tid}.torrent"}
            )
        else:
            return "下载失败，Cookie 可能失效或无权下载该种子。"
    except Exception as e:
        return f"请求下载出错: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5051)
```