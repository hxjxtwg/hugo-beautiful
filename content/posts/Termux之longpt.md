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
import os
import json
import threading
import time

app = Flask(__name__)

# ================= 必须配置的区域 =================
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    'Cookie': 'SITE_TOTAL_ID=b06a69304be7b561a043e0a5854d8e72; c_secure_pass=eyJ1c2VyX2lkIjoiMzQyODMiLCJleHBpcmVzIjoxODEzNDk1MjQyfS5lZjAyNTU4NDlkMzlkYjZhNzM4NmM5NTUzZTNlMWFjM2Q4M2VlYjE3NTUwNjI1ZjE3NjRkMzAwNjlmZTlmM2Zj; server_name_session=8d0f548489f4150c8755cd0497ba1583; XSRF-TOKEN=eyJpdiI6IjZjTVpMTkc0dmRWQk5hbzdEeTJGZVE9PSIsInZhbHVlIjoiTXEycXdQK3cxSVhCbGRLN01sM0lWUzRlb1k3MExsUzJXU082NkZzWFNCUE1TQkJzLytLR3dXTG5IczI1Zm9CWDVTN3c2dXdXL0d3YWxuQ21hZWlyN0Q3V1RFeEgwcG5SZFAwUEppaVd6Njg5YnZ2SEtaa2ZSYUxuWGZoaEJmbGQiLCJtYWMiOiJlNWY5ZGM0MmIxMzhkNzFlMWZhZmQyMGFkZWQwNDNlMjQ0N2EyZTlhZTY3MTVlZjc3MWQ3YTM1ZGExNWNhYmY1IiwidGFnIjoiIn0%3D; nexusphp_session=eyJpdiI6InV6d29tOTlJTE5pU2NmUzU2RDQwWWc9PSIsInZhbHVlIjoiVXc1dUVncTI5RVRCU2dUL3c3bS91OTU4ajN0blFOVHNhRHBsY2swUlNIMnduNVRqTW1NVjBlaHVUa3VyZnFXRC9oNGwxdGlyM0NFdCtkUkVXVm9DdlBuSlFWOStuNE9yeFhOelBtS3lad0VZQzg5ekxVWEE4MXNPRFc3ZHIvYnciLCJtYWMiOiJmZDE5MGQ3Nzk3MjlmZjcwY2Q4YTU3NWI1ZGE0YjQ0ZTRiMmY1ODM3OWYzNGNlZTYxMGNlNTA3NmU0M2QzNmE1IiwidGFnIjoiIn0%3D; cf_clearance=vFfWiQs3D8QlbbS5.Y8gP.qI9eg_zkm3NDUFKMFRFkM-1789555622-1.2.1.1-XUcX7UQU1ZLQo_GsH57MbFmsC29A0klq_4K_D4EQmj_q1HY0oMVdS16NDkVBUuBzKPozx0BVCDvulNXSE25EgEXeXVIBv._FIRIatr0IjUdj0Nzy7Rs67Pt2Pg7Z_rmBNfZFuwz.GBb_UYVy0SWkOn7amlHl.ZXKc26K_P3ABYeHfV4LJaqKdgBtTiAY9Vz.xuWD7N48sxr08kvKMTppmmlh8YHoc9wyEwphpla_GQ0K4tDTtD55v.ew5e.4BI.TD5Gp7Bwh3yY9o1BUJzULAqeLFZXYduYWLuShQ.2NSrYNHvRyK85j5vuYoWrOflx4dJW.15_4Ns421lyItYWmJU5ZH0DYS.VUW0sLE8umB_5lH2wMxNk6ABq3oEMxX9yhOkxuFLrwYNHu756bR5FycUaEOWgcOJXUdqN20oKw3OgZk_kUtbMgGSdLIijF.p9h' 
}

OFFICIAL_URL = "https://longpt.org/torrents.php"
INDEX_URL = "https://longpt.org/index.php"
SIGNIN_URL = "https://longpt.org/attendance.php"
HISTORY_FILE = "longpt_history.json"
# =================================================

# --- 核心引擎：深度抓取详情页海报、体积与所有促销标签 ---
def fetch_details_info(tid):
    try:
        r = requests.get(f"https://longpt.org/details.php?id={tid}", headers=HEADERS, timeout=10)
        soup = BeautifulSoup(r.text, 'html.parser')

        img_src = ""
        descr_div = soup.find('div', id='kdescr')
        if descr_div:
            img = descr_div.find('img')
            if img: img_src = img.get('data-src') or img.get('src') or ""
        if not img_src:
            for img in soup.find_all('img'):
                src = img.get('data-src') or img.get('src') or ""
                src_lower = src.lower()
                if src and not any(x in src_lower for x in ['category', 'smilies', 'icon', 'logo', 'avatar', 'rating', 'button', 'rule']):
                    img_src = src
                    break
        if img_src and not img_src.startswith('http'):
            img_src = "https://longpt.org/" + img_src

        size = "未知"
        size_match = re.search(r'大小.*?([\d\.]+\s*[KMGTP]B)', r.text, re.IGNORECASE)
        if size_match: size = size_match.group(1)

        extra_tags = []
        for img in soup.find_all('img'):
            alt = str(img.get('alt') or img.get('title') or '').strip()
            if alt.upper() in ['FREE', '免费']:
                if 'Free' not in extra_tags: extra_tags.append('Free')
            elif any(x in alt for x in ['50%', '2X', '30%', '热门']):
                if alt not in extra_tags: extra_tags.append(alt)
        
        page_text = soup.get_text(separator=' ')
        time_match = re.search(r'剩余时间[:：\s]*([\d]+天[\d]+[时分秒]+|[\d]+[天时分秒]+)', page_text)
        if time_match: extra_tags.append(f"剩余时间: {time_match.group(1)}")
            
        h1 = soup.find('h1')
        if h1 and 'Free' not in extra_tags:
            if '免费' in h1.get_text() or 'Free' in h1.get_text():
                extra_tags.append('Free')
        
        return img_src, size, extra_tags
    except Exception as e:
        return "", "未知", []

# --- 守护进程：后台静默刷新 30 条最近发布 ---
def background_monitor():
    while True:
        try:
            resp = requests.get(INDEX_URL, headers=HEADERS, timeout=10)
            if "登录" not in resp.text or "index.php" in resp.url:
                soup = BeautifulSoup(resp.text, 'html.parser')
                current_fetched = []
                
                for title_a in soup.find_all('a', href=lambda h: h and h.startswith('details.php?id=')):
                    tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                    title = title_a.get('title') or title_a.get_text(strip=True)
                    if not title and title_a.find('img'): title = title_a.find('img').get('alt') or title_a.find('img').get('title')
                    if not title: continue
                    
                    link = "https://longpt.org/" + title_a['href']
                    container = title_a.find_parent('tr') or title_a.find_parent('div')
                    img_src, subtitle, seeders, leechers, size = "", "", "-", "-", "加载中..."
                    
                    if container:
                        if container.name == 'tr':
                            tds = container.find_all('td')
                            if len(tds) >= 3:
                                seeders, leechers = tds[-2].get_text(strip=True), tds[-1].get_text(strip=True)

                        for img in container.find_all('img'):
                            src = img.get('data-src') or img.get('src') or ""
                            if src and 'category' not in src.lower() and 'cat' not in src.lower():
                                img_src = src if src.startswith('http') else "https://longpt.org/" + src
                                break
                        for icon in container.find_all('img'):
                            alt = icon.get('alt') or icon.get('title')
                            if alt and icon.get('src', '') != img_src: icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                        subtitle = container.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                        if seeders != "-" and subtitle.endswith(leechers): subtitle = subtitle.rsplit('|', 2)[0].strip(' | ')
                        
                        subtitle = re.sub(r'\[?(?:剩余时间)[:：]*\]?[\s\|]*([\d]+天[\d]+[时分秒]+|[\d]+[天时分秒]+)', r'[剩余时间: \1]', subtitle)
                        subtitle = subtitle.replace('[ | ]', '').replace('[ | | ]', '')
                            
                        tags, clean_sub = [], subtitle
                        for m in re.finditer(r'\[(.*?)\]', subtitle):
                            tag = m.group(1).strip(' |')
                            # 精准打击：只抓热门、折扣和时间做成徽章
                            if any(x in tag.upper() for x in ['热门', 'FREE', '免费', '%', '2X', '剩余']):
                                if len(tag) <= 18: tags.append(tag)
                                clean_sub = clean_sub.replace(m.group(0), '')
                            elif tag not in ['Sticky', 'poster', '置顶']:
                                # 其他参数（如国语、4K）脱去方括号，还给副标题
                                clean_sub = clean_sub.replace(m.group(0), tag)
                            else:
                                clean_sub = clean_sub.replace(m.group(0), '')
                        
                        clean_sub = re.sub(r'热门|免费', '', clean_sub)
                        subtitle = re.sub(r'\|(?:\s*\|)+', '|', clean_sub).strip(' |')

                    if not img_src: img_src = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23e6e6e6'/><text x='50%' y='50%' fill='%23999' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>加载海报</text></svg>"

                    current_fetched.append({
                        'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                        'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags
                    })

                history_data = []
                if os.path.exists(HISTORY_FILE):
                    try:
                        with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history_data = json.load(f)
                    except: pass
                    
                history_dict = {str(item['id']): item for item in history_data}
                
                for item in current_fetched:
                    tid_str = str(item['id'])
                    if tid_str not in history_dict or 'data:image/svg+xml' in history_dict[tid_str]['img'] or history_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_tags:
                            new_tags = item.get('tags', [])
                            for t in det_tags:
                                if t not in new_tags: new_tags.append(t)
                            item['tags'] = new_tags

                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        merged_tags = item.get('tags', [])
                        for t in cached.get('tags', []):
                            if t not in merged_tags: merged_tags.append(t)
                        item['tags'] = merged_tags
                        
                    history_dict[tid_str] = item

                merged = list(history_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(merged[:30], f, ensure_ascii=False, indent=2)
        except Exception as e: pass
        time.sleep(1200)

threading.Thread(target=background_monitor, daemon=True).start()

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>LongPT 手机端</title>
    <style>
        :root { --bg-color: #f7f7f7; --card-bg: #ffffff; --text-main: #333333; --text-sub: #888888; --border-color: #eeeeee; --topbar-bg: #ffffff; --icon-bg: #f0f0f5; }
        [data-theme="dark"] { --bg-color: #121212; --card-bg: #1e1e1e; --text-main: #eeeeee; --text-sub: #aaaaaa; --border-color: #333333; --topbar-bg: #1e1e1e; --icon-bg: #333333; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background-color: var(--bg-color); color: var(--text-main); margin: 0; padding: 0; padding-top: 55px; transition: background-color 0.3s; }
        
        .top-bar { position: fixed; top: 0; left: 0; right: 0; height: 50px; background: var(--topbar-bg); display: flex; align-items: center; justify-content: space-between; padding: 0 15px; box-shadow: 0 1px 3px rgba(0,0,0,0.05); z-index: 100; transition: background 0.3s; }
        
        /* 修改点1：更精细轻盈的菜单图标 */
        .menu-btn { width: 22px; height: 16px; position: relative; cursor: pointer; display: flex; flex-direction: column; justify-content: space-between; padding: 0; }
        .menu-btn span { display: block; height: 2px; width: 100%; background: var(--text-main); border-radius: 2px; transition: all 0.3s ease; }
        .menu-btn.open span:nth-child(1) { transform: translateY(7px) rotate(45deg); }
        .menu-btn.open span:nth-child(2) { opacity: 0; }
        .menu-btn.open span:nth-child(3) { transform: translateY(-7px) rotate(-45deg); }

        .logo { font-weight: 800; font-size: 20px; color: var(--text-main); letter-spacing: 1px; }
        .top-placeholder { width: 22px; }

        .sidebar { position: fixed; top: 0; left: -260px; bottom: 0; width: 260px; background: var(--card-bg); z-index: 1001; transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: 2px 0 10px rgba(0,0,0,0.1); padding-top: 60px; display: flex; flex-direction: column; }
        .sidebar.open { left: 0; }
        .menu-item { padding: 16px 24px; font-size: 16px; font-weight: 500; color: var(--text-main); text-decoration: none; display: block; border-bottom: 1px solid var(--border-color); }
        .menu-item.active { color: #007aff; border-left: 4px solid #007aff; padding-left: 20px; background: rgba(0,122,255,0.05); }
        
        .theme-toggle-icon { margin: auto auto 30px 24px; width: 40px; height: 40px; border-radius: 20px; background: var(--icon-bg); display: flex; align-items: center; justify-content: center; font-size: 20px; cursor: pointer; user-select: none; transition: transform 0.2s; }
        .theme-toggle-icon:active { transform: scale(0.9); }
        
        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 1000; opacity: 0; pointer-events: none; transition: opacity 0.3s; }
        .overlay.open { opacity: 1; pointer-events: auto; }

        .container { padding: 10px; }
        .search-container { display: flex; gap: 8px; margin-bottom: 15px; padding: 0 5px; }
        .search-input { flex: 1; padding: 12px; border: 1px solid var(--border-color); border-radius: 6px; outline: none; font-size: 14px; background: var(--card-bg); color: var(--text-main); }
        .search-btn { background: #2563eb; color: #fff; border: none; padding: 0 18px; border-radius: 6px; font-weight: bold; font-size: 14px; }
        .list-item { display: flex; gap: 12px; padding: 15px 5px; border-bottom: 1px solid var(--border-color); }
        .poster-wrapper { position: relative; flex-shrink: 0; width: 100px; height: 140px; }
        .poster { width: 100%; height: 100%; object-fit: cover; border-radius: 8px; background-color: #e0e0e0; transition: opacity 0.5s ease-in-out; }
        .tag-free { position: absolute; top: 0; left: 0; background: #22c55e; color: #fff; font-size: 10px; font-weight: bold; padding: 2px 6px; border-radius: 8px 0 8px 0; z-index: 10; }
        .info { flex-grow: 1; display: flex; flex-direction: column; overflow: hidden; justify-content: space-between; padding-bottom: 2px; }
        .title { font-size: 15px; font-weight: bold; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; line-height: 1.3; }
        .subtitle { font-size: 12px; color: var(--text-sub); line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; margin-bottom: 8px; }
        .stats-row { font-size: 12px; color: var(--text-sub); display: flex; align-items: center; gap: 10px; margin-bottom: 8px; flex-wrap: wrap; }
        .stat-item { display: flex; align-items: center; gap: 2px; }
        .seed-icon { color: #22c55e; font-size: 10px; }
        .leech-icon { color: #ef4444; font-size: 10px; }
        
        /* 修改点3：防止标签换行挤压下载按钮 */
        .bottom-actions { display: flex; justify-content: space-between; align-items: flex-end; margin-top: auto; gap: 10px; }
        .tags-list { display: flex; gap: 4px; flex-wrap: wrap; flex: 1; }
        .badge { background: #8b5cf6; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; }
        .badge-discount { background: #ef4444; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; }
        
        .download-btn { background: #2563eb; color: #fff; border: none; padding: 4px 12px; border-radius: 4px; font-size: 11px; font-weight: bold; text-decoration: none; flex-shrink: 0; white-space: nowrap; margin-bottom: 2px; }
        .download-btn:active { opacity: 0.8; }
        
        .account-box { background: var(--card-bg); border-radius: 12px; padding: 20px; text-align: center; border: 1px solid var(--border-color); }
        .avatar { width: 70px; height: 70px; background: #2563eb; color: #fff; border-radius: 35px; line-height: 70px; font-size: 24px; margin: 0 auto 10px; }
        .account-name { font-size: 18px; font-weight: bold; }
        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }
        .stat-val { font-size: 18px; font-weight: bold; color: #2563eb; margin-bottom: 4px; }
        .stat-lbl { font-size: 12px; color: var(--text-sub); }
        .btn-signin { background: #22c55e; color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 25px; border: none; cursor: pointer; }
    </style>
</head>
<body>
    <script>
        function setTheme(theme) {
            const html = document.documentElement;
            if (theme === 'dark') { html.setAttribute('data-theme', 'dark'); }
            else if (theme === 'light') { html.removeAttribute('data-theme'); }
            else { 
                if (window.matchMedia('(prefers-color-scheme: dark)').matches) html.setAttribute('data-theme', 'dark');
                else html.removeAttribute('data-theme');
            }
        }
        let currentTheme = localStorage.getItem('longpt_theme') || 'auto';
        setTheme(currentTheme);
        window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
            if (localStorage.getItem('longpt_theme') === 'auto') setTheme('auto');
        });
    </script>

    <div class="top-bar">
        <div class="menu-btn" id="menuBtn" onclick="toggleMenu()">
            <span></span><span></span><span></span>
        </div>
        <div class="logo">LongPT</div>
        <div class="top-placeholder"></div>
    </div>

    <div class="overlay" id="overlay" onclick="toggleMenu()"></div>
    <div class="sidebar" id="sidebar">
        <a href="/?tab=official" class="menu-item {% if tab == 'official' %}active{% endif %}">官方发布</a>
        <a href="/?tab=recent" class="menu-item {% if tab == 'recent' %}active{% endif %}">最近发布</a>
        <a href="/?tab=search" class="menu-item {% if tab == 'search' %}active{% endif %}">全站搜索</a>
        <a href="/?tab=account" class="menu-item {% if tab == 'account' %}active{% endif %}">我的帐号</a>
        
        <div class="theme-toggle-icon" id="themeIcon" onclick="cycleTheme()">🔄</div>
    </div>

    <div class="container">
        {% if tab == 'search' %}
        <form class="search-container" action="/" method="GET">
            <input type="hidden" name="tab" value="search">
            <input type="text" name="q" class="search-input" placeholder="输入关键词搜索..." value="{{ query }}">
            <button type="submit" class="search-btn">搜索</button>
        </form>
        {% endif %}

        {% if error %}<div style="text-align:center; color:#ef4444; margin-top:30px; font-weight:bold;">{{ error }}</div>{% endif %}

        {% if tab == 'account' and user_info %}
        <div class="account-box">
            <div class="avatar">{{ user_info.username[:1] | upper }}</div>
            <div class="account-name">{{ user_info.username }}</div>
            <div class="stat-grid">
                <div><div class="stat-val">{{ user_info.bonus }}</div><div class="stat-lbl">魔力值</div></div>
                <div><div class="stat-val">{{ user_info.ratio }}</div><div class="stat-lbl">分享率</div></div>
                <div><div class="stat-val">{{ user_info.upload }}</div><div class="stat-lbl">总上传</div></div>
                <div><div class="stat-val">{{ user_info.download }}</div><div class="stat-lbl">总下载</div></div>
            </div>
            <button id="signinBtn" class="btn-signin" onclick="doSignIn()">✅ 手动签到</button>
        </div>
        <script>
            function doSignIn() {
                const btn = document.getElementById('signinBtn');
                btn.innerText = '签到中...'; btn.disabled = true;
                fetch('/api/signin').then(r => r.json()).then(data => {
                    alert(data.msg); btn.innerText = '✅ 手动签到'; btn.disabled = false;
                }).catch(e => { alert('请求失败'); btn.innerText = '✅ 手动签到'; btn.disabled = false; });
            }
        </script>
        
        {% elif tab != 'account' %}
            {% for item in torrents %}
            <div class="list-item" id="item-{{ item.id }}">
                <div class="poster-wrapper">
                    {% set needs_lazy = ('data:image/svg+xml' in item.img) or (item.size == '未知') or (item.size == '加载中...') %}
                    <img class="poster {% if needs_lazy %}lazy-load{% endif %}" 
                         src="{{ item.img }}" 
                         {% if needs_lazy %}data-tid="{{ item.id }}"{% endif %}
                         referrerpolicy="no-referrer" alt="海报" onclick="window.open('{{ item.link }}', '_blank')">
                    {% if 'Free' in item.tags or '免费' in item.tags %}
                        <div class="tag-free">FREE</div>
                    {% endif %}
                </div>
                <div class="info">
                    <div class="title" onclick="window.open('{{ item.link }}', '_blank')">{{ item.title }}</div>
                    <div class="subtitle">{{ item.subtitle }}</div>
                    
                    <div class="stats-row">
                        <span class="stat-item" id="size-{{ item.id }}">📄 {{ item.size }}</span>
                        <span class="stat-item"><span class="seed-icon">▲</span> {{ item.seeders }}</span>
                        <span class="stat-item"><span class="leech-icon">▼</span> {{ item.leechers }}</span>
                    </div>

                    <div class="bottom-actions">
                        <div class="tags-list">
                            {% for tag in item.tags %}
                                {% if tag not in ['Free', '免费'] %}
                                    {% if '%' in tag or '2X' in tag or '剩余' in tag %}
                                        <span class="badge-discount">{{ tag }}</span>
                                    {% else %}
                                        <span class="badge">{{ tag }}</span>
                                    {% endif %}
                                {% endif %}
                            {% endfor %}
                        </div>
                        <a href="/download/{{ item.id }}" class="download-btn">下载</a>
                    </div>
                </div>
            </div>
            {% endfor %}
            
            {% if tab == 'search' and not torrents and query %}
                <div style="text-align:center; color:var(--text-sub); margin-top:30px;">找到 0 个结果</div>
            {% elif tab == 'search' and torrents %}
                <div style="text-align:center; color:var(--text-sub); margin: 20px 0; font-size:12px;">已展示当前页全部结果</div>
            {% elif tab == 'recent' %}
                <div style="text-align:center; color:var(--text-sub); margin: 20px 0; font-size:12px;">后台静默监控中：已累积缓存最高 30 条历史记录</div>
            {% endif %}
        {% endif %}
    </div>

    <script>
        const sidebar = document.getElementById('sidebar');
        const overlay = document.getElementById('overlay');
        const menuBtn = document.getElementById('menuBtn');
        const themeIcon = document.getElementById('themeIcon');

        function toggleMenu() { 
            sidebar.classList.toggle('open'); 
            overlay.classList.toggle('open'); 
            menuBtn.classList.toggle('open');
        }

        const icons = { 'auto': '🔄', 'dark': '🌙', 'light': '☀️' };
        themeIcon.innerText = icons[currentTheme];

        function cycleTheme() {
            if (currentTheme === 'auto') currentTheme = 'dark';
            else if (currentTheme === 'dark') currentTheme = 'light';
            else currentTheme = 'auto';
            localStorage.setItem('longpt_theme', currentTheme);
            setTheme(currentTheme);
            themeIcon.innerText = icons[currentTheme];
        }

        document.addEventListener("DOMContentLoaded", function() {
            const lazyItems = document.querySelectorAll('.lazy-load');
            lazyItems.forEach(img => {
                const tid = img.getAttribute('data-tid');
                if (tid) {
                    fetch('/api/extra/' + tid)
                        .then(r => r.json())
                        .then(data => {
                            if (data.img) {
                                img.src = data.img;
                                img.classList.remove('lazy-load');
                            }
                            if (data.size && data.size !== '未知' && data.size !== '加载中...') {
                                const sizeSpan = document.getElementById('size-' + tid);
                                if (sizeSpan) sizeSpan.innerHTML = '📄 ' + data.size;
                            }
                            
                            if (data.tags && data.tags.length > 0) {
                                const itemDiv = document.getElementById('item-' + tid);
                                if (itemDiv) {
                                    const tagsList = itemDiv.querySelector('.tags-list');
                                    const posterWrapper = itemDiv.querySelector('.poster-wrapper');
                                    const existingTags = Array.from(tagsList.querySelectorAll('span')).map(s => s.innerText);
                                    let hasFree = !!posterWrapper.querySelector('.tag-free');
                                    
                                    data.tags.forEach(tag => {
                                        if (tag === 'Free' || tag === '免费') {
                                            if (!hasFree) {
                                                posterWrapper.insertAdjacentHTML('beforeend', '<div class="tag-free">FREE</div>');
                                                hasFree = true;
                                            }
                                        } else {
                                            const isMatch = existingTags.some(t => t.includes(tag) || tag.includes(t));
                                            if (!isMatch) {
                                                if (tag.includes('%') || tag.includes('2X') || tag.includes('剩余')) {
                                                    tagsList.insertAdjacentHTML('beforeend', `<span class="badge-discount">${tag}</span>`);
                                                } else {
                                                    tagsList.insertAdjacentHTML('beforeend', `<span class="badge">${tag}</span>`);
                                                }
                                                existingTags.push(tag);
                                            }
                                        }
                                    });
                                }
                            }
                        })
                        .catch(e => console.error('懒加载失败:', e));
                }
            });
        });
    </script>
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

        elif tab in ['official', 'search']:
            if tab == 'search' and not query: pass
            else:
                if tab == 'search':
                    safe_query = urllib.parse.quote(query)
                    target_url = f"https://longpt.org/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
                else: target_url = OFFICIAL_URL
                    
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
                            size, seeders, leechers = tds[-5].get_text(strip=True), tds[-4].get_text(strip=True), tds[-3].get_text(strip=True)
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
                            
                        subtitle = re.sub(r'\[?(?:剩余时间)[:：]*\]?[\s\|]*([\d]+天[\d]+[时分秒]+|[\d]+[天时分秒]+)', r'[剩余时间: \1]', subtitle)
                        subtitle = subtitle.replace('[ | ]', '').replace('[ | | ]', '')
                            
                        tags, clean_sub = [], subtitle
                        for m in re.finditer(r'\[(.*?)\]', subtitle):
                            tag = m.group(1).strip(' |')
                            # 修改点2：精确提取徽章，普通文本脱去中括号还给副标题
                            if any(x in tag.upper() for x in ['热门', 'FREE', '免费', '%', '2X', '剩余']):
                                if len(tag) <= 18: tags.append(tag)
                                clean_sub = clean_sub.replace(m.group(0), '')
                            elif tag not in ['Sticky', 'poster', '置顶']:
                                clean_sub = clean_sub.replace(m.group(0), tag)
                            else:
                                clean_sub = clean_sub.replace(m.group(0), '')
                        
                        clean_sub = re.sub(r'热门|免费', '', clean_sub)
                        subtitle = re.sub(r'\|(?:\s*\|)+', '|', clean_sub).strip(' |')
                            
                        torrents.append({
                            'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                            'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags
                        })

        elif tab == 'recent':
            if os.path.exists(HISTORY_FILE):
                try:
                    with open(HISTORY_FILE, 'r', encoding='utf-8') as f: torrents = json.load(f)
                except: pass

            resp = requests.get(INDEX_URL, headers=HEADERS, timeout=10)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效！"
            elif resp.status_code == 200:
                soup = BeautifulSoup(resp.text, 'html.parser')
                current_fetched = []
                for title_a in soup.find_all('a', href=lambda h: h and h.startswith('details.php?id=')):
                    tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                    title = title_a.get('title') or title_a.get_text(strip=True)
                    if not title and title_a.find('img'): title = title_a.find('img').get('alt') or title_a.find('img').get('title')
                    if not title or tid in seen_ids: continue
                    seen_ids.add(tid)
                    
                    link = "https://longpt.org/" + title_a['href']
                    container = title_a.find_parent('tr') or title_a.find_parent('div')
                    img_src, subtitle, seeders, leechers, size = "", "", "-", "-", "加载中..."

                    if container:
                        if container.name == 'tr':
                            tds = container.find_all('td')
                            if len(tds) >= 3: seeders, leechers = tds[-2].get_text(strip=True), tds[-1].get_text(strip=True)

                        for img in container.find_all('img'):
                            src = img.get('data-src') or img.get('src') or ""
                            if src and 'category' not in src.lower() and 'cat' not in src.lower():
                                img_src = src if src.startswith('http') else "https://longpt.org/" + src
                                break
                        for icon in container.find_all('img'):
                            alt = icon.get('alt') or icon.get('title')
                            if alt and icon.get('src', '') != img_src: icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                        subtitle = container.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                        if seeders != "-" and subtitle.endswith(leechers): subtitle = subtitle.rsplit('|', 2)[0].strip(' | ')
                        
                        subtitle = re.sub(r'\[?(?:剩余时间)[:：]*\]?[\s\|]*([\d]+天[\d]+[时分秒]+|[\d]+[天时分秒]+)', r'[剩余时间: \1]', subtitle)
                        subtitle = subtitle.replace('[ | ]', '').replace('[ | | ]', '')
                            
                        tags, clean_sub = [], subtitle
                        for m in re.finditer(r'\[(.*?)\]', subtitle):
                            tag = m.group(1).strip(' |')
                            if any(x in tag.upper() for x in ['热门', 'FREE', '免费', '%', '2X', '剩余']):
                                if len(tag) <= 18: tags.append(tag)
                                clean_sub = clean_sub.replace(m.group(0), '')
                            elif tag not in ['Sticky', 'poster', '置顶']:
                                clean_sub = clean_sub.replace(m.group(0), tag)
                            else:
                                clean_sub = clean_sub.replace(m.group(0), '')
                        
                        time_match = re.search(r'(剩余时间[^\s\|]+)', clean_sub)
                        if time_match:
                            tags.append(time_match.group(1).strip(' |'))
                            clean_sub = clean_sub.replace(time_match.group(0), '')
                            
                        clean_sub = re.sub(r'热门|免费', '', clean_sub)
                        subtitle = re.sub(r'\|(?:\s*\|)+', '|', clean_sub).strip(' |')

                    if not img_src: img_src = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23e6e6e6'/><text x='50%' y='50%' fill='%23999' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>加载海报</text></svg>"

                    current_fetched.append({
                        'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                        'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags
                    })

                history_dict = {str(item['id']): item for item in torrents}
                for item in current_fetched:
                    tid_str = str(item['id'])
                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached['size'] not in ['未知', '加载中...']: item['size'] = cached['size']
                        merged_tags = item.get('tags', [])
                        for t in cached.get('tags', []):
                            if t not in merged_tags: merged_tags.append(t)
                        item['tags'] = merged_tags
                    history_dict[tid_str] = item

                merged = list(history_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = merged[:30]

    except Exception as e:
        error_msg = f"服务器内部请求错误: {e}"
        
    return render_template_string(HTML_TEMPLATE, tab=tab, query=query, torrents=torrents, user_info=user_info, error=error_msg)

@app.route('/api/extra/<int:tid>')
def api_extra(tid):
    img_src, size, extra_tags = fetch_details_info(tid)
    if os.path.exists(HISTORY_FILE):
        try:
            with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history = json.load(f)
            updated = False
            for item in history:
                if str(item['id']) == str(tid):
                    if img_src: item['img'] = img_src
                    if size and size != "未知": item['size'] = size
                    if extra_tags:
                        new_tags = item.get('tags', [])
                        for t in extra_tags:
                            if t not in new_tags: new_tags.append(t)
                        item['tags'] = new_tags
                    updated = True
                    break
            if updated:
                with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
        except: pass
    return jsonify({"img": img_src, "size": size, "tags": extra_tags})

@app.route('/api/signin')
def api_signin():
    try:
        r = requests.get(SIGNIN_URL, headers=HEADERS, timeout=10)
        if "签到成功" in r.text or "这是您的第" in r.text or "已连续签到" in r.text: return jsonify({"msg": "🎉 签到成功！"})
        elif "已经" in r.text or "退下" in r.text: return jsonify({"msg": "☕ 今天已经签到过了，明天再来吧。"})
        elif "登录" in r.text: return jsonify({"msg": "⚠️ 签到失败：Cookie 已失效，请更新！"})
        else: return jsonify({"msg": "🤔 状态未知，可能已签到，请留意积分变化。"})
    except Exception as e: return jsonify({"msg": f"❌ 网络请求出错: {str(e)}"})

@app.route('/download/<int:tid>')
def download_torrent(tid):
    try:
        r = requests.get(f"https://longpt.org/download.php?id={tid}", headers=HEADERS, timeout=10)
        if "application/x-bittorrent" in r.headers.get('Content-Type', '') or "octet-stream" in r.headers.get('Content-Type', ''):
            return Response(r.content, mimetype="application/x-bittorrent", headers={"Content-Disposition": f"attachment;filename=LongPT_{tid}.torrent"})
        else: return "下载失败，Cookie 可能失效或无权下载该种子。"
    except Exception as e: return f"请求下载出错: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5051)
```

5.h5baozi.py

每天记录签到、15分钟刷新、50条缓存记录
```
from flask import Flask, render_template_string, Response, request, jsonify
import requests
from bs4 import BeautifulSoup
import urllib.parse
import re
import os
import json
import threading
import time
import datetime

app = Flask(__name__)

# ================= 必须配置的区域 =================
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    # 【安全提醒】：请把你的 包子PT Cookie 填在下面
    'Cookie': 'cf_clearance=DwF4x1y14.1homeNQyBySkndTLSEKGhl3GinwCPAP9I-1788588997-1.2.1.1-O.jM_uS1JJQVblexLX6tAsmdQl5PeNPcur8eBkld0lQG_QvVgfmwRO2283Yu33TrY0rn735kiKt40PFZdtpg.aulJffet4aJwTdlQNj0r.R6mm9nO6fMCMSz5NqTmq6uyxRhtNkfAfudPD26EczjLDq9v_lgODY.6aTHJamqcmAMulg0W1pISDLmBIV5CAlmNcGQlotm7iX9g2UEGQ8rRKj79i2WYIC8BpIBlo3mKOKWXTysqD6NpkBNVfv_hPlCueBnEOFuE5E6uIsBdWUU.dA.CNAPQu1eFyCzSUAmwRPrzi1x2Mg_D2Ygd1ZuAXSeYTERPHEgNL5vc8YX7V5RrqeOp6M._g391BjhM.d3JTCdC7txZtqGdLWfYwNXaJjRACjOk9BLIc3UxNitJAAuS9rnzt1hkURy2jOWphvKecVE_wGjG6ZuBU6S7ycE_Fd6c31omxc283WRvNIQPGLbxh3zABaBkGT1p9tM7HjN8A0oe7wrtwYimgXrKLvbV4FwEfJIq39qQ.dX5XPsM4Qg8A; c_secure_pass=eyJ1c2VyX2lkIjoiMTM0NzAiLCJleHBpcmVzIjoxODIwMTI1MDE1fS5iMmI3MzU0YmVjZDZjYzg2Mjk5YWQ2MGM0Yjg3ZjdkODM1MTFlMzhkYmNhMDY2NTc5MGQ1YTJhNDUyNmVlNTk5' 
}

PROXY_URL = ""
PROXIES = {"http": PROXY_URL, "https": PROXY_URL} if PROXY_URL else None
TIMEOUT = 30

BASE_URL = "https://p.t-baozi.cc"
OFFICIAL_URL = f"{BASE_URL}/torrents.php"
INDEX_URL = f"{BASE_URL}/index.php"
SIGNIN_URL = f"{BASE_URL}/attendance.php"
HISTORY_FILE = "baozi_history.json"
# =================================================

# 严苛白名单库
TECH_KEYWORDS = [
    'FREE', '50%', '30%', '2X', '热门', '官方', '中字', '英字', '国语', '粤语', '英语', 
    '简繁', '双语', '杜比', '高帧', '高码', '臻彩MAX', '高分', '完结', '4K', '1080P', 
    '2160P', '720P', 'WEB-DL', 'BLU-RAY', 'REMUX', 'HDR10+', 'HDR10', 'HDRVIVID', 
    'HDR', 'DOVI', 'DV', 'SDR', '60FPS', 'HFR', 'HQ', 'H.265', 'H.264', 'H265', 'H264', 
    'HEVC', 'AAC', 'DDP', 'ATMOS', '10BIT', '首发', '禁转', '应求', 'DIY', '字幕'
]

def purify_tags(tags):
    valid = []
    for t in tags:
        clean_t = t.upper()
        if '剩余' in clean_t:
            valid.append(t)
        elif clean_t in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', clean_t):
            valid.append(t)
            
    final_tags = []
    time_tags = [t for t in valid if '剩余' in t]
    best_time = max(time_tags, key=len) if time_tags else None
    
    for t in valid:
        if '剩余' in t:
            if t == best_time and best_time not in final_tags:
                final_tags.append(t)
        elif t not in final_tags:
            final_tags.append(t)
            
    return final_tags

def auto_extract_tags(text, current_tags):
    en_matches = re.findall(r'(?i)\b(4K|1080P|2160P|720P|WEB-DL|BLU-RAY|REMUX|HDR10\+|HDR10|HDRVIVID|HDR|DOVI|DV|SDR|60FPS|HFR|HQ|H\.?265|H\.?264|HEVC|AAC|DDP|ATMOS|10BIT)\b', text)
    cn_matches = re.findall(r'(杜比|高帧|高码|臻彩MAX|高分|完结|国语|中字|英字|粤语|英语|简繁|双语|官方|首发)', text)
    disc_matches = re.findall(r'(50%|30%|2X)', text.upper())
    
    all_new = [m.upper() for m in en_matches] + cn_matches + disc_matches
    for tag in all_new:
        tag_up = tag.upper()
        if not any(tag_up in t.upper() or t.upper() in tag_up for t in current_tags):
            current_tags.append(tag)
    return current_tags

def sort_tags(tags):
    top_pri, top_sec, bot_disc, bot_pri, bot_sec = [], [], [], [], []
    remaining = list(tags)
    
    t_res = ['4K', '2160P', '1080P', '720P']
    t_hdr = ['HDRVIVID', 'HDR10+', 'HDR10', 'HDR', 'DOVI', 'DV', 'SDR']
    t_fps = ['60FPS', 'HFR', '高帧', '臻彩MAX', 'BLU-RAY', 'REMUX']
    t_enc = ['WEB-DL', 'H.265', 'H.264', 'H265', 'H264', 'HEVC', 'AAC', 'DDP', 'ATMOS', '杜比', '10BIT', 'HQ', '高码']
    
    for t in list(remaining):
        t_upper = t.upper()
        if '%' in t_upper or '2X' in t_upper or t_upper in ['免费', 'FREE', '30%', '50%']:
            if t_upper not in ['FREE', '免费']:
                bot_disc.append(t)
            remaining.remove(t)
    
    t_status = ['热门', '完结', '首发', '高分', 'DIY']
    t_lang = ['国语', '中字', '英字', '粤语', '英语', '简繁', '双语', '官方', '字幕']
    
    for kw in t_res + t_hdr + t_fps:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_pri.append(t)
                remaining.remove(t)
                
    for kw in t_enc:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_sec.append(t)
                remaining.remove(t)
                
    for kw in t_status:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_pri.append(t)
                remaining.remove(t)
                
    for kw in t_lang:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_sec.append(t)
                remaining.remove(t)
                
    for t in list(remaining):
        bot_pri.append(t)
        
    return top_pri, top_sec, bot_disc, bot_pri, bot_sec

def fetch_details_info(tid):
    try:
        r = requests.get(f"{BASE_URL}/details.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        soup = BeautifulSoup(r.text, 'html.parser')

        img_src = ""
        descr_div = soup.find('div', id='kdescr')
        if descr_div:
            img = descr_div.find('img')
            if img: img_src = img.get('data-src') or img.get('src') or ""
        if not img_src:
            for img in soup.find_all('img'):
                src = img.get('data-src') or img.get('src') or ""
                if src and not any(x in src.lower() for x in ['category', 'smilies', 'icon', 'logo', 'avatar', 'rating', 'button', 'rule']):
                    img_src = src
                    break
        if img_src and not img_src.startswith('http'):
            img_src = f"{BASE_URL}/" + img_src

        size = "未知"
        size_match = re.search(r'大小.*?([\d\.]+\s*[KMGTP]B)', r.text, re.IGNORECASE)
        if size_match: size = size_match.group(1)

        extra_tags = []
        for img in soup.find_all('img'):
            alt = str(img.get('alt') or img.get('title') or '').replace('[', '').replace(']', '').strip()
            if alt.upper() in ['FREE', '免费']:
                if 'Free' not in extra_tags: extra_tags.append('Free')
            elif any(x in alt for x in ['50%', '2X', '30%', '热门']):
                if alt not in extra_tags: extra_tags.append(alt)
        
        for span in soup.find_all('span', class_=re.compile(r'tag|label|badge', re.I)):
            text = span.get_text(strip=True).replace('[', '').replace(']', '').strip()
            if text.upper() in ['FREE', '免费']:
                if 'Free' not in extra_tags: extra_tags.append('Free')
            elif text and text not in extra_tags and len(text) <= 15:
                extra_tags.append(text)
        
        page_text = soup.get_text(separator=' ')
        time_match = re.search(r'剩余时间[:：\s]*(\d+\s*[天小时分秒钟]+(?:\s*\d+\s*[天小时分秒钟]+)*)', page_text)
        if time_match: extra_tags.append(f"剩余时间: {time_match.group(1).strip()}")
            
        h1 = soup.find('h1')
        h1_text = ""
        release_group = ""
        if h1:
            h1_text = h1.get_text(strip=True)
            for kw in ['免费', 'FREE', '50%', '30%', '2X']:
                if kw in h1_text.upper():
                    if kw in ['免费', 'FREE']:
                        if 'Free' not in extra_tags: extra_tags.append('Free')
                    else:
                        if kw not in extra_tags: extra_tags.append(kw)
                        
            group_match = re.search(r'-([A-Za-z0-9@_]+)$', h1_text)
            if group_match:
                release_group = group_match.group(1).upper()

        sub_td = soup.find('td', class_='rowhead', string=re.compile('副标题|副標題'))
        sub_text = ""
        if sub_td:
            sub_text = sub_td.find_next_sibling('td').get_text(strip=True)
            
        # 💡【核心修复】：连带标题 h1 和副标题一起送入扫描引擎，绝不错漏任何隐藏参数！
        extra_tags = auto_extract_tags(h1_text + " " + sub_text, extra_tags)
        
        return img_src, size, purify_tags(extra_tags), release_group
    except Exception as e:
        return "", "未知", [], ""

def parse_torrents_page(target_url, seen_ids, is_search=False):
    torrents = []
    resp = requests.get(target_url, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
    if "登录" in resp.text and "torrents.php" not in resp.url: return None, "⚠️ Cookie 已失效！"
    
    if resp.status_code == 200:
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
            link = f"{BASE_URL}/" + title_a['href']
            
            release_group = ""
            group_match = re.search(r'-([A-Za-z0-9@_]+)$', title.strip())
            if group_match:
                release_group = group_match.group(1).upper()
            
            try: size, seeders, leechers = tds[-5].get_text(strip=True), tds[-4].get_text(strip=True), tds[-3].get_text(strip=True)
            except: size, seeders, leechers = "未知", "-", "-"

            img_src = ""
            td = title_a.find_parent('td')
            tags = []
            subtitle = ""
            
            if td:
                for icon in td.find_all('img'):
                    alt = icon.get('alt') or icon.get('title')
                    if alt and icon.get('src', '') != img_src: icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                
                for span in td.find_all('span'):
                    text = span.get_text(strip=True).replace('[', '').replace(']', '').strip()
                    if not text: continue
                    if text.upper() in ['FREE', '免费']:
                        if 'Free' not in tags: tags.append('Free')
                        span.replace_with(soup.new_tag("span", string=f" [FREE] "))
                        continue
                        
                    span_class = ' '.join(span.get('class', [])).lower()
                    is_valid_class = bool(re.search(r'tag|label|badge', span_class))
                    is_known_kw = bool(text.upper() in [k.upper() for k in TECH_KEYWORDS])
                    
                    if (is_valid_class or is_known_kw) and text.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', text):
                        if text not in tags: tags.append(text)
                        span.replace_with(soup.new_tag("span", string=f" [{text}] "))
                
                subtitle = td.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                
                subtitle = re.sub(r'(?i)(?:Sticky|置顶)\s*\|\s*', '', subtitle)
                
                time_match = re.search(r'\[?(?:剩余时间)[:：\s]*\]?[\s\|]*(\d+\s*[天小时分秒钟]+(?:\s*\d+\s*[天小时分秒钟]+)*)', subtitle)
                if time_match:
                    time_str = f"剩余时间: {time_match.group(1).strip()}"
                    if time_str not in tags: tags.append(time_str)
                    subtitle = subtitle.replace(time_match.group(0), '')
                
                clean_sub = subtitle
                for m in re.finditer(r'\[(.*?)\]', subtitle):
                    tag = m.group(1).strip(' |[]')
                    if tag.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', tag) or tag.upper() in ['热门']:
                        if tag not in tags: tags.append(tag)
                        clean_sub = clean_sub.replace(m.group(0), '')
                    else:
                        clean_sub = clean_sub.replace(m.group(0), tag)
                
                clean_sub = re.sub(r'热门|免费', '', clean_sub)
                
                parts = [p.strip() for p in clean_sub.split('|') if p.strip()]
                kept_sub = []
                for p in parts:
                    clean_p = re.sub(r'^[()\[\]\-|\s]+|[()\[\]\-|\s]+$', '', p)
                    if not clean_p: continue
                    if clean_p.upper() in [k.upper() for k in TECH_KEYWORDS]:
                        if clean_p not in tags: tags.append(clean_p)
                    else:
                        kept_sub.append(p)
                subtitle = ' | '.join(kept_sub)
            
            tags = auto_extract_tags(title + " " + subtitle, tags)
            tags = purify_tags(tags)

            torrents.append({
                'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags, 'group': release_group
            })
        return torrents, None
    return None, f"请求网页失败，状态码: {resp.status_code}"

def background_monitor():
    last_sign_date = None
    while True:
        try:
            current_date = datetime.datetime.now().strftime("%Y-%m-%d")
            if last_sign_date != current_date:
                requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
                last_sign_date = current_date
        except Exception: pass

        try:
            current_fetched, err = parse_torrents_page(OFFICIAL_URL, set())
            if current_fetched:
                history_data = []
                if os.path.exists(HISTORY_FILE):
                    try:
                        with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history_data = json.load(f)
                    except: pass
                    
                history_dict = {str(item['id']): item for item in history_data}
                
                for item in current_fetched[:5]:
                    tid_str = str(item['id'])
                    if tid_str not in history_dict or 'data:image/svg+xml' in history_dict[tid_str]['img'] or history_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags, det_group = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_group and not item.get('group'): item['group'] = det_group
                        if det_tags:
                            new_tags = item.get('tags', [])
                            for t in det_tags:
                                if t not in new_tags: new_tags.append(t)
                            item['tags'] = new_tags

                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if not item.get('group') and cached.get('group'): item['group'] = cached['group']
                        
                        merged_tags = item.get('tags', [])
                        for t in cached.get('tags', []):
                            if t not in merged_tags: merged_tags.append(t)
                        item['tags'] = merged_tags
                        
                    item['tags'] = purify_tags(auto_extract_tags(item['title'] + " " + item['subtitle'], item['tags']))
                    history_dict[tid_str] = item

                merged = list(history_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(merged[:50], f, ensure_ascii=False, indent=2)
        except Exception as e: pass
        time.sleep(900)

threading.Thread(target=background_monitor, daemon=True).start()

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>包子PT 手机端</title>
    <style>
        :root { --bg-color: #f7f7f7; --card-bg: #ffffff; --text-main: #333333; --text-sub: #888888; --border-color: #eeeeee; --topbar-bg: #ffffff; --icon-bg: #f0f0f5; --primary-color: #ff8c00; }
        [data-theme="dark"] { --bg-color: #121212; --card-bg: #1e1e1e; --text-main: #eeeeee; --text-sub: #aaaaaa; --border-color: #333333; --topbar-bg: #1e1e1e; --icon-bg: #333333; --primary-color: #ff9800; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background-color: var(--bg-color); color: var(--text-main); margin: 0; padding: 0; padding-top: 55px; transition: background-color 0.3s; }
        
        .top-bar { position: fixed; top: 0; left: 0; right: 0; height: 50px; background: var(--topbar-bg); display: flex; align-items: center; justify-content: space-between; padding: 0 15px; box-shadow: 0 1px 3px rgba(0,0,0,0.05); z-index: 100; transition: background 0.3s; }
        .menu-btn { width: 22px; height: 16px; position: relative; cursor: pointer; display: flex; flex-direction: column; justify-content: space-between; padding: 0; }
        .menu-btn span { display: block; height: 2px; width: 100%; background: var(--text-main); border-radius: 2px; transition: all 0.3s ease; }
        .menu-btn.open span:nth-child(1) { transform: translateY(7px) rotate(45deg); }
        .menu-btn.open span:nth-child(2) { opacity: 0; }
        .menu-btn.open span:nth-child(3) { transform: translateY(-7px) rotate(-45deg); }

        .logo { font-weight: 800; font-size: 20px; color: var(--primary-color); letter-spacing: 1px; display: flex; align-items: center; gap: 4px;}
        .logo-icon { font-size: 22px; }
        .top-placeholder { width: 22px; }

        .sidebar { position: fixed; top: 0; left: -260px; bottom: 0; width: 260px; background: var(--card-bg); z-index: 1001; transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: 2px 0 10px rgba(0,0,0,0.1); padding-top: 60px; display: flex; flex-direction: column; }
        .sidebar.open { left: 0; }
        .menu-item { padding: 16px 24px; font-size: 16px; font-weight: 500; color: var(--text-main); text-decoration: none; display: block; border-bottom: 1px solid var(--border-color); }
        .menu-item.active { color: var(--primary-color); border-left: 4px solid var(--primary-color); padding-left: 20px; background: rgba(255,140,0,0.05); }
        .theme-toggle-icon { margin: auto auto 30px 24px; width: 40px; height: 40px; border-radius: 20px; background: var(--icon-bg); display: flex; align-items: center; justify-content: center; font-size: 20px; cursor: pointer; user-select: none; transition: transform 0.2s; }
        .theme-toggle-icon:active { transform: scale(0.9); }
        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 1000; opacity: 0; pointer-events: none; transition: opacity 0.3s; }
        .overlay.open { opacity: 1; pointer-events: auto; }

        .container { padding: 5px 10px 10px 10px; }
        
        .global-search-container { display: flex; gap: 8px; margin: 5px 5px 15px 5px; align-items: center; background: var(--card-bg); padding: 5px; border-radius: 20px; border: 1px solid var(--border-color); box-shadow: 0 2px 8px rgba(0,0,0,0.02); }
        .search-input { flex: 1; padding: 8px 15px; border: none; outline: none; font-size: 14px; background: transparent; color: var(--text-main); }
        .search-btn { background: rgba(255,140,0,0.1); color: var(--primary-color); border: none; width: 36px; height: 36px; border-radius: 18px; font-weight: bold; font-size: 16px; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s ease; flex-shrink: 0; }
        .search-btn:active { background: var(--primary-color); color: #fff; }
        [data-theme="dark"] .search-btn { background: rgba(255,152,0,0.15); }
        
        .list-item { display: flex; gap: 12px; padding: 15px 5px; border-bottom: 1px solid var(--border-color); }
        .poster-wrapper { position: relative; flex-shrink: 0; width: 100px; height: 140px; }
        .poster { width: 100%; height: 100%; object-fit: cover; border-radius: 8px; background-color: #e0e0e0; transition: opacity 0.5s ease-in-out; }
        .tag-free { position: absolute; top: 0; left: 0; background: #22c55e; color: #fff; font-size: 10px; font-weight: bold; padding: 2px 6px; border-radius: 8px 0 8px 0; z-index: 10; }
        
        .info { flex-grow: 1; display: flex; flex-direction: column; overflow: hidden; justify-content: space-between; padding-bottom: 2px; }
        .title { font-size: 15px; font-weight: bold; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; line-height: 1.3; }
        .subtitle { font-size: 12px; color: var(--text-sub); line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; margin-bottom: 4px; }
        .subtitle:empty { display: none; margin: 0; }
        
        .stats-and-download { display: flex; justify-content: space-between; align-items: center; gap: 10px; width: 100%; margin-bottom: 4px; }
        .stats-row { font-size: 12px; color: var(--text-sub); display: flex; align-items: center; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; flex: 1; -webkit-overflow-scrolling: touch; }
        .stats-row::-webkit-scrollbar { display: none; }
        .stat-item { display: flex; align-items: center; gap: 2px; }
        .seed-icon { color: #22c55e; font-size: 10px; }
        .leech-icon { color: #ef4444; font-size: 10px; }
        
        .tags-wrapper { display: flex; flex-direction: column; gap: 6px; margin-top: auto; width: 100%; overflow: hidden; padding-top: 2px; }
        .tags-top { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; }
        .tags-top::-webkit-scrollbar { display: none; }
        .tags-top:empty { display: none; }
        
        .tags-bottom { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; width: 100%; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; padding-bottom: 2px; }
        .tags-bottom::-webkit-scrollbar { display: none; }
        .tags-bottom:empty { display: none; }
        
        /* 💡折叠开关控制 */
        .secondary-tag { display: none !important; }
        .show-extra .secondary-tag { display: inline-flex !important; }
        
        .badge-toggle { background: #e5e7eb; color: #6b7280; font-size: 10px; padding: 2px 8px; border-radius: 4px; flex-shrink: 0; cursor: pointer; display: inline-flex; align-items: center; letter-spacing: 1px; font-weight: bold; transition: all 0.2s ease;}
        [data-theme="dark"] .badge-toggle { background: #374151; color: #9ca3af; }
        .badge-toggle:active { opacity: 0.7; }
        
        .badge { background: #8b5cf6; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-discount { background: #ef4444; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-tech { background: #0ea5e9; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-lang { background: #10b981; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        
        .badge-group { background: #ec4899; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; letter-spacing: 0.5px;} 
        /* 💡LongWeb 专属的纯图标精简版药丸 */
        .badge-longweb { background: linear-gradient(135deg, #2563eb 0%, #06b6d4 100%); color: #fff; font-size: 10px; padding: 2px 4px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; box-shadow: 0 1px 3px rgba(37,99,235,0.2);}

        .download-btn { background: var(--primary-color); color: #fff; font-size: 10px; padding: 2px 8px; border-radius: 4px; font-weight: bold; text-decoration: none; flex-shrink: 0; display: inline-flex; align-items: center; letter-spacing: 1px; min-width: 36px; justify-content: center;}
        .download-btn:active { opacity: 0.8; }
        
        .account-box { background: var(--card-bg); border-radius: 12px; padding: 20px; text-align: center; border: 1px solid var(--border-color); }
        .avatar { width: 70px; height: 70px; background: var(--primary-color); color: #fff; border-radius: 35px; line-height: 70px; font-size: 24px; margin: 0 auto 10px; }
        .account-name { font-size: 18px; font-weight: bold; }
        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }
        .stat-val { font-size: 18px; font-weight: bold; color: var(--primary-color); margin-bottom: 4px; }
        .stat-lbl { font-size: 12px; color: var(--text-sub); }
        .btn-signin { background: #22c55e; color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 25px; border: none; cursor: pointer; }
    </style>
</head>
<body>
    <script>
        function setTheme(theme) {
            const html = document.documentElement;
            if (theme === 'dark') { html.setAttribute('data-theme', 'dark'); }
            else if (theme === 'light') { html.removeAttribute('data-theme'); }
            else { 
                if (window.matchMedia('(prefers-color-scheme: dark)').matches) html.setAttribute('data-theme', 'dark');
                else html.removeAttribute('data-theme');
            }
        }
        let currentTheme = localStorage.getItem('baozipt_theme') || 'auto';
        setTheme(currentTheme);
        window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
            if (localStorage.getItem('baozipt_theme') === 'auto') setTheme('auto');
        });
        
        window.toggleTags = function(btn) {
            const wrapper = btn.closest('.tags-wrapper');
            wrapper.classList.toggle('show-extra');
            if (wrapper.classList.contains('show-extra')) {
                btn.innerText = '收起 ▴';
            } else {
                btn.innerText = '... ▾';
            }
        };
    </script>

    <div class="top-bar">
        <div class="menu-btn" id="menuBtn" onclick="toggleMenu()">
            <span></span><span></span><span></span>
        </div>
        <div class="logo"><span class="logo-icon">🥟</span>包子PT</div>
        <div class="top-placeholder"></div>
    </div>

    <div class="overlay" id="overlay" onclick="toggleMenu()"></div>
    <div class="sidebar" id="sidebar">
        <a href="/?tab=official" class="menu-item {% if tab == 'official' %}active{% endif %}">全部发布</a>
        <a href="/?tab=recent" class="menu-item {% if tab == 'recent' %}active{% endif %}">最近发布</a>
        <a href="/?tab=account" class="menu-item {% if tab == 'account' %}active{% endif %}">我的帐号</a>
        <div class="theme-toggle-icon" id="themeIcon" onclick="cycleTheme()">🔄</div>
    </div>

    <div class="container">
        {% if tab in ['official', 'recent', 'search'] %}
        <form class="global-search-container" action="/" method="GET">
            <input type="hidden" name="tab" value="search">
            <input type="text" name="q" class="search-input" placeholder="输入关键词搜全站..." value="{{ query }}">
            <button type="submit" class="search-btn">🔍</button>
        </form>
        {% endif %}

        {% if error %}<div style="text-align:center; color:#ef4444; margin-top:30px; font-weight:bold;">{{ error }}</div>{% endif %}

        {% if tab == 'account' and user_info %}
        <div class="account-box">
            <div class="avatar">{{ user_info.username[:1] | upper }}</div>
            <div class="account-name">{{ user_info.username }}</div>
            <div class="stat-grid">
                <div><div class="stat-val">{{ user_info.bonus }}</div><div class="stat-lbl">魔力值</div></div>
                <div><div class="stat-val">{{ user_info.ratio }}</div><div class="stat-lbl">分享率</div></div>
                <div><div class="stat-val">{{ user_info.upload }}</div><div class="stat-lbl">总上传</div></div>
                <div><div class="stat-val">{{ user_info.download }}</div><div class="stat-lbl">总下载</div></div>
            </div>
            <button id="signinBtn" class="btn-signin" onclick="doSignIn()">✅ 手动签到</button>
        </div>
        <script>
            function doSignIn() {
                const btn = document.getElementById('signinBtn');
                btn.innerText = '签到中...'; btn.disabled = true;
                fetch('/api/signin').then(r => r.json()).then(data => {
                    alert(data.msg); btn.innerText = '✅ 手动签到'; btn.disabled = false;
                }).catch(e => { alert('请求失败'); btn.innerText = '✅ 手动签到'; btn.disabled = false; });
            }
        </script>
        
        {% elif tab != 'account' %}
            {% for item in torrents %}
            <div class="list-item" id="item-{{ item.id }}">
                <div class="poster-wrapper">
                    {% set needs_lazy = not item.img or ('data:image/svg+xml' in item.img) %}
                    <img class="poster {% if needs_lazy %}lazy-load{% endif %}" 
                         src="{{ item.img if item.img else 'data:image/svg+xml;utf8,<svg xmlns=\\'http://www.w3.org/2000/svg\\' width=\\'100\\' height=\\'140\\'><rect width=\\'100%\\' height=\\'100%\\' fill=\\'%23e6e6e6\\'/><text x=\\'50%\\' y=\\'50%\\' fill=\\'%23999\\' font-size=\\'12\\' font-family=\\'sans-serif\\' text-anchor=\\'middle\\' dominant-baseline=\\'middle\\'>加载海报</text></svg>' }}" 
                         {% if needs_lazy %}data-tid="{{ item.id }}"{% endif %}
                         referrerpolicy="no-referrer" alt="海报" onclick="window.open('{{ item.link }}', '_blank')">
                    {% if 'FREE' in item.tags|join(' ')|upper %}
                        <div class="tag-free">FREE</div>
                    {% endif %}
                </div>
                <div class="info">
                    <div class="title" onclick="window.open('{{ item.link }}', '_blank')">{{ item.title }}</div>
                    <div class="subtitle">{{ item.subtitle }}</div>
                    
                    <div class="stats-and-download">
                        <div class="stats-row">
                            <span class="stat-item" id="size-{{ item.id }}">📄 {{ item.size }}</span>
                            <span class="stat-item"><span class="seed-icon">▲</span> {{ item.seeders }}</span>
                            <span class="stat-item"><span class="leech-icon">▼</span> {{ item.leechers }}</span>
                            {% if item.group %}
                                {% if item.group|upper == 'LONGWEB' %}
                                    <span class="badge-longweb">💎</span>
                                {% else %}
                                    <span class="badge-group">{{ item.group }}</span>
                                {% endif %}
                            {% endif %}
                        </div>
                        <a href="/download/{{ item.id }}" class="download-btn">下载</a>
                    </div>

                    <div class="tags-wrapper">
                        <!-- 第一栏：渲染经后端排好序的高级显式/隐式标签 -->
                        <div class="tags-top">
                            {% for tag in item.top_pri %}
                                <span class="badge-tech">{{ tag }}</span>
                            {% endfor %}
                            {% for tag in item.top_sec %}
                                <span class="badge-tech secondary-tag">{{ tag }}</span>
                            {% endfor %}
                        </div>

                        <div class="tags-bottom">
                            {% if item.time_tag %}
                                <span class="badge-discount">{{ item.time_tag }}</span>
                            {% endif %}
                            
                            {% for tag in item.bot_disc %}
                                <span class="badge-discount">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_pri %}
                                <span class="badge">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_sec %}
                                <span class="badge-lang secondary-tag">{{ tag }}</span>
                            {% endfor %}
                            
                            {% if item.top_sec|length > 0 or item.bot_sec|length > 0 %}
                                <span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>
                            {% endif %}
                        </div>
                    </div>
                </div>
            </div>
            {% endfor %}
            
            {% if tab == 'search' and not torrents and query %}
                <div style="text-align:center; color:var(--text-sub); margin-top:30px;">找到 0 个结果</div>
            {% elif tab == 'search' and torrents %}
                <div style="text-align:center; color:var(--text-sub); margin: 20px 0; font-size:12px;">已展示当前页全部结果</div>
            {% elif tab == 'recent' %}
                <div style="text-align:center; color:var(--text-sub); margin: 20px 0; font-size:12px;">智能四维折叠系统已启动，一切就绪</div>
            {% endif %}
        {% endif %}
    </div>

    <script>
        const sidebar = document.getElementById('sidebar');
        const overlay = document.getElementById('overlay');
        const menuBtn = document.getElementById('menuBtn');
        const themeIcon = document.getElementById('themeIcon');

        function toggleMenu() { 
            sidebar.classList.toggle('open'); 
            overlay.classList.toggle('open'); 
            menuBtn.classList.toggle('open');
        }

        const icons = { 'auto': '🔄', 'dark': '🌙', 'light': '☀️' };
        themeIcon.innerText = icons[currentTheme];

        function cycleTheme() {
            if (currentTheme === 'auto') currentTheme = 'dark';
            else if (currentTheme === 'dark') currentTheme = 'light';
            else currentTheme = 'auto';
            localStorage.setItem('baozipt_theme', currentTheme);
            setTheme(currentTheme);
            themeIcon.innerText = icons[currentTheme];
        }

        document.addEventListener("DOMContentLoaded", function() {
            const lazyItems = document.querySelectorAll('.lazy-load');
            lazyItems.forEach(img => {
                const tid = img.getAttribute('data-tid');
                if (tid) {
                    fetch('/api/extra/' + tid)
                        .then(r => r.json())
                        .then(data => {
                            if (data.img) {
                                img.src = data.img;
                                img.classList.remove('lazy-load');
                            }
                            if (data.size && data.size !== '未知' && data.size !== '加载中...') {
                                const sizeSpan = document.getElementById('size-' + tid);
                                if (sizeSpan) sizeSpan.innerHTML = '📄 ' + data.size;
                            }
                            
                            const itemDiv = document.getElementById('item-' + tid);
                            if (itemDiv) {
                                if (data.group && !itemDiv.querySelector('.badge-group') && !itemDiv.querySelector('.badge-longweb')) {
                                    const statsRow = itemDiv.querySelector('.stats-row');
                                    if (statsRow) {
                                        if (data.group.toUpperCase() === 'LONGWEB') {
                                            statsRow.insertAdjacentHTML('beforeend', `<span class="badge-longweb">💎</span>`);
                                        } else {
                                            statsRow.insertAdjacentHTML('beforeend', `<span class="badge-group">${data.group}</span>`);
                                        }
                                    }
                                }
                                
                                // 💡【终极修复】：前端获取接口直接返回的全量智能数据组进行渲染！
                                const tagsWrapper = itemDiv.querySelector('.tags-wrapper');
                                if (tagsWrapper && data.top_pri) {
                                    let hasFree = !!itemDiv.querySelector('.tag-free');
                                    if (data.tags.some(t => t.toUpperCase() === 'FREE' || t === '免费') && !hasFree) {
                                        itemDiv.querySelector('.poster-wrapper').insertAdjacentHTML('beforeend', '<div class="tag-free">FREE</div>');
                                    }
                                    
                                    let html = '<div class="tags-top">';
                                    data.top_pri.forEach(t => html += `<span class="badge-tech">${t}</span>`);
                                    data.top_sec.forEach(t => html += `<span class="badge-tech secondary-tag">${t}</span>`);
                                    html += '</div><div class="tags-bottom">';
                                    
                                    if (data.time_tag) html += `<span class="badge-discount">${data.time_tag}</span>`;
                                    
                                    data.bot_disc.forEach(t => { 
                                        if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge-discount">${t}</span>`; 
                                    });
                                    data.bot_pri.forEach(t => { 
                                        if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge">${t}</span>`; 
                                    });
                                    data.bot_sec.forEach(t => html += `<span class="badge-lang secondary-tag">${t}</span>`);
                                    
                                    if (data.top_sec.length > 0 || data.bot_sec.length > 0) {
                                        html += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                    }
                                    html += '</div>';
                                    
                                    tagsWrapper.innerHTML = html;
                                }
                            }
                        })
                        .catch(e => console.error('懒加载失败:', e));
                }
            });
        });
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    tab = request.args.get('tab', 'recent')
    query = request.args.get('q', '')
    torrents = []
    user_info = {}
    seen_ids = set() 
    error_msg = None
    
    try:
        if tab == 'account':
            resp = requests.get(INDEX_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效，请在代码中更新 Cookie！"
            else:
                soup = BeautifulSoup(resp.text, 'html.parser')
                for tag in soup.find_all(True):
                    title_attr = tag.get('title') or tag.get('alt') or tag.get('data-original-title')
                    if title_attr and isinstance(title_attr, str) and len(title_attr) < 20:
                        tag.append(f"{title_attr}")
                page_text = soup.get_text(separator='', strip=True)
                
                user_match = re.search(r'欢迎回来[\s,，]*(.*?)(?:\[|退出|魔力值|分享率)', page_text)
                bonus_match = re.search(r'魔力值[^\d]*([\d\.,]+)', page_text)
                ratio_match = re.search(r'分享率[^\d]*([\d\.,]+|无限|Inf)', page_text, re.IGNORECASE)
                up_match = re.search(r'(?:上传量?|上传|上行)[^\d]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                down_match = re.search(r'(?:下载量?|下载|下行)[^\d]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                
                user_info = {
                    'username': user_match.group(1).strip() if user_match else "包子会员",
                    'bonus': bonus_match.group(1) if bonus_match else "0",
                    'ratio': ratio_match.group(1) if ratio_match else "0",
                    'upload': up_match.group(1) if up_match else "0 GB",
                    'download': down_match.group(1) if down_match else "0 GB"
                }

        elif tab in ['official', 'search']:
            if tab == 'search' and not query: pass
            else:
                if tab == 'search':
                    safe_query = urllib.parse.quote(query)
                    target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
                else: 
                    target_url = OFFICIAL_URL
                    
                torrents, error_msg = parse_torrents_page(target_url, seen_ids, is_search=(tab == 'search'))
                
                if torrents and os.path.exists(HISTORY_FILE):
                    try:
                        with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history = json.load(f)
                        history_dict = {str(item['id']): item for item in history}
                        for item in torrents:
                            tid_str = str(item['id'])
                            if tid_str in history_dict:
                                cached = history_dict[tid_str]
                                if 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                                if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                                if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                                merged_tags = purify_tags(item.get('tags', []))
                                for t in purify_tags(cached.get('tags', [])):
                                    if t not in merged_tags: merged_tags.append(t)
                                item['tags'] = purify_tags(merged_tags)
                    except: pass

                if tab == 'official' and torrents:
                    try:
                        history_dict = {}
                        if os.path.exists(HISTORY_FILE):
                            with open(HISTORY_FILE, 'r', encoding='utf-8') as f:
                                history_dict = {str(x['id']): x for x in json.load(f)}
                        for item in torrents:
                            tid_str = str(item['id'])
                            if tid_str not in history_dict:
                                history_dict[tid_str] = item
                        merged = list(history_dict.values())
                        merged.sort(key=lambda x: int(x['id']), reverse=True)
                        with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(merged[:50], f, ensure_ascii=False, indent=2)
                    except: pass

        elif tab == 'recent':
            fetched_torrents, err_msg = parse_torrents_page(OFFICIAL_URL, seen_ids, is_search=False)
            if fetched_torrents:
                torrents = []
                if os.path.exists(HISTORY_FILE):
                    try:
                        with open(HISTORY_FILE, 'r', encoding='utf-8') as f: torrents = json.load(f)
                    except: pass
                
                history_dict = {str(item['id']): item for item in torrents}
                for item in fetched_torrents:
                    tid_str = str(item['id'])
                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                        merged_tags = purify_tags(item.get('tags', []))
                        for t in purify_tags(cached.get('tags', [])):
                            if t not in merged_tags: merged_tags.append(t)
                        item['tags'] = purify_tags(merged_tags)
                    history_dict[tid_str] = item
                    
                merged = list(history_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = merged[:50]
                try:
                    with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(torrents, f, ensure_ascii=False, indent=2)
                except: pass
            elif err_msg:
                error_msg = err_msg

        if torrents:
            for item in torrents:
                tpri, tsec, bdisc, bpri, bsec = sort_tags(item['tags'])
                item['top_pri'] = tpri
                item['top_sec'] = tsec
                item['bot_disc'] = bdisc
                item['bot_pri'] = bpri
                item['bot_sec'] = bsec
                item['time_tag'] = ""
                
                for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                    for t in list(arr):
                        if '剩余' in t.upper():
                            item['time_tag'] = t
                            arr.remove(t)

    except Exception as e:
        error_msg = f"服务器内部请求错误: {e}"
        
    return render_template_string(HTML_TEMPLATE, tab=tab, query=query, torrents=torrents, user_info=user_info, error=error_msg)

# 💡【核心修复】：API接口采用全量合并数据直接返回完整排序组，让前端省心渲染不掉标！
@app.route('/api/extra/<int:tid>')
def api_extra(tid):
    img_src, size, extra_tags, release_group = fetch_details_info(tid)
    merged_tags = purify_tags(extra_tags)
    
    if os.path.exists(HISTORY_FILE):
        try:
            with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history = json.load(f)
            updated = False
            for item in history:
                if str(item['id']) == str(tid):
                    if img_src: item['img'] = img_src
                    if size and size != "未知": item['size'] = size
                    if release_group and not item.get('group'): item['group'] = release_group
                    
                    new_tags = purify_tags(item.get('tags', []))
                    for t in merged_tags:
                        if t not in new_tags: new_tags.append(t)
                    item['tags'] = new_tags
                    merged_tags = new_tags  # 使用完整融合的历史标签！
                    
                    updated = True
                    break
            if updated:
                with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
        except: pass
        
    tpri, tsec, bdisc, bpri, bsec = sort_tags(merged_tags)
    time_tag = ""
    for arr in [tpri, tsec, bdisc, bpri, bsec]:
        for t in list(arr):
            if '剩余' in t.upper():
                time_tag = t
                arr.remove(t)
    
    return jsonify({
        "img": img_src, 
        "size": size, 
        "tags": merged_tags,
        "top_pri": tpri,
        "top_sec": tsec,
        "bot_disc": bdisc,
        "bot_pri": bpri,
        "bot_sec": bsec,
        "time_tag": time_tag,
        "group": release_group
    })

@app.route('/api/signin')
def api_signin():
    try:
        r = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        soup_signin = BeautifulSoup(r.text, 'html.parser')
        clean_text = soup_signin.get_text(separator=' ', strip=True)
        
        match = re.search(r'(这是您的第.*?魔力值|签到成功.*?魔力值)', clean_text)
        if match:
            return jsonify({"msg": "🎉 签到成功！\n" + match.group(1).replace('  ', ' ')})
        elif "已经" in clean_text or "退下" in clean_text or "已签到" in clean_text: 
            return jsonify({"msg": "☕ 今天已经签到过了，明天再来吧。"})
        elif "登录" in clean_text: 
            return jsonify({"msg": "⚠️ 签到失败：Cookie 已失效，请更新！"})
        else: 
            return jsonify({"msg": "🤔 状态未知，可能已签到，请留意积分变化。"})
    except Exception as e: 
        return jsonify({"msg": f"❌ 网络请求出错: {str(e)}"})

@app.route('/download/<int:tid>')
def download_torrent(tid):
    try:
        r = requests.get(f"{BASE_URL}/download.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        if "application/x-bittorrent" in r.headers.get('Content-Type', '') or "octet-stream" in r.headers.get('Content-Type', ''):
            return Response(r.content, mimetype="application/x-bittorrent", headers={"Content-Disposition": f"attachment;filename=BaoziPT_{tid}.torrent"})
        else: return "下载失败，Cookie 可能失效或无权下载该种子。"
    except Exception as e: return f"请求下载出错: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5052)
```
6.h5long.py

每天记录签到、立即参与、15分钟刷新、500条缓存记录

```
from flask import Flask, render_template_string, Response, request, jsonify
import requests
from bs4 import BeautifulSoup
import urllib.parse
import re
import os
import json
import threading
import time
import datetime
import random

# 启动时自动剿灭被龙头像污染的旧缓存文件
for f in ["longpt_official_history.json", "longpt_recent_history.json"]:
    if os.path.exists(f):
        try: os.remove(f)
        except: pass

app = Flask(__name__)

# ================= 必须配置的区域 =================
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    # 【安全提醒】：请把你的 LongPT 长效 Cookie 填在下面
    'Cookie': 'SITE_TOTAL_ID=b06a69304be7b561a043e0a5854d8e72; c_secure_pass=eyJ1c2VyX2lkIjoiMzQyODMiLCJleHBpcmVzIjoxODEzNDk1MjQyfS5lZjAyNTU4NDlkMzlkYjZhNzM4NmM5NTUzZTNlMWFjM2Q4M2VlYjE3NTUwNjI1ZjE3NjRkMzAwNjlmZTlmM2Zj; server_name_session=54eb12fe05fd05ddaaaedf7596b64a03; cf_clearance=q8.jsFZV.ZXKRO2YXJlJqX3ooETUkVOchY7OxTIoARI-1790686222-1.2.1.1-nF9k6YHJS6Ir04vlZvI5Z5RoY.vfJn3WShGqnkO_ah52LmrYTHYxYSS_SJDJOQ2x0kEqPKdPG.C5Xz8Y3aRe37Z61L_6kL_bIRI3Lc91M6CUjySriI38lUmZpwmv6tJn4WmQKvbZVaxpoS1UOD1_vCLNWHm41WdaVs5VH62iTuT9FuefQfLpCaj.SFAQuFECLU16MsYPybyIiw1QK8ZqLUFrjWCwjrtcEV_Nt1C5c1v4diyRDCqvKd3OU9xHHieZoC8DWBkPc2V98d77.xAwgXLiu2KhujJk5kvVZNp1xmHmACPquBhP9yTK_zE_2F86MEpPqzpxckFm5xM0aV_Cp4fQdDNNbEjzBESu65a6dr7zwTx4kenOTDsaImoHDWwyC9ylHDRgxw4.ANMc8hibcyG3Pj3XELHRShyyBV0q.4VlEJJNG2foaSHQgx0AGAKV' 
}

PROXY_URL = ""
PROXIES = {"http": PROXY_URL, "https": PROXY_URL} if PROXY_URL else None
TIMEOUT = 30

BASE_URL = "https://longpt.org"
OFFICIAL_URL = f"{BASE_URL}/torrents.php"
INDEX_URL = f"{BASE_URL}/index.php"
SIGNIN_URL = f"{BASE_URL}/attendance.php"

OFFICIAL_HISTORY_FILE = "longpt_official_history.json"
RECENT_HISTORY_FILE = "longpt_recent_history.json"
# =================================================

MEM_CACHE_OFFICIAL = {}
MEM_CACHE_RECENT = {}
CACHE_LOCK = threading.Lock()

def load_cache(cache_type='official'):
    global MEM_CACHE_OFFICIAL, MEM_CACHE_RECENT
    filename = OFFICIAL_HISTORY_FILE if cache_type == 'official' else RECENT_HISTORY_FILE
    target_cache = MEM_CACHE_OFFICIAL if cache_type == 'official' else MEM_CACHE_RECENT
    
    with CACHE_LOCK:
        if not target_cache and os.path.exists(filename):
            try:
                with open(filename, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    res = {str(item['id']): item for item in data}
                    if cache_type == 'official': MEM_CACHE_OFFICIAL = res
                    else: MEM_CACHE_RECENT = res
                    return res
            except Exception: pass
    return target_cache

def save_cache(cache_dict, cache_type='official'):
    global MEM_CACHE_OFFICIAL, MEM_CACHE_RECENT
    filename = OFFICIAL_HISTORY_FILE if cache_type == 'official' else RECENT_HISTORY_FILE
    with CACHE_LOCK:
        if cache_type == 'official': MEM_CACHE_OFFICIAL = cache_dict.copy()
        else: MEM_CACHE_RECENT = cache_dict.copy()
        
        merged = list(cache_dict.values())
        merged.sort(key=lambda x: int(x['id']), reverse=True)
        try:
            with open(filename, 'w', encoding='utf-8') as f:
                json.dump(merged[:500], f, ensure_ascii=False, indent=2)
        except Exception: pass

TECH_KEYWORDS = [
    'FREE', '50%', '30%', '2X', '热门', '官方', '中字', '英字', '国语', '粤语', '英语', 
    '简繁', '双语', '杜比', '高帧', '高码', '臻彩MAX', '高分', '完结', '4K', '2160P', '1440P', '1080P', '720P', 
    'WEB-DL', 'BLU-RAY', 'REMUX', 'HDR10+', 'HDR10', 'HDRVIVID', 
    'HDR', 'DOVI', 'DV', 'SDR', '60FPS', 'HFR', 'HQ', 'H.265', 'H.264', 'H265', 'H264', 
    'HEVC', 'AAC', 'DDP', 'ATMOS', '10BIT', '首发', '禁转', '应求', 'DIY', '字幕'
]

def purify_tags(tags):
    valid = []
    for t in tags:
        clean_t = t.upper()
        if '剩余' in clean_t or '🕒' in clean_t:
            valid.append(t)
        elif clean_t in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', clean_t):
            valid.append(t)
            
    final_tags = []
    time_tags = [t for t in valid if '剩余' in t]
    best_time = max(time_tags, key=len) if time_tags else None
    
    clock_tags = [t for t in valid if '🕒' in t]
    best_clock = clock_tags[-1] if clock_tags else None
    
    if best_time and best_clock:
        pure_time = re.sub(r'^.*?(?:剩余时间|免费剩余)[:：\s]*', '', best_time).strip()
        pure_clock = best_clock.replace('🕒', '').strip()
        if pure_time == pure_clock:
            best_clock = None
            
    for t in valid:
        if '剩余' in t:
            if t == best_time and best_time not in final_tags:
                if not re.match(r'^剩余时间[:：\s]*\d+\s*分$', t):
                    final_tags.append(t)
        elif '🕒' in t:
            if t == best_clock and best_clock not in final_tags:
                final_tags.append(t)
        elif t not in final_tags:
            final_tags.append(t)
            
    return final_tags

def merge_tags_list(list_tags, cached_tags):
    res = list(list_tags)
    for t in cached_tags:
        t_up = t.upper()
        is_dynamic = '%' in t_up or '2X' in t_up or t_up in ['FREE', '免费'] or '剩余' in t_up
        if not is_dynamic and t not in res:
            res.append(t)
    return res

def merge_tags_details(details_tags, cached_tags):
    res = list(details_tags)
    has_discount = any('%' in x.upper() or '2X' in x.upper() for x in res)
    has_free = any(x.upper() in ['FREE', '免费'] for x in res)
    has_time = any('剩余' in x for x in res)
    has_clock = any('🕒' in x for x in res)
    
    for t in cached_tags:
        if t in res: continue
        t_up = t.upper()
        if '%' in t_up or '2X' in t_up:
            if not has_discount: res.append(t)
        elif t_up in ['FREE', '免费']:
            if not has_free: res.append(t)
        elif '剩余' in t_up:
            if not has_time: res.append(t)
        elif '🕒' in t_up:
            if not has_clock: res.append(t)
        else:
            res.append(t)
    return res

def auto_extract_tags(text, current_tags):
    en_matches = re.findall(r'(?i)\b(4K|2160P|1440P|1080P|720P|WEB-DL|BLU-RAY|REMUX|HDR10\+|HDR10|HDRVIVID|HDR|DOVI|DV|SDR|60FPS|HFR|HQ|H\.?265|H\.?264|HEVC|AAC|DDP|ATMOS|10BIT)\b', text)
    cn_matches = re.findall(r'(杜比|高帧|高码|臻彩MAX|高分|完结|国语|中字|英字|粤语|英语|简繁|双语|官方|首发)', text)
    disc_matches = re.findall(r'(50%|30%|2X)', text.upper())
    
    all_new = [m.upper() for m in en_matches] + cn_matches + disc_matches
    for tag in all_new:
        tag_up = tag.upper()
        if not any(tag_up in t.upper() or t.upper() in tag_up for t in current_tags):
            current_tags.append(tag)
    return current_tags

def sort_tags(tags):
    top_pri, top_sec, bot_disc, bot_pri, bot_sec = [], [], [], [], []
    remaining = list(tags)
    
    t_res = ['4K', '2160P', '1440P', '1080P', '720P']
    t_hdr = ['HDRVIVID', 'HDR10+', 'HDR10', 'HDR', 'DOVI', 'DV', 'SDR']
    t_fps = ['60FPS', 'HFR', '高帧', '臻彩MAX', 'BLU-RAY', 'REMUX']
    t_enc = ['WEB-DL', 'H.265', 'H.264', 'H265', 'H264', 'HEVC', 'AAC', 'DDP', 'ATMOS', '杜比', '10BIT', 'HQ', '高码']
    
    for t in list(remaining):
        t_upper = t.upper()
        if '%' in t_upper or '2X' in t_upper or t_upper in ['免费', 'FREE', '30%', '50%']:
            if t_upper not in ['FREE', '免费']:
                bot_disc.append(t)
            remaining.remove(t)
    
    t_status = ['热门', '完结', '首发', '高分', 'DIY']
    t_lang = ['国语', '中字', '英字', '粤语', '英语', '简繁', '双语', '官方', '字幕']
    
    for kw in t_res + t_hdr + t_fps:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_pri.append(t)
                remaining.remove(t)
                
    for kw in t_enc:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_sec.append(t)
                remaining.remove(t)
                
    for kw in t_status:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_pri.append(t)
                remaining.remove(t)
                
    for kw in t_lang:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_sec.append(t)
                remaining.remove(t)
                
    for t in list(remaining):
        bot_pri.append(t)
        
    return top_pri, top_sec, bot_disc, bot_pri, bot_sec

def fetch_details_info(tid):
    try:
        r = requests.get(f"{BASE_URL}/details.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        raw_html = r.text
        soup = BeautifulSoup(raw_html, 'html.parser')
        extra_tags = []

        time_match = re.search(r'(?:免费剩余时间|免费剩余|优惠剩余时间|剩余时间|还有|剩余)[^\d<]{0,30}(?:<[^>]+>)*\s*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', raw_html)
        if time_match:
            extra_tags.append(f"剩余时间: {time_match.group(1).strip()}")
        else:
            text_clean = soup.get_text(separator=' ')
            tm2 = re.search(r'(?:免费剩余时间|免费剩余|优惠剩余时间|剩余时间|还有|剩余)[:：\s]+(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', text_clean)
            if tm2: extra_tags.append(f"剩余时间: {tm2.group(1).strip()}")

        img_src = ""
        descr_div = soup.find('div', id='kdescr')
        if descr_div:
            img = descr_div.find('img')
            if img: 
                img_src = img.get('data-src') or img.get('src') or ""
                if img_src and not img_src.startswith('http'):
                    img_src = f"{BASE_URL}/" + img_src

        size = "未知"
        size_match = re.search(r'大小.*?([\d\.]+\s*[KMGTP]B)', raw_html, re.IGNORECASE)
        if size_match: size = size_match.group(1)

        h1 = soup.find('h1')
        release_group = ""
        if h1:
            h1_text = h1.get_text(strip=True)
            for kw in ['免费', 'FREE', '50%', '30%', '2X']:
                if kw in h1_text.upper():
                    if kw in ['免费', 'FREE']:
                        if 'Free' not in extra_tags: extra_tags.append('Free')
                    else:
                        if kw not in extra_tags: extra_tags.append(kw)
            
            for span in h1.find_all('span', class_=re.compile(r'tag|label|badge', re.I)):
                text = span.get_text(strip=True).replace('[', '').replace(']', '').strip()
                if text.upper() in ['FREE', '免费'] and 'Free' not in extra_tags: extra_tags.append('Free')
                elif text and text not in extra_tags and len(text) <= 15: extra_tags.append(text)
                    
            group_match = re.search(r'-([A-Za-z0-9@_]+)$', h1_text)
            if group_match: release_group = group_match.group(1).upper()

        sub_td = soup.find('td', class_='rowhead', string=re.compile('副标题|副標題'))
        sub_text = ""
        if sub_td:
            sub_text = sub_td.find_next_sibling('td').get_text(strip=True)
            
        extra_tags = auto_extract_tags(h1_text + " " + sub_text, extra_tags)
        
        page_clean_text = soup.get_text(separator=' ')
        pub_exact = re.search(r'(?:发布时间|添加时间|上传时间|创建时间)[^\d]*(\d{4}-\d{2}-\d{2}\s*\d{2}:\d{2}(?::\d{2})?)', page_clean_text)
        if pub_exact:
            extra_tags.append(f"🕒 {pub_exact.group(1)}")
        else:
            pub_rel = re.search(r'发布于\s*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)\s*前', page_clean_text)
            if pub_rel:
                extra_tags.append(f"🕒 {re.sub(r'\s+', '', pub_rel.group(1))}前")
        
        return img_src, size, purify_tags(extra_tags), release_group
    except Exception as e:
        return "", "未知", [], ""

def is_real_sticky(row):
    for img in row.find_all('img'):
        src = img.get('src', '').lower()
        alt = img.get('alt', '').lower()
        title = img.get('title', '').lower()
        cls = ' '.join(img.get('class', [])).lower()
        if 'sticky' in cls: return True
        if alt in ['sticky', '置顶'] or title in ['sticky', '置顶']: return True
        if any(x in src for x in ['sticky.gif', 'pin.gif', 'pro_sticky.gif']): return True
    for tag in row.find_all(['span', 'font', 'b']):
        t = tag.get_text(strip=True).upper()
        if t in ['[置顶]', '置顶', '[STICKY]', 'STICKY']: return True
        if 'sticky' in ' '.join(tag.get('class', [])).lower(): return True
    tr_cls = ' '.join(row.get('class', [])).lower()
    if any(x in tr_cls for x in ['sticky_bg', 'sticky_blank', 'pro_sticky']): return True
    return False

def parse_torrents_page(target_url_base, seen_ids, is_search=False, skip_sticky=False, start_page=0, max_pages=1):
    torrents = []
    try:
        for page in range(start_page, start_page + max_pages):
            url = f"{target_url_base}&page={page}" if "?" in target_url_base else f"{target_url_base}?page={page}"
            resp = requests.get(url, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            
            if "登录" in resp.text and "torrents.php" not in resp.url: return None, "⚠️ Cookie 已失效！请更新。"
            if resp.status_code != 200: break

            soup = BeautifulSoup(resp.text, 'html.parser')
            for row in soup.find_all('tr'):
                if skip_sticky and is_real_sticky(row): continue

                tds = row.find_all('td')
                if len(tds) < 6: continue
                title_a = row.find('a', href=lambda h: h and h.startswith('details.php?id='))
                if not title_a: continue
                
                tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                title = title_a.get('title') or title_a.get_text(strip=True)
                if not title and title_a.find('b'): title = title_a.find('b').get_text(strip=True)
                if not title or tid in seen_ids: continue
                seen_ids.add(tid)
                link = f"{BASE_URL}/" + title_a['href']
                
                release_group = ""
                group_match = re.search(r'-([A-Za-z0-9@_]+)$', title.strip())
                if group_match: release_group = group_match.group(1).upper()
                
                try: 
                    size = tds[-5].get_text(strip=True)
                    seeders = tds[-4].get_text(strip=True)
                    leechers = tds[-3].get_text(strip=True)
                    time_col_td = tds[-6]
                except: 
                    size, seeders, leechers, time_col_td = "未知", "-", "-", None

                img_src = ""
                td = title_a.find_parent('td')
                tags = []
                
                if time_col_td:
                    time_span = time_col_td.find('span', title=re.compile(r'\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}'))
                    t_text = time_span.get_text(separator='', strip=True) if time_span else time_col_td.get_text(separator='', strip=True)
                    if t_text and any(x in t_text for x in ['年', '月', '周', '天', '时', '分', '秒']):
                        clean_t = re.sub(r'\s+', '', t_text)
                        if not clean_t.endswith('前') and not re.search(r'\d{4}-\d{2}-\d{2}', clean_t):
                            clean_t += '前'
                        tags.append(f"🕒 {clean_t}")
                    elif re.search(r'\d{4}-\d{2}-\d{2}', t_text):
                        tags.append(f"🕒 {t_text.strip()}")
                
                subtitle = ""
                if td:
                    raw_html = str(td)
                    
                    if not any('🕒' in t for t in tags):
                        t_match = re.search(r'\b(20\d{2}-\d{2}-\d{2}\s+\d{2}:\d{2}(?::\d{2})?)\b', raw_html)
                        if t_match: tags.append(f"🕒 {t_match.group(1)}")

                    time_matches = re.finditer(r'(?:免费剩余时间|免费剩余|剩余时间|剩余)[:：\s]*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', raw_html)
                    for tm in time_matches: tags.append(f"剩余时间: {tm.group(1).strip()}")

                    for icon in td.find_all('img'):
                        alt = str(icon.get('alt') or '')
                        title_attr = str(icon.get('title') or '')
                        combined_text = alt + " " + title_attr
                        if '免费' in combined_text or 'Free' in combined_text:
                            if 'Free' not in tags: tags.append('Free')
                        if icon.get('src', '') != img_src: 
                            icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                    
                    for span in td.find_all('span'):
                        text = span.get_text(strip=True)
                        title_attr = str(span.get('title') or '')
                        combined = (text + " " + title_attr).upper()
                        if 'FREE' in combined or '免费' in combined:
                            if 'Free' not in tags: tags.append('Free')
                        for disc in ['50%', '30%', '2X']:
                            if disc in combined and disc not in tags: tags.append(disc)
                                
                        text_clean = text.replace('[', '').replace(']', '').strip()
                        if not text_clean: continue
                        span_class = ' '.join(span.get('class', [])).lower()
                        if bool(re.search(r'tag|label|badge', span_class)) or text_clean.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', text_clean):
                            if text_clean not in tags: tags.append(text_clean)
                            span.replace_with(soup.new_tag("span", string=f" [{text_clean}] "))
                    
                    subtitle = td.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                    subtitle = re.sub(r'(?i)\[?(?:Sticky|置顶|poster)\]?[\s\|]*', '', subtitle)
                    
                    inline_time = re.search(r'\[?(?:剩余时间)[:：\s]*\]?[\s\|]*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', subtitle)
                    if inline_time:
                        t_str = f"剩余时间: {inline_time.group(1).strip()}"
                        if t_str not in tags: tags.append(t_str)
                        subtitle = subtitle.replace(inline_time.group(0), '')
                    
                    clean_sub = subtitle
                    for m in re.finditer(r'\[(.*?)\]', subtitle):
                        tag = m.group(1).strip(' |[]')
                        if tag.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', tag) or tag.upper() in ['热门']:
                            if tag not in tags: tags.append(tag)
                            clean_sub = clean_sub.replace(m.group(0), '')
                        else: clean_sub = clean_sub.replace(m.group(0), tag)
                    
                    clean_sub = re.sub(r'热门|免费', '', clean_sub)
                    parts = [p.strip() for p in clean_sub.split('|') if p.strip()]
                    kept_sub = []
                    for p in parts:
                        clean_p = re.sub(r'^[()\[\]\-|\s]+|[()\[\]\-|\s]+$', '', p)
                        if not clean_p: continue
                        if clean_p.upper() in [k.upper() for k in TECH_KEYWORDS]:
                            if clean_p not in tags: tags.append(clean_p)
                        else: kept_sub.append(p)
                    subtitle = ' | '.join(kept_sub)
                
                tags = auto_extract_tags(title + " " + subtitle, tags)
                tags = purify_tags(tags)

                torrents.append({
                    'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                    'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags, 'group': release_group
                })
                
                if len(torrents) >= 50 and max_pages > 1: break
            if len(torrents) >= 50 and max_pages > 1: break
        return torrents, None
    except Exception as e:
        return None, f"网络请求异常: {str(e)}"

def parse_index_page(seen_ids):
    torrents = []
    try:
        resp = requests.get(INDEX_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        if "登录" in resp.text and "index.php" not in resp.url: return None, "⚠️ Cookie 已失效！"
        
        if resp.status_code == 200:
            soup = BeautifulSoup(resp.text, 'html.parser')
            cached_recent = load_cache('recent')
            
            for title_a in soup.find_all('a', href=lambda h: h and h.startswith('details.php?id=')):
                tid_m = re.search(r'id=(\d+)', title_a['href'])
                if not tid_m: continue
                tid = tid_m.group(1)
                if tid in seen_ids: continue
                
                title = title_a.get('title') or title_a.get_text(strip=True)
                if not title and title_a.find('img'): title = title_a.find('img').get('alt') or title_a.find('img').get('title')
                if not title: continue
                seen_ids.add(tid)
                
                link = f"{BASE_URL}/details.php?id={tid}"
                container = title_a.find_parent('tr') or title_a.find_parent('div')
                
                img_src, subtitle, seeders, leechers, size = "", "", "-", "-", "加载中..."
                tags = []
                release_group = ""
                
                group_match = re.search(r'-([A-Za-z0-9@_]+)$', title.strip())
                if group_match: release_group = group_match.group(1).upper()
                    
                if container:
                    if container.name == 'tr':
                        tds = container.find_all('td')
                        if len(tds) >= 3:
                            seeders, leechers = tds[-2].get_text(strip=True), tds[-1].get_text(strip=True)
                            for td in tds:
                                txt = td.get_text(strip=True)
                                if re.match(r'^\d+(\.\d+)?\s*[KMGTP]B$', txt, re.I):
                                    size = txt
                                    break
                            
                    subtitle = container.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                    if seeders != "-" and subtitle.endswith(leechers): subtitle = subtitle.rsplit('|', 2)[0].strip(' | ')
                    subtitle = re.sub(r'(?i)\[?(?:Sticky|置顶|poster)\]?[\s\|]*', '', subtitle)
                    subtitle = re.sub(r'(?i)^\s*\[?新\]?\s*\|\s*', '', subtitle)

                    text_clean = container.get_text(separator=' ', strip=True)
                    t_match = re.search(r'\b(20\d{2}-\d{2}-\d{2}\s+\d{2}:\d{2}(?::\d{2})?)\b', text_clean)
                    if t_match: tags.append(f"🕒 {t_match.group(1)}")
                    else:
                        rel_m = re.search(r'(?:发布于)?\s*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)\s*前', text_clean)
                        if rel_m: tags.append(f"🕒 {re.sub(r'\s+', '', rel_m.group(1))}前")

                tags = auto_extract_tags(title + " " + subtitle, tags)
                
                tid_str = str(tid)
                if tid_str in cached_recent:
                    cached_item = cached_recent[tid_str]
                    # 💡【超级护盾】：彻底守护首页第三方种子的 FREE 和 剩余时间，绝不让 index.php 刷没！
                    tags = merge_tags_details(tags, cached_item.get('tags', []))
                    if cached_item.get('img') and 'data:image/svg+xml' not in cached_item['img']: img_src = cached_item['img']
                    if cached_item.get('size') and cached_item.get('size') not in ['未知', '加载中...']: size = cached_item['size']
                
                tags = purify_tags(tags)
                torrents.append({
                    'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                    'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags, 'group': release_group
                })
            return torrents, None
        return None, f"获取最新种子失败，状态码: {resp.status_code}"
    except Exception as e:
        return None, f"网络请求异常: {str(e)}"

def background_monitor():
    last_sign_date = None
    last_draw_date = None
    while True:
        try:
            current_time = datetime.datetime.now()
            current_date_str = current_time.strftime("%Y-%m-%d")
            
            if last_sign_date != current_date_str:
                r_sign = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
                r_sign.encoding = 'utf-8'
                if any(kw in r_sign.text for kw in ["已经签到", "请明天再来", "签到成功", "魔力值", "次签到", "退下", "已经", "这是您的第"]):
                    last_sign_date = current_date_str

            if last_draw_date != current_date_str:
                h = current_time.hour
                m = current_time.minute
                if (h == 0 and m >= 45) or (h == 1 and m <= 30):
                    time.sleep(random.randint(1, 300))
                    try:
                        requests.get(f"{BASE_URL}/pt-api/v1/lucky/pluginsPrizeReceiptRecord/join", headers=HEADERS, proxies=PROXIES, timeout=5)
                        last_draw_date = current_date_str
                    except: pass
        except Exception: pass

        try:
            current_off, err = parse_torrents_page(OFFICIAL_URL, set(), skip_sticky=False, max_pages=2)
            if current_off:
                off_dict = load_cache('official')
                for item in current_off[:5]:
                    tid_str = str(item['id'])
                    if tid_str not in off_dict or not off_dict[tid_str].get('img') or 'data:image/svg+xml' in off_dict[tid_str]['img'] or off_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags, det_group = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_group and not item.get('group'): item['group'] = det_group
                        if det_tags:
                            item['tags'] = purify_tags(merge_tags_details(det_tags, item.get('tags', [])))

                    if tid_str in off_dict:
                        cached = off_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if not item.get('group') and cached.get('group'): item['group'] = cached['group']
                        item['tags'] = purify_tags(merge_tags_list(item.get('tags', []), cached.get('tags', [])))
                        
                    off_dict[tid_str] = item
                save_cache(off_dict, 'official')
        except Exception: pass

        try:
            current_rec, err = parse_index_page(set())
            if current_rec:
                rec_dict = load_cache('recent')
                for item in current_rec:
                    tid_str = str(item['id'])
                    if tid_str not in rec_dict or not rec_dict[tid_str].get('img') or 'data:image/svg+xml' in rec_dict[tid_str]['img'] or rec_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags, det_group = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_group and not item.get('group'): item['group'] = det_group
                        if det_tags:
                            item['tags'] = purify_tags(merge_tags_details(det_tags, item.get('tags', [])))

                    if tid_str in rec_dict:
                        cached = rec_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if not item.get('group') and cached.get('group'): item['group'] = cached['group']
                        # 💡【超级护盾】：后台监控刷新 index.php 也要带上护盾！
                        item['tags'] = purify_tags(merge_tags_details(item.get('tags', []), cached.get('tags', [])))
                        
                    rec_dict[tid_str] = item
                save_cache(rec_dict, 'recent')
        except Exception: pass

        time.sleep(900)

threading.Thread(target=background_monitor, daemon=True).start()

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>LongPT 手机端</title>
    <style>
        :root { --bg-color: #f7f7f7; --card-bg: #ffffff; --text-main: #333333; --text-sub: #888888; --border-color: #eeeeee; --topbar-bg: #ffffff; --icon-bg: #f0f0f5; --primary-color: #2563eb; }
        [data-theme="dark"] { --bg-color: #121212; --card-bg: #1e1e1e; --text-main: #eeeeee; --text-sub: #aaaaaa; --border-color: #333333; --topbar-bg: #1e1e1e; --icon-bg: #333333; --primary-color: #3b82f6; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background-color: var(--bg-color); color: var(--text-main); margin: 0; padding: 0; padding-top: 55px; transition: background-color 0.3s; }
        
        .top-bar { position: fixed; top: 0; left: 0; right: 0; height: 50px; background: var(--topbar-bg); display: flex; align-items: center; justify-content: space-between; padding: 0 15px; box-shadow: 0 1px 3px rgba(0,0,0,0.05); z-index: 100; transition: background 0.3s; }
        .menu-btn { width: 22px; height: 16px; position: relative; cursor: pointer; display: flex; flex-direction: column; justify-content: space-between; padding: 0; }
        .menu-btn span { display: block; height: 2px; width: 100%; background: var(--text-main); border-radius: 2px; transition: all 0.3s ease; }
        .menu-btn.open span:nth-child(1) { transform: translateY(7px) rotate(45deg); }
        .menu-btn.open span:nth-child(2) { opacity: 0; }
        .menu-btn.open span:nth-child(3) { transform: translateY(-7px) rotate(-45deg); }

        .logo { font-weight: 800; font-size: 20px; color: var(--text-main); letter-spacing: 1px; }
        .top-placeholder { width: 22px; }

        .sidebar { position: fixed; top: 0; left: -260px; bottom: 0; width: 260px; background: var(--card-bg); z-index: 1001; transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: 2px 0 10px rgba(0,0,0,0.1); padding-top: 60px; display: flex; flex-direction: column; }
        .sidebar.open { left: 0; }
        .menu-item { padding: 16px 24px; font-size: 16px; font-weight: 500; color: var(--text-main); text-decoration: none; display: block; border-bottom: 1px solid var(--border-color); }
        .menu-item.active { color: var(--primary-color); border-left: 4px solid var(--primary-color); padding-left: 20px; background: rgba(37,99,235,0.05); }
        .theme-toggle-icon { margin: auto auto 30px 24px; width: 40px; height: 40px; border-radius: 20px; background: var(--icon-bg); display: flex; align-items: center; justify-content: center; font-size: 20px; cursor: pointer; user-select: none; transition: transform 0.2s; }
        .theme-toggle-icon:active { transform: scale(0.9); }
        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 1000; opacity: 0; pointer-events: none; transition: opacity 0.3s; }
        .overlay.open { opacity: 1; pointer-events: auto; }

        .container { padding: 5px 10px 10px 10px; }
        
        .global-search-container { display: flex; gap: 8px; margin: 5px 5px 15px 5px; align-items: center; background: var(--card-bg); padding: 5px; border-radius: 20px; border: 1px solid var(--border-color); box-shadow: 0 2px 8px rgba(0,0,0,0.02); }
        .search-input { flex: 1; padding: 8px 15px; border: none; outline: none; font-size: 14px; background: transparent; color: var(--text-main); }
        .search-btn { background: rgba(37,99,235,0.1); color: var(--primary-color); border: none; width: 36px; height: 36px; border-radius: 18px; font-weight: bold; font-size: 16px; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s ease; flex-shrink: 0; }
        .search-btn:active { background: var(--primary-color); color: #fff; }
        [data-theme="dark"] .search-btn { background: rgba(59,130,246,0.15); }
        
        .list-item { display: flex; gap: 12px; padding: 15px 5px; border-bottom: 1px solid var(--border-color); }
        .poster-wrapper { position: relative; flex-shrink: 0; width: 100px; height: 140px; }
        .poster { width: 100%; height: 100%; object-fit: cover; border-radius: 8px; background-color: #e0e0e0; transition: opacity 0.5s ease-in-out; }
        .tag-free { position: absolute; top: 0; left: 0; background: #22c55e; color: #fff; font-size: 10px; font-weight: bold; padding: 2px 6px; border-radius: 8px 0 8px 0; z-index: 10; }
        
        .info { flex-grow: 1; display: flex; flex-direction: column; overflow: hidden; justify-content: space-between; padding-bottom: 2px; }
        .title { font-size: 15px; font-weight: bold; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; line-height: 1.3; }
        .subtitle { font-size: 12px; color: var(--text-sub); line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; margin-bottom: 4px; }
        .subtitle:empty { display: none; margin: 0; }
        
        .stats-and-download { display: flex; justify-content: space-between; align-items: center; gap: 10px; width: 100%; margin-bottom: 4px; }
        .stats-row { font-size: 12px; color: var(--text-sub); display: flex; align-items: center; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; flex: 1; -webkit-overflow-scrolling: touch; }
        .stats-row::-webkit-scrollbar { display: none; }
        .stat-item { display: flex; align-items: center; gap: 2px; }
        .seed-icon { color: #22c55e; font-size: 10px; }
        .leech-icon { color: #ef4444; font-size: 10px; }
        
        .tags-wrapper { display: flex; flex-direction: column; gap: 6px; margin-top: auto; width: 100%; overflow: hidden; padding-top: 2px; }
        .tags-top { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; }
        .tags-top::-webkit-scrollbar { display: none; }
        .tags-top:empty { display: none; }
        
        .tags-bottom { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; width: 100%; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; padding-bottom: 2px; }
        .tags-bottom::-webkit-scrollbar { display: none; }
        .tags-bottom:empty { display: none; }
        
        .secondary-tag { display: none !important; }
        .show-extra .secondary-tag { display: inline-flex !important; }
        
        .badge-toggle { background: #e5e7eb; color: #6b7280; font-size: 10px; padding: 2px 8px; border-radius: 4px; flex-shrink: 0; cursor: pointer; display: inline-flex; align-items: center; letter-spacing: 1px; font-weight: bold; transition: all 0.2s ease;}
        [data-theme="dark"] .badge-toggle { background: #374151; color: #9ca3af; }
        .badge-toggle:active { opacity: 0.7; }
        
        .badge { background: #8b5cf6; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-discount { background: #ef4444; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-tech { background: #0ea5e9; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-lang { background: #10b981; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-group { background: #ec4899; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; letter-spacing: 0.5px;} 
        .badge-longweb { background: linear-gradient(135deg, #2563eb 0%, #06b6d4 100%); color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; letter-spacing: 0.5px; box-shadow: 0 1px 3px rgba(37,99,235,0.2);}

        .download-btn { background: var(--primary-color); color: #fff; font-size: 10px; padding: 2px 8px; border-radius: 4px; font-weight: bold; text-decoration: none; flex-shrink: 0; display: inline-flex; align-items: center; letter-spacing: 1px; min-width: 36px; justify-content: center;}
        .download-btn:active { opacity: 0.8; }
        
        .account-box { background: var(--card-bg); border-radius: 12px; padding: 20px; text-align: center; border: 1px solid var(--border-color); }
        .avatar { width: 70px; height: 70px; background: var(--primary-color); color: #fff; border-radius: 35px; line-height: 70px; font-size: 24px; margin: 0 auto 10px; }
        .account-name { font-size: 18px; font-weight: bold; }
        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }
        .stat-val { font-size: 18px; font-weight: bold; color: var(--primary-color); margin-bottom: 4px; }
        .stat-lbl { font-size: 12px; color: var(--text-sub); }
        .btn-signin { background: #22c55e; color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 25px; border: none; cursor: pointer; }
        .btn-lottery { background: linear-gradient(135deg, #8b5cf6 0%, #6366f1 100%); color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 15px; border: none; cursor: pointer; box-shadow: 0 4px 6px rgba(139, 92, 246, 0.3); transition: transform 0.2s;}
        .btn-lottery:active { transform: scale(0.98); }
    </style>
</head>
<body>
    <script>
        function setTheme(theme) {
            const html = document.documentElement;
            if (theme === 'dark') { html.setAttribute('data-theme', 'dark'); }
            else if (theme === 'light') { html.removeAttribute('data-theme'); }
            else { 
                if (window.matchMedia('(prefers-color-scheme: dark)').matches) html.setAttribute('data-theme', 'dark');
                else html.removeAttribute('data-theme');
            }
        }
        let currentTheme = localStorage.getItem('longpt_theme') || 'auto';
        setTheme(currentTheme);
        window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
            if (localStorage.getItem('longpt_theme') === 'auto') setTheme('auto');
        });
        
        window.toggleTags = function(btn) {
            const wrapper = btn.closest('.tags-wrapper');
            wrapper.classList.toggle('show-extra');
            if (wrapper.classList.contains('show-extra')) {
                btn.innerText = '收起 ▴';
            } else {
                btn.innerText = '... ▾';
            }
        };
    </script>

    <div class="top-bar">
        <div class="menu-btn" id="menuBtn" onclick="toggleMenu()">
            <span></span><span></span><span></span>
        </div>
        <div class="logo">🌟 LongPT</div>
        <div class="top-placeholder"></div>
    </div>

    <div class="overlay" id="overlay" onclick="toggleMenu()"></div>
    <div class="sidebar" id="sidebar">
        <a href="/?tab=official" class="menu-item {% if tab == 'official' %}active{% endif %}">官方发布</a>
        <a href="/?tab=recent" class="menu-item {% if tab == 'recent' %}active{% endif %}">最近发布</a>
        <a href="/?tab=account" class="menu-item {% if tab == 'account' %}active{% endif %}">我的帐号</a>
        <div class="theme-toggle-icon" id="themeIcon" onclick="cycleTheme()">🔄</div>
    </div>

    <div class="container" id="list-container">
        {% if tab in ['official', 'recent', 'search'] %}
        <form class="global-search-container" action="/" method="GET">
            <input type="hidden" name="tab" value="search">
            <input type="text" name="q" class="search-input" placeholder="输入关键词搜全站..." value="{{ query }}">
            <button type="submit" class="search-btn">🔍</button>
        </form>
        {% endif %}

        {% if error %}<div style="text-align:center; color:#ef4444; margin-top:30px; font-weight:bold;">{{ error }}</div>{% endif %}

        {% if tab == 'account' and user_info %}
        <div class="account-box">
            <div class="avatar">{{ user_info.username[:1] | upper }}</div>
            <div class="account-name">{{ user_info.username }}</div>
            <div class="stat-grid">
                <div><div class="stat-val">{{ user_info.bonus }}</div><div class="stat-lbl">魔力值</div></div>
                <div><div class="stat-val">{{ user_info.ratio }}</div><div class="stat-lbl">分享率</div></div>
                <div><div class="stat-val">{{ user_info.upload }}</div><div class="stat-lbl">总上传</div></div>
                <div><div class="stat-val">{{ user_info.download }}</div><div class="stat-lbl">总下载</div></div>
            </div>
            <button id="signinBtn" class="btn-signin" onclick="doSignIn()">✅ 每日签到</button>
            <button id="lotteryBtn" class="btn-lottery" onclick="doLottery()">🎰 立即参与抽奖</button>
        </div>
        <script>
            function doSignIn() {
                const btn = document.getElementById('signinBtn');
                btn.innerText = '打卡中...'; btn.disabled = true;
                fetch('/api/signin').then(r => r.json()).then(data => {
                    alert(data.msg); btn.innerText = '✅ 每日签到'; btn.disabled = false;
                }).catch(e => { alert('请求失败'); btn.innerText = '✅ 每日签到'; btn.disabled = false; });
            }
            function doLottery() {
                const btn = document.getElementById('lotteryBtn');
                btn.innerText = '参与中...'; btn.disabled = true;
                fetch('/api/lottery').then(r => r.json()).then(data => {
                    alert(data.msg); btn.innerText = '🎰 立即参与抽奖'; btn.disabled = false;
                }).catch(e => { alert('请求失败'); btn.innerText = '🎰 立即参与抽奖'; btn.disabled = false; });
            }
        </script>
        
        {% elif tab != 'account' %}
            {% for item in torrents %}
            <div class="list-item" id="item-{{ item.id }}">
                <div class="poster-wrapper">
                    {% set needs_lazy = not item.img or ('data:image/svg+xml' in item.img) %}
                    <img class="poster {% if needs_lazy %}lazy-load{% endif %}" 
                         src="{{ item.img if item.img else 'data:image/svg+xml;utf8,<svg xmlns=\\'http://www.w3.org/2000/svg\\' width=\\'100\\' height=\\'140\\'><rect width=\\'100%\\' height=\\'100%\\' fill=\\'%23e6e6e6\\'/><text x=\\'50%\\' y=\\'50%\\' fill=\\'%23999\\' font-size=\\'12\\' font-family=\\'sans-serif\\' text-anchor=\\'middle\\' dominant-baseline=\\'middle\\'>加载海报</text></svg>' }}" 
                         {% if needs_lazy %}data-tid="{{ item.id }}"{% endif %}
                         referrerpolicy="no-referrer" alt="海报" onclick="window.open('{{ item.link }}', '_blank')">
                    {% if 'FREE' in item.tags|join(' ')|upper %}
                        <div class="tag-free">FREE</div>
                    {% endif %}
                </div>
                <div class="info">
                    <div class="title" onclick="window.open('{{ item.link }}', '_blank')">{{ item.title }}</div>
                    <div class="subtitle">{{ item.subtitle }}</div>
                    
                    <div class="stats-and-download">
                        <div class="stats-row">
                            <span class="stat-item" id="size-{{ item.id }}">📄 {{ item.size }}</span>
                            <span class="stat-item"><span class="seed-icon">▲</span> {{ item.seeders }}</span>
                            <span class="stat-item"><span class="leech-icon">▼</span> {{ item.leechers }}</span>
                            {% if item.group %}
                                {% if item.group|upper == 'LONGWEB' %}
                                    <span class="badge-longweb">💎</span>
                                {% else %}
                                    <span class="badge-group">{{ item.group }}</span>
                                {% endif %}
                            {% endif %}
                        </div>
                        <a href="/download/{{ item.id }}" class="download-btn">下载</a>
                    </div>

                    <div class="tags-wrapper">
                        <div class="tags-top">
                            {% for tag in item.top_pri %}
                                <span class="badge-tech">{{ tag }}</span>
                            {% endfor %}
                            {% for tag in item.top_sec %}
                                <span class="badge-tech secondary-tag">{{ tag }}</span>
                            {% endfor %}
                        </div>

                        <div class="tags-bottom">
                            {% if item.time_tag %}
                                <span class="badge-discount">{{ item.time_tag }}</span>
                            {% endif %}
                            
                            {% if item.clock_tag %}
                                <span class="badge" style="background:#64748b;">{{ item.clock_tag }}</span>
                            {% endif %}
                            
                            {% for tag in item.bot_disc %}
                                <span class="badge-discount">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_pri %}
                                <span class="badge">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_sec %}
                                <span class="badge-lang secondary-tag">{{ tag }}</span>
                            {% endfor %}
                            
                            {% if item.top_sec|length > 0 or item.bot_sec|length > 0 %}
                                <span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>
                            {% endif %}
                        </div>
                    </div>
                </div>
            </div>
            {% endfor %}
            
            {% if tab == 'search' and not torrents and query %}
                <div style="text-align:center; color:var(--text-sub); margin-top:30px;">找到 0 个结果</div>
            {% elif tab == 'search' and torrents %}
                <div id="load-more-sentinel" style="text-align:center; padding: 20px; color:var(--text-sub); font-size:12px; clear:both;">
                    向下划动，无痕加载更多... ⚡
                </div>
            {% elif tab in ['recent', 'official'] and not error %}
                <div id="load-more-sentinel" style="text-align:center; padding: 20px; color:var(--text-sub); font-size:12px; clear:both;">
                    向下划动，无痕加载更多... ⚡
                </div>
            {% endif %}
        {% endif %}
    </div>

    <!-- 💡真正的 IntersectionObserver 滑动懒加载 -->
    <script>
        const sidebar = document.getElementById('sidebar');
        const overlay = document.getElementById('overlay');
        const menuBtn = document.getElementById('menuBtn');
        const themeIcon = document.getElementById('themeIcon');

        function toggleMenu() { 
            sidebar.classList.toggle('open'); 
            overlay.classList.toggle('open'); 
            menuBtn.classList.toggle('open');
        }

        const icons = { 'auto': '🔄', 'dark': '🌙', 'light': '☀️' };
        themeIcon.innerText = icons[currentTheme];

        function cycleTheme() {
            if (currentTheme === 'auto') currentTheme = 'dark';
            else if (currentTheme === 'dark') currentTheme = 'light';
            else currentTheme = 'auto';
            localStorage.setItem('longpt_theme', currentTheme);
            setTheme(currentTheme);
            themeIcon.innerText = icons[currentTheme];
        }

        window.lazyObserver = null;

        document.addEventListener("DOMContentLoaded", function() {
            if ('IntersectionObserver' in window) {
                window.lazyObserver = new IntersectionObserver((entries, obs) => {
                    entries.forEach(entry => {
                        if (entry.isIntersecting) {
                            const img = entry.target;
                            const tid = img.getAttribute('data-tid');
                            if (tid) {
                                fetch('/api/extra/' + tid)
                                    .then(r => r.json())
                                    .then(data => {
                                        if (data.img) {
                                            img.src = data.img;
                                        } else {
                                            img.src = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23f4f4f5'/><text x='50%' y='50%' fill='%23a1a1aa' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>暂无海报</text></svg>";
                                        }
                                        img.classList.remove('lazy-load');
                                        
                                        if (data.size && data.size !== '未知' && data.size !== '加载中...') {
                                            const sizeSpan = document.getElementById('size-' + tid);
                                            if (sizeSpan) sizeSpan.innerHTML = '📄 ' + data.size;
                                        }
                                        
                                        const itemDiv = document.getElementById('item-' + tid);
                                        if (itemDiv && data.tags) {
                                            if (data.group && !itemDiv.querySelector('.badge-group') && !itemDiv.querySelector('.badge-longweb')) {
                                                const statsRow = itemDiv.querySelector('.stats-row');
                                                if (statsRow) {
                                                    if (data.group.toUpperCase() === 'LONGWEB') {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-longweb">💎</span>`);
                                                    } else {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-group">${data.group}</span>`);
                                                    }
                                                }
                                            }
                                            
                                            const tagsWrapper = itemDiv.querySelector('.tags-wrapper');
                                            if (tagsWrapper && data.top_pri) {
                                                let hasFree = !!itemDiv.querySelector('.tag-free');
                                                if (data.tags.some(t => t.toUpperCase() === 'FREE' || t === '免费') && !hasFree) {
                                                    itemDiv.querySelector('.poster-wrapper').insertAdjacentHTML('beforeend', '<div class="tag-free">FREE</div>');
                                                }
                                                
                                                let html = '<div class="tags-top">';
                                                data.top_pri.forEach(t => html += `<span class="badge-tech">${t}</span>`);
                                                data.top_sec.forEach(t => html += `<span class="badge-tech secondary-tag">${t}</span>`);
                                                html += '</div><div class="tags-bottom">';
                                                
                                                if (data.time_tag) html += `<span class="badge-discount">${data.time_tag}</span>`;
                                                if (data.clock_tag) html += `<span class="badge" style="background:#64748b;">${data.clock_tag}</span>`;
                                                
                                                data.bot_disc.forEach(t => { 
                                                    if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge-discount">${t}</span>`; 
                                                });
                                                data.bot_pri.forEach(t => { 
                                                    if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge">${t}</span>`; 
                                                });
                                                data.bot_sec.forEach(t => html += `<span class="badge-lang secondary-tag">${t}</span>`);
                                                
                                                if (data.top_sec.length > 0 || data.bot_sec.length > 0) {
                                                    html += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                                }
                                                html += '</div>';
                                                tagsWrapper.innerHTML = html;
                                            }
                                        }
                                    })
                                    .catch(e => console.error('懒加载失败:', e));
                                    
                                img.removeAttribute('data-tid');
                                obs.unobserve(img);
                            }
                        }
                    });
                }, { rootMargin: "300px" });

                document.querySelectorAll('.lazy-load').forEach(img => {
                    window.lazyObserver.observe(img);
                });

                // 💡无痕无限滑动监听器逻辑
                const sentinel = document.getElementById('load-more-sentinel');
                if (sentinel) {
                    let currentPage = 1; 
                    let isLoadingMore = false;
                    let hasMore = true;
                    // 初始化去重缓存
                    const seenIds = new Set();
                    document.querySelectorAll('.list-item').forEach(el => seenIds.add(el.id.replace('item-', '')));

                    const loadMoreObserver = new IntersectionObserver((entries) => {
                        const entry = entries[0];
                        if (entry.isIntersecting && !isLoadingMore && hasMore) {
                            loadMore();
                        }
                    }, { rootMargin: "200px" });
                    
                    loadMoreObserver.observe(sentinel);

                    function loadMore() {
                        isLoadingMore = true;
                        sentinel.innerText = "正在无痕加载下一页...";
                        
                        const params = new URLSearchParams(window.location.search);
                        const tab = params.get('tab') || 'official';
                        const q = params.get('q') || '';
                        
                        fetch(`/api/load_more?tab=${tab}&q=${encodeURIComponent(q)}&page=${currentPage}`)
                            .then(r => r.json())
                            .then(data => {
                                currentPage++;
                                isLoadingMore = false;
                                
                                if (data.error) {
                                    sentinel.innerText = "加载失败: " + data.error + " (点击重试)";
                                    sentinel.onclick = loadMore;
                                    return;
                                }
                                
                                if (data.torrents.length === 0) {
                                    if (currentPage < 10) {
                                        loadMore();
                                    } else {
                                        hasMore = false;
                                        sentinel.innerText = "缓存已探底，没有更多数据了";
                                    }
                                    return;
                                }
                                
                                let addedCount = 0;
                                data.torrents.forEach(item => {
                                    if (seenIds.has(item.id)) return;
                                    seenIds.add(item.id);
                                    addedCount++;
                                    
                                    let tagsTopHtml = '';
                                    item.top_pri.forEach(t => tagsTopHtml += `<span class="badge-tech">${t}</span>`);
                                    item.top_sec.forEach(t => tagsTopHtml += `<span class="badge-tech secondary-tag">${t}</span>`);
                                    
                                    let tagsBotHtml = '';
                                    if (item.time_tag) tagsBotHtml += `<span class="badge-discount">${item.time_tag}</span>`;
                                    if (item.clock_tag) tagsBotHtml += `<span class="badge" style="background:#64748b;">${item.clock_tag}</span>`;
                                    item.bot_disc.forEach(t => tagsBotHtml += `<span class="badge-discount">${t}</span>`);
                                    item.bot_pri.forEach(t => tagsBotHtml += `<span class="badge">${t}</span>`);
                                    item.bot_sec.forEach(t => tagsBotHtml += `<span class="badge-lang secondary-tag">${t}</span>`);
                                    
                                    if (item.top_sec.length > 0 || item.bot_sec.length > 0) {
                                        tagsBotHtml += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                    }

                                    let groupHtml = '';
                                    if (item.group) {
                                        if (item.group.toUpperCase() === 'LONGWEB') {
                                            groupHtml = `<span class="badge-longweb">💎</span>`;
                                        } else {
                                            groupHtml = `<span class="badge-group">${item.group}</span>`;
                                        }
                                    }
                                    
                                    let freeTagHtml = '';
                                    if (item.tags.join(' ').toUpperCase().includes('FREE')) {
                                        freeTagHtml = `<div class="tag-free">FREE</div>`;
                                    }

                                    const imgPlaceholder = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23e6e6e6'/><text x='50%' y='50%' fill='%23999' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>加载海报</text></svg>";
                                    const imgSrc = item.img ? item.img : imgPlaceholder;
                                    const needsLazy = !item.img || item.img.includes('data:image/svg+xml');

                                    const html = `
                                    <div class="list-item" id="item-${item.id}">
                                        <div class="poster-wrapper">
                                            <img class="poster ${needsLazy ? 'lazy-load' : ''}" 
                                                 src="${imgSrc}" 
                                                 ${needsLazy ? `data-tid="${item.id}"` : ''}
                                                 referrerpolicy="no-referrer" alt="海报" onclick="window.open('${item.link}', '_blank')">
                                            ${freeTagHtml}
                                        </div>
                                        <div class="info">
                                            <div class="title" onclick="window.open('${item.link}', '_blank')">${item.title}</div>
                                            <div class="subtitle">${item.subtitle || ''}</div>
                                            <div class="stats-and-download">
                                                <div class="stats-row">
                                                    <span class="stat-item" id="size-${item.id}">📄 ${item.size}</span>
                                                    <span class="stat-item"><span class="seed-icon">▲</span> ${item.seeders}</span>
                                                    <span class="stat-item"><span class="leech-icon">▼</span> ${item.leechers}</span>
                                                    ${groupHtml}
                                                </div>
                                                <a href="/download/${item.id}" class="download-btn">下载</a>
                                            </div>
                                            <div class="tags-wrapper">
                                                <div class="tags-top">${tagsTopHtml}</div>
                                                <div class="tags-bottom">${tagsBotHtml}</div>
                                            </div>
                                        </div>
                                    </div>
                                    `;
                                    sentinel.insertAdjacentHTML('beforebegin', html);
                                });
                                
                                document.querySelectorAll('.lazy-load:not([data-observed])').forEach(img => {
                                    img.setAttribute('data-observed', '1');
                                    window.lazyObserver.observe(img);
                                });
                                
                                if (addedCount === 0) {
                                    loadMore(); 
                                } else {
                                    sentinel.innerText = "向下划动，无痕加载更多... ⚡";
                                    sentinel.onclick = null;
                                }
                            })
                            .catch(e => {
                                isLoadingMore = false;
                                sentinel.innerText = "网络错误，点击重试";
                                sentinel.onclick = loadMore;
                            });
                    }
                }
            }
        });
    </script>
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
            resp = requests.get(INDEX_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效，请在代码中更新 Cookie！"
            else:
                soup = BeautifulSoup(resp.text, 'html.parser')
                for tag in soup.find_all(True):
                    title_attr = tag.get('title') or tag.get('alt') or tag.get('data-original-title')
                    if title_attr and isinstance(title_attr, str) and len(title_attr) < 20:
                        tag.append(f"{title_attr}")
                page_text = soup.get_text(separator='', strip=True)
                
                user_match = re.search(r'欢迎回来[\s,，]*([A-Za-z0-9_\-]+)', page_text)
                bonus_match = re.search(r'魔力值[^\d]*([\d\.,]+)', page_text)
                ratio_match = re.search(r'分享率[^\d]*([\d\.,]+|无限|Inf)', page_text, re.IGNORECASE)
                up_match = re.search(r'(?:上传量?|上传|上行)[^\d]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                down_match = re.search(r'(?:下载量?|下载|下行)[^\d]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                
                user_info = {
                    'username': user_match.group(1).strip() if user_match else "LongPT会员",
                    'bonus': bonus_match.group(1) if bonus_match else "0",
                    'ratio': ratio_match.group(1) if ratio_match else "0",
                    'upload': up_match.group(1) if up_match else "0 GB",
                    'download': down_match.group(1) if down_match else "0 GB"
                }

        elif tab in ['official', 'search']:
            if tab == 'search' and not query: pass
            else:
                if tab == 'search':
                    safe_query = urllib.parse.quote(query)
                    target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
                else: 
                    target_url = OFFICIAL_URL
                    
                fetched_torrents, err = parse_torrents_page(target_url, seen_ids, is_search=(tab == 'search'), skip_sticky=False)
                if fetched_torrents is not None:
                    history_dict = load_cache('official')
                    for item in fetched_torrents:
                        tid_str = str(item['id'])
                        if tid_str in history_dict:
                            cached = history_dict[tid_str]
                            if 'data:image' not in cached.get('img','') and 'data:image' in item.get('img',''):
                                item['img'] = cached['img']
                            if cached.get('size') not in ['未知', '加载中...'] and item.get('size') in ['未知', '-']:
                                item['size'] = cached['size']
                            if not item.get('group') and cached.get('group'):
                                item['group'] = cached['group']
                            item['tags'] = purify_tags(merge_tags_list(item.get('tags', []), cached.get('tags', [])))
                        
                        history_dict[tid_str] = item
                    if tab != 'search':
                        save_cache(history_dict, 'official')
                    torrents = fetched_torrents
                else:
                    if err: error_msg = err

        elif tab == 'recent':
            fetched_torrents, err_msg = parse_index_page(seen_ids)
            recent_dict = load_cache('recent')
            
            if fetched_torrents is not None:
                for item in fetched_torrents:
                    tid_str = str(item['id'])
                    if tid_str in recent_dict:
                        cached = recent_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') and cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                        # 💡 强制护盾：不论最近发布怎么刷新，都不会擦除已经抓到的时间标签
                        item['tags'] = purify_tags(merge_tags_details(item.get('tags', []), cached.get('tags', [])))
                    recent_dict[tid_str] = item
                save_cache(recent_dict, 'recent')
                
                merged = list(recent_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = merged[:50]
            else:
                history_data = list(recent_dict.values())
                history_data.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = history_data[:50]
                if err_msg: error_msg = err_msg

        if torrents:
            for item in torrents:
                tpri, tsec, bdisc, bpri, bsec = sort_tags(item.get('tags', []))
                item['top_pri'] = tpri
                item['top_sec'] = tsec
                item['bot_disc'] = bdisc
                item['bot_pri'] = bpri
                item['bot_sec'] = bsec
                item['time_tag'] = ""
                item['clock_tag'] = ""
                
                for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                    for t in list(arr):
                        if '剩余' in t.upper():
                            item['time_tag'] = t
                            arr.remove(t)
                        elif '🕒' in t:
                            item['clock_tag'] = t
                            arr.remove(t)

    except Exception as e:
        error_msg = f"服务器内部请求错误: {e}"
        torrents = torrents or []
        
    return render_template_string(HTML_TEMPLATE, tab=tab, query=query, torrents=torrents or [], user_info=user_info, error=error_msg)

@app.route('/api/load_more')
def api_load_more():
    tab = request.args.get('tab', 'official')
    query = request.args.get('q', '')
    try: page = int(request.args.get('page', 1))
    except: page = 1
        
    seen_ids = set()
    err = None
    fetched = []
    
    if tab == 'search':
        safe_query = urllib.parse.quote(query)
        target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0&page={page}"
        fetched, err = parse_torrents_page(target_url, seen_ids, skip_sticky=False, start_page=page, max_pages=1)
    elif tab == 'official':
        target_url = f"{OFFICIAL_URL}?page={page}" if "?" not in OFFICIAL_URL else f"{OFFICIAL_URL}&page={page}"
        fetched, err = parse_torrents_page(target_url, seen_ids, skip_sticky=False, start_page=page, max_pages=1)
    else: 
        rec_dict = load_cache('recent')
        merged = list(rec_dict.values())
        merged.sort(key=lambda x: int(x['id']), reverse=True)
        start_idx = page * 50
        fetched = merged[start_idx : start_idx + 50]
        
    if fetched:
        for item in fetched:
            tpri, tsec, bdisc, bpri, bsec = sort_tags(item.get('tags', []))
            item['top_pri'] = tpri
            item['top_sec'] = tsec
            item['bot_disc'] = bdisc
            item['bot_pri'] = bpri
            item['bot_sec'] = bsec
            item['time_tag'] = ""
            item['clock_tag'] = ""
            
            for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                for t in list(arr):
                    if '剩余' in t.upper():
                        item['time_tag'] = t
                        arr.remove(t)
                    elif '🕒' in t:
                        item['clock_tag'] = t
                        arr.remove(t)
                        
    return jsonify({"torrents": fetched or [], "error": err})

@app.route('/api/extra/<int:tid>')
def api_extra(tid):
    img_src, size, extra_tags, release_group = fetch_details_info(tid)
    merged_tags = purify_tags(extra_tags)
    
    tid_str = str(tid)
    for c_type in ['recent', 'official']:
        history_dict = load_cache(c_type)
        if tid_str in history_dict:
            cached_item = history_dict[tid_str]
            final_tags = purify_tags(merge_tags_details(merged_tags, cached_item.get('tags', [])))
            cached_item['tags'] = final_tags
            merged_tags = final_tags
            
            if img_src: cached_item['img'] = img_src
            if size and size != "未知": cached_item['size'] = size
            if release_group and not cached_item.get('group'): cached_item['group'] = release_group
            
            history_dict[tid_str] = cached_item
            save_cache(history_dict, c_type)
        
    tpri, tsec, bdisc, bpri, bsec = sort_tags(merged_tags)
    time_tag = ""
    clock_tag = ""
    for arr in [tpri, tsec, bdisc, bpri, bsec]:
        for t in list(arr):
            if '剩余' in t.upper():
                time_tag = t
                arr.remove(t)
            elif '🕒' in t:
                clock_tag = t
                arr.remove(t)
    
    return jsonify({
        "img": img_src, 
        "size": size, 
        "tags": merged_tags,
        "top_pri": tpri,
        "top_sec": tsec,
        "bot_disc": bdisc,
        "bot_pri": bpri,
        "bot_sec": bsec,
        "time_tag": time_tag,
        "clock_tag": clock_tag,
        "group": release_group
    })

@app.route('/api/lottery')
def api_lottery():
    try:
        target_api = f"{BASE_URL}/pt-api/v1/lucky/pluginsPrizeReceiptRecord/join"
        r = requests.get(target_api, headers=HEADERS, proxies=PROXIES, timeout=5)
        r.encoding = 'utf-8'
        
        if r.status_code == 200:
            try:
                data = r.json()
                msg_text = data.get('msg') or data.get('message') or str(data)
                clean_msg = re.sub(r'<[^>]+>', ' ', msg_text).replace('\n', '').strip()
                return jsonify({"msg": f"🎰 抽奖真实反馈：\n{clean_msg}"})
            except Exception:
                clean_txt = re.sub(r'<[^>]+>', ' ', r.text).replace('\n', '').strip()
                match = re.search(r'(恭喜.*?|获得.*?|已参与.*?|积分不足.*?|不够.*?|成功.*)', clean_txt)
                if match:
                    return jsonify({"msg": f"🎰 抽奖反馈：\n{match.group(1)[:100]}"})
                else:
                    return jsonify({"msg": f"🎰 已发送，服务器返回：\n{clean_txt[:60]}..."})
        else:
            return jsonify({"msg": f"❌ 抽奖接口报错，状态码: {r.status_code}"})
    except Exception as e:
        return jsonify({"msg": f"❌ 抽奖请求异常: {str(e)}"})

@app.route('/api/signin')
def api_signin():
    try:
        r = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        r.encoding = 'utf-8'
        if r.status_code == 200:
            if any(kw in r.text for kw in ["已经", "退下", "已签到", "请明天再来"]):
                return jsonify({"msg": "☕ 今天已经签到过了，明天再来吧。"})
            elif any(kw in r.text for kw in ["这是您的第", "签到成功", "魔力值"]):
                match = re.search(r'(这是您的第.*?魔力值|签到成功.*?魔力值)', BeautifulSoup(r.text, 'html.parser').get_text(separator=' ', strip=True))
                msg = match.group(1).replace('  ', ' ') if match else "签到成功！"
                return jsonify({"msg": f"🎉 {msg}"})
            elif "登录" in r.text:
                return jsonify({"msg": "⚠️ 签到失败：Cookie 已失效，请更新！"})
            else:
                return jsonify({"msg": "✅ 签到请求已发送，请留意积分变化。"})
        else:
            return jsonify({"msg": f"❌ 签到接口报错，状态码: {r.status_code}"})
    except Exception as e:
        return jsonify({"msg": f"❌ 网络请求出错: {str(e)}"})

@app.route('/download/<int:tid>')
def download_torrent(tid):
    try:
        r = requests.get(f"{BASE_URL}/download.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        if "application/x-bittorrent" in r.headers.get('Content-Type', '') or "octet-stream" in r.headers.get('Content-Type', ''):
            return Response(r.content, mimetype="application/x-bittorrent", headers={"Content-Disposition": f"attachment;filename=LongPT_{tid}.torrent"})
        else: return "下载失败，Cookie 可能失效或无权下载该种子。"
    except Exception as e: return f"请求下载出错: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5051)
```

7.h5school.py
```
from flask import Flask, render_template_string, Response, request, jsonify
import requests
from bs4 import BeautifulSoup
import urllib.parse
import re
import os
import json
import threading
import time
import datetime

app = Flask(__name__)

# ================= 必须配置的区域 =================
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    # 【安全提醒】：请把你的 BTSCHOOL Cookie 填在下面
    'Cookie': 'c_secure_uid=MTg4NTY4; c_secure_pass=bdcd9db1dee67203b3ff5cc2dca1db3c; c_secure_ssl=eWVhaA%3D%3D; c_secure_tracker_ssl=eWVhaA%3D%3D; c_secure_login=bm9wZQ%3D%3D; sl-session=mtXjA0OowGrmPpC8iv38uA==' 
}

PROXY_URL = ""
PROXIES = {"http": PROXY_URL, "https": PROXY_URL} if PROXY_URL else None
TIMEOUT = 30

BASE_URL = "https://pt.btschool.club"
OFFICIAL_URL = f"{BASE_URL}/torrents.php"
INDEX_URL = f"{BASE_URL}/index.php"
SIGNIN_URL = f"{BASE_URL}/index.php?action=addbonus"
HISTORY_FILE = "btschool_history.json"
# =================================================

MEM_CACHE = {}
CACHE_LOCK = threading.Lock()

def load_history():
    global MEM_CACHE
    with CACHE_LOCK:
        if not MEM_CACHE:
            if os.path.exists(HISTORY_FILE):
                try:
                    with open(HISTORY_FILE, 'r', encoding='utf-8') as f:
                        data = json.load(f)
                        MEM_CACHE = {str(item['id']): item for item in data}
                except Exception: pass
    return MEM_CACHE

def save_history(history_dict):
    global MEM_CACHE
    with CACHE_LOCK:
        MEM_CACHE = history_dict.copy()
        merged = list(MEM_CACHE.values())
        merged.sort(key=lambda x: int(x['id']), reverse=True)
        try:
            with open(HISTORY_FILE, 'w', encoding='utf-8') as f:
                json.dump(merged[:500], f, ensure_ascii=False, indent=2)
        except Exception: pass

TECH_KEYWORDS = [
    'FREE', '50%', '30%', '2X', '热门', '官方', '中字', '英字', '国语', '粤语', '英语', 
    '简繁', '双语', '杜比', '高帧', '高码', '臻彩MAX', '高分', '完结', '4K', '1080P', 
    '2160P', '720P', 'WEB-DL', 'BLU-RAY', 'REMUX', 'HDR10+', 'HDR10', 'HDRVIVID', 
    'HDR', 'DOVI', 'DV', 'SDR', '60FPS', 'HFR', 'HQ', 'H.265', 'H.264', 'H265', 'H264', 
    'HEVC', 'AAC', 'DDP', 'ATMOS', '10BIT', '首发', '禁转', '应求', 'DIY', '字幕'
]

def purify_tags(tags):
    valid = []
    for t in tags:
        clean_t = t.upper()
        if '剩余' in clean_t or '🕒' in clean_t:
            valid.append(t)
        elif clean_t in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', clean_t):
            valid.append(t)
            
    final_tags = []
    time_tags = [t for t in valid if '剩余' in t]
    best_time = max(time_tags, key=len) if time_tags else None
    
    clock_tags = [t for t in valid if '🕒' in t]
    best_clock = clock_tags[-1] if clock_tags else None
    
    if best_time and best_clock:
        pure_time = re.sub(r'^.*?(?:剩余时间|免费剩余)[:：\s]*', '', best_time).strip()
        pure_clock = best_clock.replace('🕒', '').strip()
        if pure_time == pure_clock:
            best_clock = None
            
    for t in valid:
        if '剩余' in t:
            if t == best_time and best_time not in final_tags:
                if not re.match(r'^剩余时间[:：\s]*\d+\s*分$', t):
                    final_tags.append(t)
        elif '🕒' in t:
            if t == best_clock and best_clock not in final_tags:
                final_tags.append(t)
        elif t not in final_tags:
            final_tags.append(t)
            
    return final_tags

def merge_tags_from_list(list_tags, cached_tags):
    res = list(list_tags)
    for t in cached_tags:
        t_up = t.upper()
        is_dynamic = '%' in t_up or '2X' in t_up or t_up in ['FREE', '免费'] or '剩余' in t_up
        if not is_dynamic and t not in res:
            res.append(t)
    return res

def merge_tags_from_details(details_tags, cached_tags):
    res = list(details_tags)
    has_discount = any('%' in x.upper() or '2X' in x.upper() for x in res)
    has_free = any(x.upper() in ['FREE', '免费'] for x in res)
    has_time = any('剩余' in x for x in res)
    has_clock = any('🕒' in x for x in res)
    
    for t in cached_tags:
        if t in res: continue
        t_up = t.upper()
        if '%' in t_up or '2X' in t_up:
            if not has_discount: res.append(t)
        elif t_up in ['FREE', '免费']:
            if not has_free: res.append(t)
        elif '剩余' in t_up:
            if not has_time: res.append(t)
        elif '🕒' in t_up:
            if not has_clock: res.append(t)
        else:
            res.append(t)
    return res

def auto_extract_tags(text, current_tags):
    en_matches = re.findall(r'(?i)\b(4K|1080P|2160P|720P|WEB-DL|BLU-RAY|REMUX|HDR10\+|HDR10|HDRVIVID|HDR|DOVI|DV|SDR|60FPS|HFR|HQ|H\.?265|H\.?264|HEVC|AAC|DDP|ATMOS|10BIT)\b', text)
    cn_matches = re.findall(r'(杜比|高帧|高码|臻彩MAX|高分|完结|国语|中字|英字|粤语|英语|简繁|双语|官方|首发)', text)
    disc_matches = re.findall(r'(50%|30%|2X)', text.upper())
    
    all_new = [m.upper() for m in en_matches] + cn_matches + disc_matches
    for tag in all_new:
        tag_up = tag.upper()
        if not any(tag_up in t.upper() or t.upper() in tag_up for t in current_tags):
            current_tags.append(tag)
    return current_tags

def sort_tags(tags):
    top_pri, top_sec, bot_disc, bot_pri, bot_sec = [], [], [], [], []
    remaining = list(tags)
    
    t_res = ['4K', '2160P', '1080P', '720P']
    t_hdr = ['HDRVIVID', 'HDR10+', 'HDR10', 'HDR', 'DOVI', 'DV', 'SDR']
    t_fps = ['60FPS', 'HFR', '高帧', '臻彩MAX', 'BLU-RAY', 'REMUX']
    t_enc = ['WEB-DL', 'H.265', 'H.264', 'H265', 'H264', 'HEVC', 'AAC', 'DDP', 'ATMOS', '杜比', '10BIT', 'HQ', '高码']
    
    for t in list(remaining):
        t_upper = t.upper()
        if '%' in t_upper or '2X' in t_upper or t_upper in ['免费', 'FREE', '30%', '50%']:
            if t_upper not in ['FREE', '免费']:
                bot_disc.append(t)
            remaining.remove(t)
    
    t_status = ['热门', '完结', '首发', '高分', 'DIY']
    t_lang = ['国语', '中字', '英字', '粤语', '英语', '简繁', '双语', '官方', '字幕']
    
    for kw in t_res + t_hdr + t_fps:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_pri.append(t)
                remaining.remove(t)
                
    for kw in t_enc:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_sec.append(t)
                remaining.remove(t)
                
    for kw in t_status:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_pri.append(t)
                remaining.remove(t)
                
    for kw in t_lang:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_sec.append(t)
                remaining.remove(t)
                
    for t in list(remaining):
        bot_pri.append(t)
        
    return top_pri, top_sec, bot_disc, bot_pri, bot_sec

def fetch_details_info(tid):
    try:
        r = requests.get(f"{BASE_URL}/details.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        
        raw_html = r.text
        soup = BeautifulSoup(raw_html, 'html.parser')
        extra_tags = []

        text_clean = soup.get_text(separator=' ')
        tm2 = re.search(r'(?:免费剩余时间|免费剩余|优惠剩余时间|剩余时间|还有|剩余)[:：\s]+(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', text_clean)
        if tm2: 
            extra_tags.append(f"剩余时间: {tm2.group(1).strip()}")
        else:
            time_match = re.search(r'(?:免费剩余|剩余时间|剩余)[^\d<]{0,30}(?:<[^>]+>)*\s*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', raw_html)
            if time_match: extra_tags.append(f"剩余时间: {time_match.group(1).strip()}")

        img_src = ""
        descr_div = soup.find('div', id='kdescr')
        if descr_div:
            img = descr_div.find('img')
            if img: img_src = img.get('data-src') or img.get('src') or ""
        if not img_src:
            for img in soup.find_all('img'):
                src = img.get('data-src') or img.get('src') or ""
                if src and not any(x in src.lower() for x in ['category', 'smilies', 'icon', 'logo', 'avatar', 'rating', 'button', 'rule']):
                    img_src = src
                    break
        if img_src and not img_src.startswith('http'):
            img_src = f"{BASE_URL}/" + img_src

        size = "未知"
        size_match = re.search(r'大小.*?([\d\.]+\s*[KMGTP]B)', raw_html, re.IGNORECASE)
        if size_match: size = size_match.group(1)

        h1 = soup.find('h1')
        h1_text = ""
        release_group = ""
        
        if h1:
            h1_text = h1.get_text(strip=True)
            for kw in ['免费', 'FREE', '50%', '30%', '2X']:
                if kw in h1_text.upper():
                    if kw in ['免费', 'FREE']:
                        if 'Free' not in extra_tags: extra_tags.append('Free')
                    else:
                        if kw not in extra_tags: extra_tags.append(kw)
            
            for img in h1.find_all('img'):
                alt = str(img.get('alt') or img.get('title') or '').replace('[', '').replace(']', '').strip()
                if alt.upper() in ['FREE', '免费']:
                    if 'Free' not in extra_tags: extra_tags.append('Free')
                elif any(x in alt for x in ['50%', '2X', '30%', '热门']):
                    if alt not in extra_tags: extra_tags.append(alt)
            
            for span in h1.find_all('span', class_=re.compile(r'tag|label|badge', re.I)):
                text = span.get_text(strip=True).replace('[', '').replace(']', '').strip()
                if text.upper() in ['FREE', '免费']:
                    if 'Free' not in extra_tags: extra_tags.append('Free')
                elif text and text not in extra_tags and len(text) <= 15:
                    extra_tags.append(text)
                    
            group_match = re.search(r'-([A-Za-z0-9@_]+)$', h1_text)
            if group_match:
                release_group = group_match.group(1).upper()

        sub_td = soup.find('td', class_='rowhead', string=re.compile('副标题|副標題'))
        sub_text = ""
        if sub_td:
            sub_text = sub_td.find_next_sibling('td').get_text(strip=True)
            
        extra_tags = auto_extract_tags(h1_text + " " + sub_text, extra_tags)
        
        return img_src, size, purify_tags(extra_tags), release_group
    except Exception as e:
        return "", "未知", [], ""

def parse_torrents_page(target_url_base, seen_ids, is_search=False, skip_sticky=False, start_page=0, max_pages=1):
    torrents = []
    try:
        for page in range(start_page, start_page + max_pages):
            url = f"{target_url_base}&page={page}" if "?" in target_url_base else f"{target_url_base}?page={page}"
            resp = requests.get(url, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            if "登录" in resp.text and "torrents.php" not in resp.url: return None, "⚠️ Cookie 已失效！请更新。"
            
            if resp.status_code == 200:
                soup = BeautifulSoup(resp.text, 'html.parser')
                for row in soup.find_all('tr'):
                    if skip_sticky:
                        sticky_img = row.find('img', class_=re.compile(r'sticky', re.I)) or \
                                     row.find('img', alt=re.compile(r'Sticky|置顶', re.I)) or \
                                     row.find('img', title=re.compile(r'Sticky|置顶', re.I)) or \
                                     row.find('img', src=re.compile(r'sticky', re.I))
                        if sticky_img: continue
                        
                        row_text = row.get_text()
                        if '[置顶]' in row_text or 'Sticky' in row_text:
                            continue

                    tds = row.find_all('td')
                    if len(tds) < 6: continue
                    title_a = row.find('a', href=lambda h: h and h.startswith('details.php?id='))
                    if not title_a: continue
                    
                    tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                    title = title_a.get('title') or title_a.get_text(strip=True)
                    if not title and title_a.find('b'): title = title_a.find('b').get_text(strip=True)
                    if not title or tid in seen_ids: continue
                    seen_ids.add(tid)
                    link = f"{BASE_URL}/" + title_a['href']
                    
                    release_group = ""
                    group_match = re.search(r'-([A-Za-z0-9@_]+)$', title.strip())
                    if group_match:
                        release_group = group_match.group(1).upper()
                    
                    try: 
                        size = tds[-5].get_text(strip=True)
                        seeders = tds[-4].get_text(strip=True)
                        leechers = tds[-3].get_text(strip=True)
                        time_col_text = tds[-6].get_text(separator='', strip=True)
                    except: 
                        size, seeders, leechers, time_col_text = "未知", "-", "-", ""

                    img_src = ""
                    td = title_a.find_parent('td')
                    tags = []
                    
                    if time_col_text and any(x in time_col_text for x in ['年', '月', '周', '天', '时', '分', '秒']):
                        time_val = re.sub(r'\s+', '', time_col_text)
                        tags.append(f"🕒 {time_val}")
                    
                    subtitle = ""
                    
                    if td:
                        for icon in td.find_all('img'):
                            alt = str(icon.get('alt') or '')
                            title_attr = str(icon.get('title') or '')
                            combined_text = alt + " " + title_attr
                            
                            if '免费' in combined_text or 'Free' in combined_text:
                                if 'Free' not in tags: tags.append('Free')
                                
                            tm = re.search(r'(?:免费剩余|剩余时间|剩余)[:：\s]*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', combined_text)
                            if tm:
                                t_str = f"剩余时间: {tm.group(1).strip()}"
                                if t_str not in tags: tags.append(t_str)

                            if icon.get('src', '') != img_src: 
                                icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                        
                        for span in td.find_all('span'):
                            text = span.get_text(strip=True).replace('[', '').replace(']', '').strip()
                            if not text: continue
                            if text.upper() in ['FREE', '免费']:
                                if 'Free' not in tags: tags.append('Free')
                                span.replace_with(soup.new_tag("span", string=f" [FREE] "))
                                continue
                                
                            span_class = ' '.join(span.get('class', [])).lower()
                            is_valid_class = bool(re.search(r'tag|label|badge', span_class))
                            is_known_kw = bool(text.upper() in [k.upper() for k in TECH_KEYWORDS])
                            
                            if (is_valid_class or is_known_kw) and text.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', text):
                                if text not in tags: tags.append(text)
                                span.replace_with(soup.new_tag("span", string=f" [{text}] "))
                        
                        subtitle = td.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                        subtitle = re.sub(r'(?i)\[?(?:Sticky|置顶|poster)\]?[\s\|]*', '', subtitle)
                        subtitle = re.sub(r'(?i)^\s*\[?新\]?\s*\|\s*', '', subtitle)
                        
                        inline_time = re.search(r'\[?(?:剩余时间)[:：\s]*\]?[\s\|]*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', subtitle)
                        if inline_time:
                            t_str = f"剩余时间: {inline_time.group(1).strip()}"
                            if t_str not in tags: tags.append(t_str)
                            subtitle = subtitle.replace(inline_time.group(0), '')
                        
                        clean_sub = subtitle
                        for m in re.finditer(r'\[(.*?)\]', subtitle):
                            tag = m.group(1).strip(' |[]')
                            if tag.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', tag) or tag.upper() in ['热门']:
                                if tag not in tags: tags.append(tag)
                                clean_sub = clean_sub.replace(m.group(0), '')
                            else:
                                clean_sub = clean_sub.replace(m.group(0), tag)
                        
                        clean_sub = re.sub(r'热门|免费', '', clean_sub)
                        
                        parts = [p.strip() for p in clean_sub.split('|') if p.strip()]
                        kept_sub = []
                        for p in parts:
                            clean_p = re.sub(r'^[()\[\]\-|\s]+|[()\[\]\-|\s]+$', '', p)
                            if not clean_p: continue
                            if clean_p.upper() in [k.upper() for k in TECH_KEYWORDS]:
                                if clean_p not in tags: tags.append(clean_p)
                            else:
                                kept_sub.append(p)
                        subtitle = ' | '.join(kept_sub)
                    
                    tags = auto_extract_tags(title + " " + subtitle, tags)
                    tags = purify_tags(tags)

                    torrents.append({
                        'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                        'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags, 'group': release_group
                    })
                if len(torrents) >= 50 and max_pages > 1: break
            if len(torrents) >= 50 and max_pages > 1: break
        return torrents, None
    except Exception as e:
        return None, f"网络请求异常: {str(e)}"

def background_monitor():
    last_sign_date = None
    while True:
        try:
            current_date = datetime.datetime.now().strftime("%Y-%m-%d")
            if last_sign_date != current_date:
                r = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
                r.encoding = 'utf-8'
                res_text = r.text
                if any(kw in res_text for kw in ["已经签到", "请明天再来", "签到成功", "魔力值", "次签到", "退下"]):
                    last_sign_date = current_date
        except Exception: pass

        try:
            current_fetched, err = parse_torrents_page(OFFICIAL_URL, set(), skip_sticky=True)
            if current_fetched:
                history_dict = load_history()
                
                for item in current_fetched[:5]:
                    tid_str = str(item['id'])
                    if tid_str not in history_dict or not history_dict[tid_str].get('img') or 'data:image/svg+xml' in history_dict[tid_str]['img'] or history_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags, det_group = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_group and not item.get('group'): item['group'] = det_group
                        if det_tags:
                            item['tags'] = purify_tags(merge_tags_from_details(det_tags, item.get('tags', [])))

                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if not item.get('group') and cached.get('group'): item['group'] = cached['group']
                        item['tags'] = purify_tags(merge_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                        
                    history_dict[tid_str] = item

                save_history(history_dict)
        except Exception as e: pass
        time.sleep(900)

threading.Thread(target=background_monitor, daemon=True).start()

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>BTSCHOOL 手机端</title>
    <style>
        :root { --bg-color: #f0f4f8; --card-bg: #ffffff; --text-main: #333333; --text-sub: #666666; --border-color: #e2e8f0; --topbar-bg: #ffffff; --icon-bg: #e0f2fe; --primary-color: #1976d2; }
        [data-theme="dark"] { --bg-color: #0f172a; --card-bg: #1e293b; --text-main: #f1f5f9; --text-sub: #94a3b8; --border-color: #334155; --topbar-bg: #1e293b; --icon-bg: #334155; --primary-color: #3b82f6; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background-color: var(--bg-color); color: var(--text-main); margin: 0; padding: 0; padding-top: 55px; transition: background-color 0.3s; }
        
        .top-bar { position: fixed; top: 0; left: 0; right: 0; height: 50px; background: var(--topbar-bg); display: flex; align-items: center; justify-content: space-between; padding: 0 15px; box-shadow: 0 1px 3px rgba(0,0,0,0.05); z-index: 100; transition: background 0.3s; }
        .menu-btn { width: 22px; height: 16px; position: relative; cursor: pointer; display: flex; flex-direction: column; justify-content: space-between; padding: 0; }
        .menu-btn span { display: block; height: 2px; width: 100%; background: var(--text-main); border-radius: 2px; transition: all 0.3s ease; }
        .menu-btn.open span:nth-child(1) { transform: translateY(7px) rotate(45deg); }
        .menu-btn.open span:nth-child(2) { opacity: 0; }
        .menu-btn.open span:nth-child(3) { transform: translateY(-7px) rotate(-45deg); }

        .logo { font-weight: 800; font-size: 20px; color: var(--primary-color); letter-spacing: 1px; display: flex; align-items: center; gap: 4px;}
        .logo-icon { font-size: 22px; }
        .top-placeholder { width: 22px; }

        .sidebar { position: fixed; top: 0; left: -260px; bottom: 0; width: 260px; background: var(--card-bg); z-index: 1001; transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: 2px 0 10px rgba(0,0,0,0.1); padding-top: 60px; display: flex; flex-direction: column; }
        .sidebar.open { left: 0; }
        .menu-item { padding: 16px 24px; font-size: 16px; font-weight: 500; color: var(--text-main); text-decoration: none; display: block; border-bottom: 1px solid var(--border-color); }
        .menu-item.active { color: var(--primary-color); border-left: 4px solid var(--primary-color); padding-left: 20px; background: rgba(25,118,210,0.05); }
        .theme-toggle-icon { margin: auto auto 30px 24px; width: 40px; height: 40px; border-radius: 20px; background: var(--icon-bg); display: flex; align-items: center; justify-content: center; font-size: 20px; cursor: pointer; user-select: none; transition: transform 0.2s; }
        .theme-toggle-icon:active { transform: scale(0.9); }
        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 1000; opacity: 0; pointer-events: none; transition: opacity 0.3s; }
        .overlay.open { opacity: 1; pointer-events: auto; }

        .container { padding: 5px 10px 10px 10px; }
        
        .global-search-container { display: flex; gap: 8px; margin: 5px 5px 15px 5px; align-items: center; background: var(--card-bg); padding: 5px; border-radius: 20px; border: 1px solid var(--border-color); box-shadow: 0 2px 8px rgba(0,0,0,0.02); }
        .search-input { flex: 1; padding: 8px 15px; border: none; outline: none; font-size: 14px; background: transparent; color: var(--text-main); }
        .search-btn { background: rgba(25,118,210,0.1); color: var(--primary-color); border: none; width: 36px; height: 36px; border-radius: 18px; font-weight: bold; font-size: 16px; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s ease; flex-shrink: 0; }
        .search-btn:active { background: var(--primary-color); color: #fff; }
        [data-theme="dark"] .search-btn { background: rgba(59,130,246,0.15); }
        
        .list-item { display: flex; gap: 12px; padding: 15px 5px; border-bottom: 1px solid var(--border-color); }
        .poster-wrapper { position: relative; flex-shrink: 0; width: 100px; height: 140px; }
        .poster { width: 100%; height: 100%; object-fit: cover; border-radius: 8px; background-color: #e0e0e0; transition: opacity 0.5s ease-in-out; }
        .tag-free { position: absolute; top: 0; left: 0; background: #22c55e; color: #fff; font-size: 10px; font-weight: bold; padding: 2px 6px; border-radius: 8px 0 8px 0; z-index: 10; }
        
        .info { flex-grow: 1; display: flex; flex-direction: column; overflow: hidden; justify-content: space-between; padding-bottom: 2px; }
        .title { font-size: 15px; font-weight: bold; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; line-height: 1.3; }
        .subtitle { font-size: 12px; color: var(--text-sub); line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; margin-bottom: 4px; }
        .subtitle:empty { display: none; margin: 0; }
        
        .stats-and-download { display: flex; justify-content: space-between; align-items: center; gap: 10px; width: 100%; margin-bottom: 4px; }
        .stats-row { font-size: 12px; color: var(--text-sub); display: flex; align-items: center; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; flex: 1; -webkit-overflow-scrolling: touch; }
        .stats-row::-webkit-scrollbar { display: none; }
        .stat-item { display: flex; align-items: center; gap: 2px; }
        .seed-icon { color: #22c55e; font-size: 10px; }
        .leech-icon { color: #ef4444; font-size: 10px; }
        
        .tags-wrapper { display: flex; flex-direction: column; gap: 6px; margin-top: auto; width: 100%; overflow: hidden; padding-top: 2px; }
        .tags-top { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; }
        .tags-top::-webkit-scrollbar { display: none; }
        .tags-top:empty { display: none; }
        
        .tags-bottom { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; width: 100%; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; padding-bottom: 2px; }
        .tags-bottom::-webkit-scrollbar { display: none; }
        .tags-bottom:empty { display: none; }
        
        .secondary-tag { display: none !important; }
        .show-extra .secondary-tag { display: inline-flex !important; }
        
        .badge-toggle { background: #e5e7eb; color: #6b7280; font-size: 10px; padding: 2px 8px; border-radius: 4px; flex-shrink: 0; cursor: pointer; display: inline-flex; align-items: center; letter-spacing: 1px; font-weight: bold; transition: all 0.2s ease;}
        [data-theme="dark"] .badge-toggle { background: #374151; color: #9ca3af; }
        .badge-toggle:active { opacity: 0.7; }
        
        .badge { background: #8b5cf6; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-discount { background: #ef4444; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-tech { background: #0ea5e9; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-lang { background: #10b981; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-group { background: #ec4899; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; letter-spacing: 0.5px;} 
        
        .badge-btshd { background: linear-gradient(135deg, #0284c7 0%, #38bdf8 100%); color: #fff; font-size: 10px; padding: 2px 4px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; box-shadow: 0 1px 3px rgba(2,132,199,0.3);}
        .badge-adweb { background: linear-gradient(135deg, #ef4444 0%, #991b1b 100%); color: #fff; font-size: 11px; padding: 2px 5px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; box-shadow: 0 1px 3px rgba(239,68,68,0.3);}

        .download-btn { background: var(--primary-color); color: #fff; font-size: 10px; padding: 2px 8px; border-radius: 4px; font-weight: bold; text-decoration: none; flex-shrink: 0; display: inline-flex; align-items: center; letter-spacing: 1px; min-width: 36px; justify-content: center;}
        .download-btn:active { opacity: 0.8; }
        
        .account-box { background: var(--card-bg); border-radius: 12px; padding: 20px; text-align: center; border: 1px solid var(--border-color); }
        .avatar { width: 70px; height: 70px; background: var(--primary-color); color: #fff; border-radius: 35px; line-height: 70px; font-size: 24px; margin: 0 auto 10px; }
        .account-name { font-size: 18px; font-weight: bold; }
        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }
        .stat-val { font-size: 18px; font-weight: bold; color: var(--primary-color); margin-bottom: 4px; }
        .stat-lbl { font-size: 12px; color: var(--text-sub); }
        .btn-signin { background: #22c55e; color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 25px; border: none; cursor: pointer; }
    </style>
</head>
<body>
    <script>
        function setTheme(theme) {
            const html = document.documentElement;
            if (theme === 'dark') { html.setAttribute('data-theme', 'dark'); }
            else if (theme === 'light') { html.removeAttribute('data-theme'); }
            else { 
                if (window.matchMedia('(prefers-color-scheme: dark)').matches) html.setAttribute('data-theme', 'dark');
                else html.removeAttribute('data-theme');
            }
        }
        let currentTheme = localStorage.getItem('btschool_theme') || 'auto';
        setTheme(currentTheme);
        window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
            if (localStorage.getItem('btschool_theme') === 'auto') setTheme('auto');
        });
        
        window.toggleTags = function(btn) {
            const wrapper = btn.closest('.tags-wrapper');
            wrapper.classList.toggle('show-extra');
            if (wrapper.classList.contains('show-extra')) {
                btn.innerText = '收起 ▴';
            } else {
                btn.innerText = '... ▾';
            }
        };
    </script>

    <div class="top-bar">
        <div class="menu-btn" id="menuBtn" onclick="toggleMenu()">
            <span></span><span></span><span></span>
        </div>
        <div class="logo"><span class="logo-icon">🏫</span> BTSCHOOL</div>
        <div class="top-placeholder"></div>
    </div>

    <div class="overlay" id="overlay" onclick="toggleMenu()"></div>
    <div class="sidebar" id="sidebar">
        <a href="/?tab=recent" class="menu-item {% if tab == 'recent' %}active{% endif %}">最近发布</a>
        <a href="/?tab=official" class="menu-item {% if tab == 'official' %}active{% endif %}">全部发布</a>
        <a href="/?tab=account" class="menu-item {% if tab == 'account' %}active{% endif %}">我的帐号</a>
        <div class="theme-toggle-icon" id="themeIcon" onclick="cycleTheme()">🔄</div>
    </div>

    <div class="container" id="list-container">
        {% if tab in ['official', 'recent', 'search'] %}
        <form class="global-search-container" action="/" method="GET">
            <input type="hidden" name="tab" value="search">
            <input type="text" name="q" class="search-input" placeholder="输入关键词搜全站..." value="{{ query }}">
            <button type="submit" class="search-btn">🔍</button>
        </form>
        {% endif %}

        {% if error %}<div style="text-align:center; color:#ef4444; margin-top:30px; font-weight:bold;">{{ error }}</div>{% endif %}

        {% if tab == 'account' and user_info %}
        <div class="account-box">
            <div class="avatar">{{ user_info.username[:1] | upper }}</div>
            <div class="account-name">{{ user_info.username }}</div>
            <div class="stat-grid">
                <div><div class="stat-val">{{ user_info.bonus }}</div><div class="stat-lbl">魔力值</div></div>
                <div><div class="stat-val">{{ user_info.ratio }}</div><div class="stat-lbl">分享率</div></div>
                <div><div class="stat-val">{{ user_info.upload }}</div><div class="stat-lbl">总上传</div></div>
                <div><div class="stat-val">{{ user_info.download }}</div><div class="stat-lbl">总下载</div></div>
            </div>
            <button id="signinBtn" class="btn-signin" onclick="doSignIn()">✅ 每日签到</button>
        </div>
        <script>
            function doSignIn() {
                const btn = document.getElementById('signinBtn');
                btn.innerText = '签到中...'; btn.disabled = true;
                fetch('/api/signin').then(r => r.json()).then(data => {
                    alert(data.msg); btn.innerText = '✅ 每日签到'; btn.disabled = false;
                }).catch(e => { alert('请求失败'); btn.innerText = '✅ 每日签到'; btn.disabled = false; });
            }
        </script>
        
        {% elif tab != 'account' %}
            {% for item in torrents %}
            <div class="list-item" id="item-{{ item.id }}">
                <div class="poster-wrapper">
                    {% set needs_lazy = not item.img or ('data:image/svg+xml' in item.img) %}
                    <img class="poster {% if needs_lazy %}lazy-load{% endif %}" 
                         src="{{ item.img if item.img else 'data:image/svg+xml;utf8,<svg xmlns=\\'http://www.w3.org/2000/svg\\' width=\\'100\\' height=\\'140\\'><rect width=\\'100%\\' height=\\'100%\\' fill=\\'%23e6e6e6\\'/><text x=\\'50%\\' y=\\'50%\\' fill=\\'%23999\\' font-size=\\'12\\' font-family=\\'sans-serif\\' text-anchor=\\'middle\\' dominant-baseline=\\'middle\\'>加载海报</text></svg>' }}" 
                         {% if needs_lazy %}data-tid="{{ item.id }}"{% endif %}
                         referrerpolicy="no-referrer" alt="海报" onclick="window.open('{{ item.link }}', '_blank')">
                    {% if 'FREE' in item.tags|join(' ')|upper %}
                        <div class="tag-free">FREE</div>
                    {% endif %}
                </div>
                <div class="info">
                    <div class="title" onclick="window.open('{{ item.link }}', '_blank')">{{ item.title }}</div>
                    <div class="subtitle">{{ item.subtitle }}</div>
                    
                    <div class="stats-and-download">
                        <div class="stats-row">
                            <span class="stat-item" id="size-{{ item.id }}">📄 {{ item.size }}</span>
                            <span class="stat-item"><span class="seed-icon">▲</span> {{ item.seeders }}</span>
                            <span class="stat-item"><span class="leech-icon">▼</span> {{ item.leechers }}</span>
                            {% if item.group %}
                                {% if item.group|upper == 'BTSHD' %}
                                    <span class="badge-btshd">💎 BtsHD</span>
                                {% elif item.group|upper == 'ADWEB' %}
                                    <span class="badge-adweb">◆</span>
                                {% else %}
                                    <span class="badge-group">{{ item.group }}</span>
                                {% endif %}
                            {% endif %}
                        </div>
                        <a href="/download/{{ item.id }}" class="download-btn">下载</a>
                    </div>

                    <div class="tags-wrapper">
                        <div class="tags-top">
                            {% for tag in item.top_pri %}
                                <span class="badge-tech">{{ tag }}</span>
                            {% endfor %}
                            {% for tag in item.top_sec %}
                                <span class="badge-tech secondary-tag">{{ tag }}</span>
                            {% endfor %}
                        </div>

                        <div class="tags-bottom">
                            {% if item.time_tag %}
                                <span class="badge-discount">{{ item.time_tag }}</span>
                            {% endif %}
                            
                            {% if item.clock_tag %}
                                <span class="badge" style="background:#64748b;">{{ item.clock_tag }}</span>
                            {% endif %}
                            
                            {% for tag in item.bot_disc %}
                                <span class="badge-discount">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_pri %}
                                <span class="badge">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_sec %}
                                <span class="badge-lang secondary-tag">{{ tag }}</span>
                            {% endfor %}
                            
                            {% if item.top_sec|length > 0 or item.bot_sec|length > 0 %}
                                <span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>
                            {% endif %}
                        </div>
                    </div>
                </div>
            </div>
            {% endfor %}
            
            {% if tab == 'search' and not torrents and query %}
                <div style="text-align:center; color:var(--text-sub); margin-top:30px;">找到 0 个结果</div>
            {% elif tab in ['search', 'recent', 'official'] and not error %}
                <!-- 💡添加了触发无限滚动的观察点 -->
                <div id="load-more-sentinel" style="text-align:center; padding: 20px; color:var(--text-sub); font-size:12px; clear:both;">
                    向下划动，无痕加载更多... ⚡
                </div>
            {% endif %}
        {% endif %}
    </div>

    <script>
        const sidebar = document.getElementById('sidebar');
        const overlay = document.getElementById('overlay');
        const menuBtn = document.getElementById('menuBtn');
        const themeIcon = document.getElementById('themeIcon');

        function toggleMenu() { 
            sidebar.classList.toggle('open'); 
            overlay.classList.toggle('open'); 
            menuBtn.classList.toggle('open');
        }

        const icons = { 'auto': '🔄', 'dark': '🌙', 'light': '☀️' };
        themeIcon.innerText = icons[currentTheme];

        function cycleTheme() {
            if (currentTheme === 'auto') currentTheme = 'dark';
            else if (currentTheme === 'dark') currentTheme = 'light';
            else currentTheme = 'auto';
            localStorage.setItem('btschool_theme', currentTheme);
            setTheme(currentTheme);
            themeIcon.innerText = icons[currentTheme];
        }

        document.addEventListener("DOMContentLoaded", function() {
            if ('IntersectionObserver' in window) {
                const observer = new IntersectionObserver((entries, obs) => {
                    entries.forEach(entry => {
                        if (entry.isIntersecting) {
                            const img = entry.target;
                            const tid = img.getAttribute('data-tid');
                            if (tid) {
                                fetch('/api/extra/' + tid)
                                    .then(r => r.json())
                                    .then(data => {
                                        // 💡 防卡死：如果没图，显示暂无海报
                                        if (data.img) {
                                            img.src = data.img;
                                        } else {
                                            img.src = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23f4f4f5'/><text x='50%' y='50%' fill='%23a1a1aa' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>暂无海报</text></svg>";
                                        }
                                        img.classList.remove('lazy-load');
                                        
                                        if (data.size && data.size !== '未知' && data.size !== '加载中...') {
                                            const sizeSpan = document.getElementById('size-' + tid);
                                            if (sizeSpan) sizeSpan.innerHTML = '📄 ' + data.size;
                                        }
                                        
                                        const itemDiv = document.getElementById('item-' + tid);
                                        if (itemDiv && data.tags) {
                                            if (data.group && !itemDiv.querySelector('.badge-group') && !itemDiv.querySelector('.badge-btshd') && !itemDiv.querySelector('.badge-adweb')) {
                                                const statsRow = itemDiv.querySelector('.stats-row');
                                                if (statsRow) {
                                                    if (data.group.toUpperCase() === 'BTSHD') {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-btshd">💎 BtsHD</span>`);
                                                    } else if (data.group.toUpperCase() === 'ADWEB') {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-adweb">◆</span>`);
                                                    } else {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-group">${data.group}</span>`);
                                                    }
                                                }
                                            }
                                            
                                            const tagsWrapper = itemDiv.querySelector('.tags-wrapper');
                                            if (tagsWrapper && data.top_pri) {
                                                let hasFree = !!itemDiv.querySelector('.tag-free');
                                                if (data.tags.some(t => t.toUpperCase() === 'FREE' || t === '免费') && !hasFree) {
                                                    itemDiv.querySelector('.poster-wrapper').insertAdjacentHTML('beforeend', '<div class="tag-free">FREE</div>');
                                                }
                                                
                                                let html = '<div class="tags-top">';
                                                data.top_pri.forEach(t => html += `<span class="badge-tech">${t}</span>`);
                                                data.top_sec.forEach(t => html += `<span class="badge-tech secondary-tag">${t}</span>`);
                                                html += '</div><div class="tags-bottom">';
                                                
                                                if (data.time_tag) html += `<span class="badge-discount">${data.time_tag}</span>`;
                                                if (data.clock_tag) html += `<span class="badge" style="background:#64748b;">${data.clock_tag}</span>`;
                                                
                                                data.bot_disc.forEach(t => { 
                                                    if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge-discount">${t}</span>`; 
                                                });
                                                data.bot_pri.forEach(t => { 
                                                    if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge">${t}</span>`; 
                                                });
                                                data.bot_sec.forEach(t => html += `<span class="badge-lang secondary-tag">${t}</span>`);
                                                
                                                if (data.top_sec.length > 0 || data.bot_sec.length > 0) {
                                                    html += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                                }
                                                html += '</div>';
                                                tagsWrapper.innerHTML = html;
                                            }
                                        }
                                    })
                                    .catch(e => console.error('懒加载失败:', e));
                                    
                                img.removeAttribute('data-tid');
                                obs.unobserve(img);
                            }
                        }
                    });
                }, { rootMargin: "300px" });

                document.querySelectorAll('.lazy-load').forEach(img => {
                    observer.observe(img);
                });

                // 💡前端无穷下拉加载引擎
                const sentinel = document.getElementById('load-more-sentinel');
                if (sentinel) {
                    let currentPage = 1; 
                    let isLoadingMore = false;
                    let hasMore = true;
                    const seenIds = new Set();
                    document.querySelectorAll('.list-item').forEach(el => seenIds.add(el.id.replace('item-', '')));

                    const loadMoreObserver = new IntersectionObserver((entries) => {
                        const entry = entries[0];
                        if (entry.isIntersecting && !isLoadingMore && hasMore) {
                            loadMore();
                        }
                    }, { rootMargin: "200px" });
                    
                    loadMoreObserver.observe(sentinel);

                    function loadMore() {
                        isLoadingMore = true;
                        sentinel.innerText = "正在无痕加载下一页...";
                        
                        const params = new URLSearchParams(window.location.search);
                        const tab = params.get('tab') || 'recent'; // 💡 严格遵循你的默认选项
                        const q = params.get('q') || '';
                        
                        fetch(`/api/load_more?tab=${tab}&q=${encodeURIComponent(q)}&page=${currentPage}`)
                            .then(r => r.json())
                            .then(data => {
                                currentPage++;
                                isLoadingMore = false;
                                
                                if (data.error) {
                                    sentinel.innerText = "加载失败: " + data.error + " (点击重试)";
                                    sentinel.onclick = loadMore;
                                    return;
                                }
                                
                                if (data.torrents.length === 0) {
                                    hasMore = false;
                                    sentinel.innerText = "没有更多数据了";
                                    return;
                                }
                                
                                let addedCount = 0;
                                data.torrents.forEach(item => {
                                    if (seenIds.has(item.id)) return;
                                    seenIds.add(item.id);
                                    addedCount++;
                                    
                                    let tagsTopHtml = '';
                                    item.top_pri.forEach(t => tagsTopHtml += `<span class="badge-tech">${t}</span>`);
                                    item.top_sec.forEach(t => tagsTopHtml += `<span class="badge-tech secondary-tag">${t}</span>`);
                                    
                                    let tagsBotHtml = '';
                                    if (item.time_tag) tagsBotHtml += `<span class="badge-discount">${item.time_tag}</span>`;
                                    if (item.clock_tag) tagsBotHtml += `<span class="badge" style="background:#64748b;">${item.clock_tag}</span>`;
                                    item.bot_disc.forEach(t => tagsBotHtml += `<span class="badge-discount">${t}</span>`);
                                    item.bot_pri.forEach(t => tagsBotHtml += `<span class="badge">${t}</span>`);
                                    item.bot_sec.forEach(t => tagsBotHtml += `<span class="badge-lang secondary-tag">${t}</span>`);
                                    
                                    if (item.top_sec.length > 0 || item.bot_sec.length > 0) {
                                        tagsBotHtml += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                    }

                                    let groupHtml = '';
                                    if (item.group) {
                                        if (item.group.toUpperCase() === 'BTSHD') {
                                            groupHtml = `<span class="badge-btshd">💎 BtsHD</span>`;
                                        } else if (item.group.toUpperCase() === 'ADWEB') {
                                            groupHtml = `<span class="badge-adweb">◆</span>`;
                                        } else {
                                            groupHtml = `<span class="badge-group">${item.group}</span>`;
                                        }
                                    }
                                    
                                    let freeTagHtml = '';
                                    if (item.tags.join(' ').toUpperCase().includes('FREE')) {
                                        freeTagHtml = `<div class="tag-free">FREE</div>`;
                                    }

                                    const imgPlaceholder = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23e6e6e6'/><text x='50%' y='50%' fill='%23999' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>加载海报</text></svg>";
                                    const imgSrc = item.img ? item.img : imgPlaceholder;
                                    const needsLazy = !item.img || item.img.includes('data:image/svg+xml');

                                    const html = `
                                    <div class="list-item" id="item-${item.id}">
                                        <div class="poster-wrapper">
                                            <img class="poster ${needsLazy ? 'lazy-load' : ''}" 
                                                 src="${imgSrc}" 
                                                 ${needsLazy ? `data-tid="${item.id}"` : ''}
                                                 referrerpolicy="no-referrer" alt="海报" onclick="window.open('${item.link}', '_blank')">
                                            ${freeTagHtml}
                                        </div>
                                        <div class="info">
                                            <div class="title" onclick="window.open('${item.link}', '_blank')">${item.title}</div>
                                            <div class="subtitle">${item.subtitle || ''}</div>
                                            <div class="stats-and-download">
                                                <div class="stats-row">
                                                    <span class="stat-item" id="size-${item.id}">📄 ${item.size}</span>
                                                    <span class="stat-item"><span class="seed-icon">▲</span> ${item.seeders}</span>
                                                    <span class="stat-item"><span class="leech-icon">▼</span> ${item.leechers}</span>
                                                    ${groupHtml}
                                                </div>
                                                <a href="/download/${item.id}" class="download-btn">下载</a>
                                            </div>
                                            <div class="tags-wrapper">
                                                <div class="tags-top">${tagsTopHtml}</div>
                                                <div class="tags-bottom">${tagsBotHtml}</div>
                                            </div>
                                        </div>
                                    </div>
                                    `;
                                    sentinel.insertAdjacentHTML('beforebegin', html);
                                });
                                
                                document.querySelectorAll('.lazy-load:not([data-observed])').forEach(img => {
                                    img.setAttribute('data-observed', '1');
                                    observer.observe(img);
                                });
                                
                                if (addedCount === 0) {
                                    loadMore(); 
                                } else {
                                    sentinel.innerText = "向下划动，无痕加载更多... ⚡";
                                    sentinel.onclick = null;
                                }
                            })
                            .catch(e => {
                                isLoadingMore = false;
                                sentinel.innerText = "网络错误，点击重试";
                                sentinel.onclick = loadMore;
                            });
                    }
                }
            }
        });
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    # 💡【精准修改】：改回了你的默认页面 recent
    tab = request.args.get('tab', 'recent')
    query = request.args.get('q', '')
    torrents = []
    user_info = {}
    seen_ids = set() 
    error_msg = None
    
    try:
        if tab == 'account':
            resp = requests.get(INDEX_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效，请在代码中更新 Cookie！"
            else:
                soup = BeautifulSoup(resp.text, 'html.parser')
                for tag in soup.find_all(True):
                    title_attr = tag.get('title') or tag.get('alt') or tag.get('data-original-title')
                    if title_attr and isinstance(title_attr, str) and len(title_attr) < 20:
                        tag.append(f"{title_attr}")
                page_text = soup.get_text(separator='', strip=True)
                
                user_match = re.search(r'欢迎回来[\s,，]*(.*?)(?:\[|退出|魔力值|分享率)', page_text)
                bonus_match = re.search(r'魔力值[^\d]*([\d\.,]+)', page_text)
                ratio_match = re.search(r'分享率[^\d]*([\d\.,]+|无限|Inf)', page_text, re.IGNORECASE)
                up_match = re.search(r'(?:上传量?|上传|上行)[^\d]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                down_match = re.search(r'(?:下载量?|下载|下行)[^\d]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                
                user_info = {
                    'username': user_match.group(1).strip() if user_match else "同学",
                    'bonus': bonus_match.group(1) if bonus_match else "0",
                    'ratio': ratio_match.group(1) if ratio_match else "0",
                    'upload': up_match.group(1) if up_match else "0 GB",
                    'download': down_match.group(1) if down_match else "0 GB"
                }

        elif tab in ['official', 'search']:
            if tab == 'search' and not query: pass
            else:
                if tab == 'search':
                    safe_query = urllib.parse.quote(query)
                    target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
                else: 
                    target_url = OFFICIAL_URL
                    
                fetched_torrents, err = parse_torrents_page(target_url, seen_ids, is_search=(tab == 'search'), skip_sticky=False)
                if fetched_torrents is not None:
                    torrents = fetched_torrents
                else:
                    if err: error_msg = err
                
                if torrents and os.path.exists(HISTORY_FILE):
                    try:
                        with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history = json.load(f)
                        history_dict = {str(item['id']): item for item in history}
                        for item in torrents:
                            tid_str = str(item['id'])
                            if tid_str in history_dict:
                                cached = history_dict[tid_str]
                                if 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                                if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                                if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                                item['tags'] = purify_tags(merge_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                    except: pass

        elif tab == 'recent':
            fetched_torrents, err_msg = parse_torrents_page(OFFICIAL_URL, seen_ids, is_search=False, skip_sticky=True)
            
            history_data = []
            if os.path.exists(HISTORY_FILE):
                try:
                    with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history_data = json.load(f)
                except: pass
            
            if fetched_torrents is not None:
                history_dict = {str(item['id']): item for item in history_data}
                for item in fetched_torrents:
                    tid_str = str(item['id'])
                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                        item['tags'] = purify_tags(merge_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                    history_dict[tid_str] = item
                    
                merged = list(history_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = merged[:50]
                try:
                    with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(torrents, f, ensure_ascii=False, indent=2)
                except: pass
            else:
                torrents = history_data[:50]
                if err_msg: error_msg = err_msg

        if torrents:
            for item in torrents:
                tpri, tsec, bdisc, bpri, bsec = sort_tags(item.get('tags', []))
                item['top_pri'] = tpri
                item['top_sec'] = tsec
                item['bot_disc'] = bdisc
                item['bot_pri'] = bpri
                item['bot_sec'] = bsec
                item['time_tag'] = ""
                item['clock_tag'] = ""
                
                for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                    for t in list(arr):
                        if '剩余' in t.upper():
                            item['time_tag'] = t
                            arr.remove(t)
                        elif '🕒' in t:
                            item['clock_tag'] = t
                            arr.remove(t)

    except Exception as e:
        error_msg = f"服务器内部请求错误: {e}"
        torrents = torrents or []
        
    return render_template_string(HTML_TEMPLATE, tab=tab, query=query, torrents=torrents or [], user_info=user_info, error=error_msg)

# 💡【全新接口】：无缝下拉接口，直接请求服务器的下一页
@app.route('/api/load_more')
def api_load_more():
    tab = request.args.get('tab', 'recent')
    query = request.args.get('q', '')
    try: page = int(request.args.get('page', 1))
    except: page = 1
        
    seen_ids = set()
    err = None
    fetched = []
    
    if tab == 'search':
        safe_query = urllib.parse.quote(query)
        target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
        fetched, err = parse_torrents_page(target_url, seen_ids, is_search=True, skip_sticky=False, start_page=page, max_pages=1)
    elif tab == 'official':
        fetched, err = parse_torrents_page(OFFICIAL_URL, seen_ids, is_search=False, skip_sticky=False, start_page=page, max_pages=1)
    else: 
        # SchoolPT 的“最近发布”，直接去服务器端请求去掉置顶项的新一页！
        fetched, err = parse_torrents_page(OFFICIAL_URL, seen_ids, is_search=False, skip_sticky=True, start_page=page, max_pages=1)
        
    if fetched:
        history_dict = load_history()
        for item in fetched:
            tid_str = str(item['id'])
            if tid_str in history_dict:
                cached = history_dict[tid_str]
                if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                item['tags'] = purify_tags(merge_tags_from_list(item.get('tags', []), cached.get('tags', [])))

            tpri, tsec, bdisc, bpri, bsec = sort_tags(item.get('tags', []))
            item['top_pri'] = tpri
            item['top_sec'] = tsec
            item['bot_disc'] = bdisc
            item['bot_pri'] = bpri
            item['bot_sec'] = bsec
            item['time_tag'] = ""
            item['clock_tag'] = ""
            
            for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                for t in list(arr):
                    if '剩余' in t.upper():
                        item['time_tag'] = t
                        arr.remove(t)
                    elif '🕒' in t:
                        item['clock_tag'] = t
                        arr.remove(t)
                        
    return jsonify({"torrents": fetched or [], "error": err})

@app.route('/api/extra/<int:tid>')
def api_extra(tid):
    img_src, size, extra_tags, release_group = fetch_details_info(tid)
    merged_tags = purify_tags(extra_tags)
    
    if os.path.exists(HISTORY_FILE):
        try:
            with open(HISTORY_FILE, 'r', encoding='utf-8') as f: history = json.load(f)
            updated = False
            for item in history:
                if str(item['id']) == str(tid):
                    if img_src: item['img'] = img_src
                    if size and size != "未知": item['size'] = size
                    if release_group and not item.get('group'): item['group'] = release_group
                    
                    merged_tags = purify_tags(merge_tags_from_details(merged_tags, item.get('tags', [])))
                    item['tags'] = merged_tags
                    
                    updated = True
                    break
            if updated:
                with open(HISTORY_FILE, 'w', encoding='utf-8') as f: json.dump(history, f, ensure_ascii=False, indent=2)
        except: pass
        
    tpri, tsec, bdisc, bpri, bsec = sort_tags(merged_tags)
    time_tag = ""
    clock_tag = ""
    for arr in [tpri, tsec, bdisc, bpri, bsec]:
        for t in list(arr):
            if '剩余' in t.upper():
                time_tag = t
                arr.remove(t)
            elif '🕒' in t:
                clock_tag = t
                arr.remove(t)
    
    return jsonify({
        "img": img_src, 
        "size": size, 
        "tags": merged_tags,
        "top_pri": tpri,
        "top_sec": tsec,
        "bot_disc": bdisc,
        "bot_pri": bpri,
        "bot_sec": bsec,
        "time_tag": time_tag,
        "clock_tag": clock_tag,
        "group": release_group
    })

@app.route('/api/signin')
def api_signin():
    try:
        r = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        r.encoding = 'utf-8'
        res_text = r.text
        if r.status_code == 200:
            if "您今天已经签到过了" in res_text or "请明天再来" in res_text or "已经" in res_text:
                return jsonify({"msg": "☕ 您今天已经签到过了，明天再来吧！"})
            elif "签到成功" in res_text or "魔力值" in res_text or "次签到" in res_text or "退下" in res_text:
                return jsonify({"msg": "🎉 签到成功！魔力值已到账。"})
            elif "登录" in res_text:
                return jsonify({"msg": "⚠️ 签到失败：Cookie 已失效，请更新！"})
            else:
                return jsonify({"msg": "✅ 签到请求已发送，请在个人面板留意魔力值变化。"})
        else:
            return jsonify({"msg": f"❌ 签到接口报错，状态码: {r.status_code}"})
            
    except Exception as e: 
        return jsonify({"msg": f"❌ 网络请求出错: {str(e)}"})

@app.route('/download/<int:tid>')
def download_torrent(tid):
    try:
        r = requests.get(f"{BASE_URL}/download.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        if "application/x-bittorrent" in r.headers.get('Content-Type', '') or "octet-stream" in r.headers.get('Content-Type', ''):
            return Response(r.content, mimetype="application/x-bittorrent", headers={"Content-Disposition": f"attachment;filename=BTSchool_{tid}.torrent"})
        else: return "下载失败，Cookie 可能失效或无权下载该种子。"
    except Exception as e: return f"请求下载出错: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5053)
```

8.h5zm.py
```
from flask import Flask, render_template_string, Response, request, jsonify
import requests
from bs4 import BeautifulSoup
import urllib.parse
import re
import os
import json
import threading
import time
import datetime

app = Flask(__name__)

# ================= 必须配置的区域 =================
HEADERS = {
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    # 【安全提醒】：请把你的 ZmPT (织梦) Cookie 填在下面
    'Cookie': 'cf_clearance=5ZBTxMjwbWjt0CdRG0MCruVac3dFptwGijOurXms3eM-1791902735-1.2.1.1-hswQ7D0XbIKhyq8Pkpwd4.i01CwuzGwxO9BGSMvIYp9D15bgjy8yQwpD_dK_c3C5YbF0Fun7XHoCxrdkRmH.b9Z7MwJ0gbx9H6bIa89Ns4zVXmGX_PohPE2a5etU3eEvuHFY9551hnEuJU0x7sC1.rGjWjuz9vodqJMkFYgWhKf2TW_tM9jjV.dStjka3EJ0iJpw40lyKyOf.e4HN.MUbquV5yEaQk6mN5wdooTneHscUignAhTsNSk4bqWf6_Lby6ahljaxbT6Z1I_dFesFlmIEFBQiJgJZfTfF1uhpx0bGkOzd_2fAvXm_uwADLvFYkZF93Up6ruQfuJRElE3eTaSJvlG644odsDRVsjmfQcdVTN_gkOb83m7jLPfJDWSx8sYv9HYkloeXsBTJAJfUFuTMDxgOpFZutH3XqkKffjpR8Nm5cdhc7y9lJkGhockX7I29vCJH41BhepoZVhh5Ddoupx_xxMQ.xsI.FrF5ZQZM9FAZJ09v.K2qkKonJl7ctshh3lAW_ipLIk.vcL2WKA; c_secure_pass=eyJ1c2VyX2lkIjoyNTQ4MSwiZXhwaXJlcyI6MTgyMzQzODc3MH0uNDY0YzZiOGJiZDEzODM2Nzc1NDhjOTgzMDdhOWFmNWFkMGJiMWM3ZTQwZTYwMTkyMGUzMWYyY2NmZWJmZWEwYg%3D%3D' 
}

PROXY_URL = ""
PROXIES = {"http": PROXY_URL, "https": PROXY_URL} if PROXY_URL else None
TIMEOUT = 30

BASE_URL = "https://zmpt.cc"

INDEX_URL = f"{BASE_URL}/index.php"
COMPREHENSIVE_URL = f"{BASE_URL}/torrents.php"
OFFICIAL_URL = f"{BASE_URL}/torrents.php?tag_id=3" 
SIGNIN_URL = f"{BASE_URL}/attendance.php"

# 织梦专属缓存文件
OFFICIAL_HISTORY_FILE = "zmpt_official_history.json"
RECENT_HISTORY_FILE = "zmpt_recent_history.json"
# =================================================

MEM_CACHE_OFFICIAL = {}
MEM_CACHE_RECENT = {}
CACHE_LOCK = threading.Lock()

def load_cache(cache_type='official'):
    global MEM_CACHE_OFFICIAL, MEM_CACHE_RECENT
    filename = OFFICIAL_HISTORY_FILE if cache_type == 'official' else RECENT_HISTORY_FILE
    target_cache = MEM_CACHE_OFFICIAL if cache_type == 'official' else MEM_CACHE_RECENT
    
    with CACHE_LOCK:
        if not target_cache and os.path.exists(filename):
            try:
                with open(filename, 'r', encoding='utf-8') as f:
                    data = json.load(f)
                    res = {str(item['id']): item for item in data}
                    if cache_type == 'official': MEM_CACHE_OFFICIAL = res
                    else: MEM_CACHE_RECENT = res
                    return res
            except Exception: pass
    return target_cache

def save_cache(cache_dict, cache_type='official'):
    global MEM_CACHE_OFFICIAL, MEM_CACHE_RECENT
    filename = OFFICIAL_HISTORY_FILE if cache_type == 'official' else RECENT_HISTORY_FILE
    with CACHE_LOCK:
        if cache_type == 'official': MEM_CACHE_OFFICIAL = cache_dict.copy()
        else: MEM_CACHE_RECENT = cache_dict.copy()
        
        merged = list(cache_dict.values())
        merged.sort(key=lambda x: int(x['id']), reverse=True)
        try:
            with open(filename, 'w', encoding='utf-8') as f:
                json.dump(merged[:50], f, ensure_ascii=False, indent=2)
        except Exception: pass

TECH_KEYWORDS = [
    'FREE', '50%', '30%', '2X', '热门', '官方', '中字', '英字', '国语', '粤语', '英语', 
    '简繁', '双语', '杜比', '高帧', '高码', '臻彩MAX', '高分', '完结', '4K', '2160P', '1440P', '1080P', '720P', 
    'WEB-DL', 'BLU-RAY', 'REMUX', 'HDR10+', 'HDR10', 'HDRVIVID', 
    'HDR', 'DOVI', 'DV', 'SDR', '60FPS', 'HFR', 'HQ', 'H.265', 'H.264', 'H265', 'H264', 
    'HEVC', 'AAC', 'DDP', 'ATMOS', '10BIT', '首发', '禁转', '应求', 'DIY', '字幕'
]

def purify_tags(tags):
    valid = []
    for t in tags:
        clean_t = t.upper()
        if '剩余' in clean_t or '🕒' in clean_t:
            valid.append(t)
        elif clean_t in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', clean_t):
            valid.append(t)
            
    final_tags = []
    time_tags = [t for t in valid if '剩余' in t]
    best_time = max(time_tags, key=len) if time_tags else None
    
    clock_tags = [t for t in valid if '🕒' in t]
    best_clock = clock_tags[-1] if clock_tags else None
    
    if best_time and best_clock:
        pure_time = re.sub(r'^.*?(?:剩余时间|免费剩余)[:：\s]*', '', best_time).strip()
        pure_clock = best_clock.replace('🕒', '').strip()
        if pure_time == pure_clock:
            best_clock = None
            
    for t in valid:
        if '剩余' in t:
            if t == best_time and best_time not in final_tags:
                if not re.match(r'^剩余时间[:：\s]*\d+\s*分$', t):
                    final_tags.append(t)
        elif '🕒' in t:
            if t == best_clock and best_clock not in final_tags:
                final_tags.append(t)
        elif t not in final_tags:
            final_tags.append(t)
            
    return final_tags

def merge_smart_tags_from_list(list_tags, cached_tags):
    res = list(list_tags)
    for t in cached_tags:
        t_up = t.upper()
        is_dynamic = '%' in t_up or '2X' in t_up or t_up in ['FREE', '免费'] or '剩余' in t_up
        if not is_dynamic and t not in res:
            res.append(t)
    return res

def merge_smart_tags_from_details(details_tags, cached_tags):
    res = list(details_tags)
    has_discount = any('%' in x.upper() or '2X' in x.upper() for x in res)
    has_free = any(x.upper() in ['FREE', '免费'] for x in res)
    has_time = any('剩余' in x for x in res)
    has_clock = any('🕒' in x for x in res)
    
    for t in cached_tags:
        if t in res: continue
        t_up = t.upper()
        if '%' in t_up or '2X' in t_up:
            if not has_discount: res.append(t)
        elif t_up in ['FREE', '免费']:
            if not has_free: res.append(t)
        elif '剩余' in t_up:
            if not has_time: res.append(t)
        elif '🕒' in t_up:
            if not has_clock: res.append(t)
        else:
            res.append(t)
    return res

def auto_extract_tags(text, current_tags):
    en_matches = re.findall(r'(?i)\b(4K|2160P|1440P|1080P|720P|WEB-DL|BLU-RAY|REMUX|HDR10\+|HDR10|HDRVIVID|HDR|DOVI|DV|SDR|60FPS|HFR|HQ|H\.?265|H\.?264|HEVC|AAC|DDP|ATMOS|10BIT)\b', text)
    cn_matches = re.findall(r'(杜比|高帧|高码|臻彩MAX|高分|完结|国语|中字|英字|粤语|英语|简繁|双语|官方|首发)', text)
    disc_matches = re.findall(r'(50%|30%|2X)', text.upper())
    
    all_new = [m.upper() for m in en_matches] + cn_matches + disc_matches
    for tag in all_new:
        tag_up = tag.upper()
        if not any(tag_up in t.upper() or t.upper() in tag_up for t in current_tags):
            current_tags.append(tag)
    return current_tags

def sort_tags(tags):
    top_pri, top_sec, bot_disc, bot_pri, bot_sec = [], [], [], [], []
    remaining = list(tags)
    
    t_res = ['4K', '2160P', '1440P', '1080P', '720P']
    t_hdr = ['HDRVIVID', 'HDR10+', 'HDR10', 'HDR', 'DOVI', 'DV', 'SDR']
    t_fps = ['60FPS', 'HFR', '高帧', '臻彩MAX', 'BLU-RAY', 'REMUX']
    t_enc = ['WEB-DL', 'H.265', 'H.264', 'H265', 'H264', 'HEVC', 'AAC', 'DDP', 'ATMOS', '杜比', '10BIT', 'HQ', '高码']
    
    for t in list(remaining):
        t_upper = t.upper()
        if '%' in t_upper or '2X' in t_upper or t_upper in ['免费', 'FREE', '30%', '50%']:
            if t_upper not in ['FREE', '免费']:
                bot_disc.append(t)
            remaining.remove(t)
    
    t_status = ['热门', '完结', '首发', '高分', 'DIY']
    t_lang = ['国语', '中字', '英字', '粤语', '英语', '简繁', '双语', '官方', '字幕']
    
    for kw in t_res + t_hdr + t_fps:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_pri.append(t)
                remaining.remove(t)
                
    for kw in t_enc:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                top_sec.append(t)
                remaining.remove(t)
                
    for kw in t_status:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_pri.append(t)
                remaining.remove(t)
                
    for kw in t_lang:
        for t in list(remaining):
            if kw in t.upper() and t in remaining:
                bot_sec.append(t)
                remaining.remove(t)
                
    for t in list(remaining):
        bot_pri.append(t)
        
    return top_pri, top_sec, bot_disc, bot_pri, bot_sec

def fetch_details_info(tid):
    try:
        r = requests.get(f"{BASE_URL}/details.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        raw_html = r.text
        soup = BeautifulSoup(raw_html, 'html.parser')
        extra_tags = []

        is_global_free = bool(re.search(r'(?:全站|全局).*?(?:Free|免费)', raw_html, re.IGNORECASE))
        if is_global_free: extra_tags.append('Free')

        time_match = re.search(r'(?:免费剩余时间|免费剩余|优惠剩余时间|剩余时间|还有|剩余)[^\d<]{0,30}(?:<[^>]+>)*\s*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', raw_html)
        if time_match:
            extra_tags.append(f"剩余时间: {time_match.group(1).strip()}")
        else:
            text_clean = soup.get_text(separator=' ')
            tm2 = re.search(r'(?:免费剩余时间|免费剩余|优惠剩余时间|剩余时间|还有|剩余)[:：\s]+(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', text_clean)
            if tm2: extra_tags.append(f"剩余时间: {tm2.group(1).strip()}")

        img_src = ""
        descr_div = soup.find('div', id='kdescr')
        if descr_div:
            img = descr_div.find('img')
            if img: img_src = img.get('data-src') or img.get('src') or ""
        if not img_src:
            for img in soup.find_all('img'):
                src = img.get('data-src') or img.get('src') or ""
                if src and not any(x in src.lower() for x in ['category', 'smilies', 'icon', 'logo', 'avatar', 'rating', 'button', 'rule']):
                    img_src = src
                    break
        if img_src and not img_src.startswith('http'):
            img_src = f"{BASE_URL}/" + img_src

        size = "未知"
        size_match = re.search(r'大小.*?([\d\.]+\s*[KMGTP]B)', raw_html, re.IGNORECASE)
        if size_match: size = size_match.group(1)

        h1 = soup.find('h1')
        release_group = ""
        if h1:
            h1_text = h1.get_text(strip=True)
            for kw in ['免费', 'FREE', '50%', '30%', '2X']:
                if kw in h1_text.upper():
                    if kw in ['免费', 'FREE']:
                        if 'Free' not in extra_tags: extra_tags.append('Free')
                    else:
                        if kw not in extra_tags: extra_tags.append(kw)
            
            for span in h1.find_all('span', class_=re.compile(r'tag|label|badge', re.I)):
                text = span.get_text(strip=True).replace('[', '').replace(']', '').strip()
                if text.upper() in ['FREE', '免费']:
                    if 'Free' not in extra_tags: extra_tags.append('Free')
                elif text and text not in extra_tags and len(text) <= 15:
                    extra_tags.append(text)
                    
            group_match = re.search(r'-([A-Za-z0-9@_]+)$', h1_text)
            if group_match:
                release_group = group_match.group(1).upper()

        sub_td = soup.find('td', class_='rowhead', string=re.compile('副标题|副標題'))
        sub_text = ""
        if sub_td:
            sub_text = sub_td.find_next_sibling('td').get_text(strip=True)
            
        extra_tags = auto_extract_tags(h1_text + " " + sub_text, extra_tags)
        
        page_clean_text = soup.get_text(separator=' ')
        pub_exact = re.search(r'(?:发布时间|添加时间|上传时间|创建时间)[^\d]*(\d{4}-\d{2}-\d{2}\s*\d{2}:\d{2}(?::\d{2})?)', page_clean_text)
        if pub_exact:
            extra_tags.append(f"🕒 {pub_exact.group(1)}")
        else:
            pub_rel = re.search(r'发布于\s*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)\s*前', page_clean_text)
            if pub_rel:
                extra_tags.append(f"🕒 {re.sub(r'\s+', '', pub_rel.group(1))}前")
        
        return img_src, size, purify_tags(extra_tags), release_group
    except Exception as e:
        return "", "未知", [], ""

def is_real_sticky(row):
    for img in row.find_all('img'):
        src = img.get('src', '').lower()
        alt = img.get('alt', '').lower()
        title = img.get('title', '').lower()
        cls = ' '.join(img.get('class', [])).lower()
        if 'sticky' in cls: return True
        if alt in ['sticky', '置顶'] or title in ['sticky', '置顶']: return True
        if any(x in src for x in ['sticky.gif', 'pin.gif', 'pro_sticky.gif']): return True
        
    for tag in row.find_all(['span', 'font', 'b']):
        t = tag.get_text(strip=True).upper()
        if t in ['[置顶]', '置顶', '[STICKY]', 'STICKY']: return True
        if 'sticky' in ' '.join(tag.get('class', [])).lower(): return True
            
    tr_cls = ' '.join(row.get('class', [])).lower()
    if any(x in tr_cls for x in ['sticky_bg', 'sticky_blank', 'pro_sticky']): return True
    return False

# 💡【核心升级】：支持指定页码单次抓取的底层引擎
def parse_torrents_page(target_url_base, seen_ids, is_search=False, skip_sticky=False, start_page=0, max_pages=1):
    torrents = []
    try:
        for page in range(start_page, start_page + max_pages):
            url = f"{target_url_base}&page={page}" if "?" in target_url_base else f"{target_url_base}?page={page}"
            resp = requests.get(url, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            
            if "登录" in resp.text and "torrents.php" not in resp.url: return None, "⚠️ Cookie 已失效！请更新。"
            if resp.status_code != 200: break

            is_global_free = bool(re.search(r'(?:全站|全局).*?(?:Free|免费)', resp.text, re.IGNORECASE))
            soup = BeautifulSoup(resp.text, 'html.parser')
            
            for row in soup.find_all('tr'):
                if skip_sticky and is_real_sticky(row): continue

                tds = row.find_all('td')
                if len(tds) < 6: continue
                title_a = row.find('a', href=lambda h: h and h.startswith('details.php?id='))
                if not title_a: continue
                
                tid = re.search(r'id=(\d+)', title_a['href']).group(1)
                title = title_a.get('title') or title_a.get_text(strip=True)
                if not title and title_a.find('b'): title = title_a.find('b').get_text(strip=True)
                if not title or tid in seen_ids: continue
                seen_ids.add(tid)
                link = f"{BASE_URL}/" + title_a['href']
                
                release_group = ""
                group_match = re.search(r'-([A-Za-z0-9@_]+)$', title.strip())
                if group_match: release_group = group_match.group(1).upper()
                
                try: 
                    size = tds[-5].get_text(strip=True)
                    seeders = tds[-4].get_text(strip=True)
                    leechers = tds[-3].get_text(strip=True)
                    time_col_td = tds[-6]
                except: 
                    size, seeders, leechers, time_col_td = "未知", "-", "-", None

                img_src = ""
                td = title_a.find_parent('td')
                tags = []
                
                if is_global_free: tags.append('Free')
                
                if time_col_td:
                    time_span = time_col_td.find('span', title=re.compile(r'\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}'))
                    t_text = time_span.get_text(separator='', strip=True) if time_span else time_col_td.get_text(separator='', strip=True)
                    if t_text and any(x in t_text for x in ['年', '月', '周', '天', '时', '分', '秒']):
                        clean_t = re.sub(r'\s+', '', t_text)
                        if not clean_t.endswith('前') and not re.search(r'\d{4}-\d{2}-\d{2}', clean_t):
                            clean_t += '前'
                        tags.append(f"🕒 {clean_t}")
                    elif re.search(r'\d{4}-\d{2}-\d{2}', t_text):
                        tags.append(f"🕒 {t_text.strip()}")
                
                subtitle = ""
                if td:
                    raw_html = str(td)
                    
                    if not any('🕒' in t for t in tags):
                        t_match = re.search(r'\b(20\d{2}-\d{2}-\d{2}\s+\d{2}:\d{2}(?::\d{2})?)\b', raw_html)
                        if t_match: tags.append(f"🕒 {t_match.group(1)}")

                    time_matches = re.finditer(r'(?:免费剩余时间|免费剩余|剩余时间|剩余)[:：\s]*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', raw_html)
                    for tm in time_matches:
                        t_str = f"剩余时间: {tm.group(1).strip()}"
                        if t_str not in tags: tags.append(t_str)

                    for icon in td.find_all('img'):
                        alt = str(icon.get('alt') or '')
                        title_attr = str(icon.get('title') or '')
                        combined_text = alt + " " + title_attr
                        if '免费' in combined_text or 'Free' in combined_text:
                            if 'Free' not in tags: tags.append('Free')
                        if icon.get('src', '') != img_src: 
                            icon.replace_with(soup.new_tag("span", string=f" [{alt}] "))
                    
                    for span in td.find_all('span'):
                        text = span.get_text(strip=True)
                        title_attr = str(span.get('title') or '')
                        combined = (text + " " + title_attr).upper()
                        if 'FREE' in combined or '免费' in combined:
                            if 'Free' not in tags: tags.append('Free')
                        for disc in ['50%', '30%', '2X']:
                            if disc in combined and disc not in tags: tags.append(disc)
                                
                        text_clean = text.replace('[', '').replace(']', '').strip()
                        if not text_clean: continue
                        
                        span_class = ' '.join(span.get('class', [])).lower()
                        is_valid_class = bool(re.search(r'tag|label|badge', span_class))
                        is_known_kw = bool(text_clean.upper() in [k.upper() for k in TECH_KEYWORDS])
                        if (is_valid_class or is_known_kw) and text_clean.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', text_clean):
                            if text_clean not in tags: tags.append(text_clean)
                            span.replace_with(soup.new_tag("span", string=f" [{text_clean}] "))
                    
                    subtitle = td.get_text(separator=' | ', strip=True).replace(title, '').strip(' | ')
                    subtitle = re.sub(r'(?i)\[?(?:Sticky|置顶|poster)\]?[\s\|]*', '', subtitle)
                    
                    inline_time = re.search(r'\[?(?:剩余时间)[:：\s]*\]?[\s\|]*(\d+\s*[年月周天小时分秒钟]+(?:\s*\d+\s*[年月周天小时分秒钟]+)*)', subtitle)
                    if inline_time:
                        t_str = f"剩余时间: {inline_time.group(1).strip()}"
                        if t_str not in tags: tags.append(t_str)
                        subtitle = subtitle.replace(inline_time.group(0), '')
                    
                    clean_sub = subtitle
                    for m in re.finditer(r'\[(.*?)\]', subtitle):
                        tag = m.group(1).strip(' |[]')
                        if tag.upper() in [k.upper() for k in TECH_KEYWORDS] or re.match(r'^[\d]+%$', tag) or tag.upper() in ['热门']:
                            if tag not in tags: tags.append(tag)
                            clean_sub = clean_sub.replace(m.group(0), '')
                        else:
                            clean_sub = clean_sub.replace(m.group(0), tag)
                    
                    clean_sub = re.sub(r'热门|免费', '', clean_sub)
                    parts = [p.strip() for p in clean_sub.split('|') if p.strip()]
                    kept_sub = []
                    for p in parts:
                        clean_p = re.sub(r'^[()\[\]\-|\s]+|[()\[\]\-|\s]+$', '', p)
                        if not clean_p: continue
                        if clean_p.upper() in [k.upper() for k in TECH_KEYWORDS]:
                            if clean_p not in tags: tags.append(clean_p)
                        else:
                            kept_sub.append(p)
                    subtitle = ' | '.join(kept_sub)
                
                tags = auto_extract_tags(title + " " + subtitle, tags)
                tags = purify_tags(tags)

                torrents.append({
                    'id': tid, 'title': title, 'link': link, 'subtitle': subtitle, 'img': img_src,
                    'seeders': seeders, 'leechers': leechers, 'size': size, 'tags': tags, 'group': release_group
                })
                
                if len(torrents) >= 50 and max_pages > 1: break
            if len(torrents) >= 50 and max_pages > 1: break
            
        return torrents, None
    except Exception as e:
        return None, f"网络请求异常: {str(e)}"

def background_monitor():
    last_sign_date = None
    while True:
        try:
            current_date = datetime.datetime.now().strftime("%Y-%m-%d")
            if last_sign_date != current_date:
                r = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
                r.encoding = 'utf-8'
                res_text = r.text
                if any(kw in res_text for kw in ["已经签到", "请明天再来", "签到成功", "魔力值", "电力值", "次签到", "退下", "已经"]):
                    last_sign_date = current_date
        except Exception: pass

        try:
            current_off, err = parse_torrents_page(OFFICIAL_URL, set(), skip_sticky=False, max_pages=2)
            if current_off:
                off_dict = load_cache('official')
                for item in current_off[:5]:
                    tid_str = str(item['id'])
                    if tid_str not in off_dict or not off_dict[tid_str].get('img') or 'data:image/svg+xml' in off_dict[tid_str]['img'] or off_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags, det_group = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_group and not item.get('group'): item['group'] = det_group
                        if det_tags:
                            item['tags'] = purify_tags(merge_smart_tags_from_details(det_tags, item.get('tags', [])))

                    if tid_str in off_dict:
                        cached = off_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if not item.get('group') and cached.get('group'): item['group'] = cached['group']
                        item['tags'] = purify_tags(merge_smart_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                        
                    off_dict[tid_str] = item
                save_cache(off_dict, 'official')
        except Exception: pass

        try:
            current_rec, err = parse_torrents_page(COMPREHENSIVE_URL, set(), skip_sticky=True, max_pages=4)
            if current_rec:
                rec_dict = load_cache('recent')
                for item in current_rec[:5]:
                    tid_str = str(item['id'])
                    if tid_str not in rec_dict or not rec_dict[tid_str].get('img') or 'data:image/svg+xml' in rec_dict[tid_str]['img'] or rec_dict[tid_str].get('size') in ['未知', '加载中...']:
                        det_img, det_size, det_tags, det_group = fetch_details_info(item['id'])
                        if det_img: item['img'] = det_img
                        if det_size and det_size != "未知": item['size'] = det_size
                        if det_group and not item.get('group'): item['group'] = det_group
                        if det_tags:
                            item['tags'] = purify_tags(merge_smart_tags_from_details(det_tags, item.get('tags', [])))

                    if tid_str in rec_dict:
                        cached = rec_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if not item.get('group') and cached.get('group'): item['group'] = cached['group']
                        item['tags'] = purify_tags(merge_smart_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                        
                    rec_dict[tid_str] = item
                save_cache(rec_dict, 'recent')
        except Exception: pass

        time.sleep(900)

threading.Thread(target=background_monitor, daemon=True).start()

HTML_TEMPLATE = """
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
    <title>织梦PT 手机端</title>
    <style>
        :root { --bg-color: #f7f7f7; --card-bg: #ffffff; --text-main: #333333; --text-sub: #888888; --border-color: #eeeeee; --topbar-bg: #ffffff; --icon-bg: #fef3c7; --primary-color: #f59e0b; }
        [data-theme="dark"] { --bg-color: #121212; --card-bg: #1e1e1e; --text-main: #eeeeee; --text-sub: #aaaaaa; --border-color: #333333; --topbar-bg: #1e1e1e; --icon-bg: #452e05; --primary-color: #fbbf24; }
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif; background-color: var(--bg-color); color: var(--text-main); margin: 0; padding: 0; padding-top: 55px; transition: background-color 0.3s; }
        
        .top-bar { position: fixed; top: 0; left: 0; right: 0; height: 50px; background: var(--topbar-bg); display: flex; align-items: center; justify-content: space-between; padding: 0 15px; box-shadow: 0 1px 3px rgba(0,0,0,0.05); z-index: 100; transition: background 0.3s; }
        .menu-btn { width: 22px; height: 16px; position: relative; cursor: pointer; display: flex; flex-direction: column; justify-content: space-between; padding: 0; }
        .menu-btn span { display: block; height: 2px; width: 100%; background: var(--text-main); border-radius: 2px; transition: all 0.3s ease; }
        .menu-btn.open span:nth-child(1) { transform: translateY(7px) rotate(45deg); }
        .menu-btn.open span:nth-child(2) { opacity: 0; }
        .menu-btn.open span:nth-child(3) { transform: translateY(-7px) rotate(-45deg); }

        .logo { font-weight: 800; font-size: 20px; color: var(--primary-color); letter-spacing: 1px; display: flex; align-items: center; gap: 4px;}
        .logo-icon { font-size: 22px; }
        .top-placeholder { width: 22px; }

        .sidebar { position: fixed; top: 0; left: -260px; bottom: 0; width: 260px; background: var(--card-bg); z-index: 1001; transition: left 0.3s cubic-bezier(0.4, 0, 0.2, 1); box-shadow: 2px 0 10px rgba(0,0,0,0.1); padding-top: 60px; display: flex; flex-direction: column; }
        .sidebar.open { left: 0; }
        .menu-item { padding: 16px 24px; font-size: 16px; font-weight: 500; color: var(--text-main); text-decoration: none; display: block; border-bottom: 1px solid var(--border-color); }
        .menu-item.active { color: var(--primary-color); border-left: 4px solid var(--primary-color); padding-left: 20px; background: rgba(245,158,11,0.1); }
        .theme-toggle-icon { margin: auto auto 30px 24px; width: 40px; height: 40px; border-radius: 20px; background: var(--icon-bg); display: flex; align-items: center; justify-content: center; font-size: 20px; cursor: pointer; user-select: none; transition: transform 0.2s; }
        .theme-toggle-icon:active { transform: scale(0.9); }
        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.5); z-index: 1000; opacity: 0; pointer-events: none; transition: opacity 0.3s; }
        .overlay.open { opacity: 1; pointer-events: auto; }

        .container { padding: 5px 10px 10px 10px; }
        
        .global-search-container { display: flex; gap: 8px; margin: 5px 5px 15px 5px; align-items: center; background: var(--card-bg); padding: 5px; border-radius: 20px; border: 1px solid var(--border-color); box-shadow: 0 2px 8px rgba(0,0,0,0.02); }
        .search-input { flex: 1; padding: 8px 15px; border: none; outline: none; font-size: 14px; background: transparent; color: var(--text-main); }
        .search-btn { background: rgba(245,158,11,0.1); color: var(--primary-color); border: none; width: 36px; height: 36px; border-radius: 18px; font-weight: bold; font-size: 16px; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s ease; flex-shrink: 0; }
        .search-btn:active { background: var(--primary-color); color: #fff; }
        [data-theme="dark"] .search-btn { background: rgba(251,191,36,0.15); }
        
        .list-item { display: flex; gap: 12px; padding: 15px 5px; border-bottom: 1px solid var(--border-color); }
        .poster-wrapper { position: relative; flex-shrink: 0; width: 100px; height: 140px; }
        .poster { width: 100%; height: 100%; object-fit: cover; border-radius: 8px; background-color: #e0e0e0; transition: opacity 0.5s ease-in-out; }
        .tag-free { position: absolute; top: 0; left: 0; background: #22c55e; color: #fff; font-size: 10px; font-weight: bold; padding: 2px 6px; border-radius: 8px 0 8px 0; z-index: 10; }
        
        .info { flex-grow: 1; display: flex; flex-direction: column; overflow: hidden; justify-content: space-between; padding-bottom: 2px; }
        .title { font-size: 15px; font-weight: bold; margin-bottom: 4px; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; overflow: hidden; line-height: 1.3; }
        .subtitle { font-size: 12px; color: var(--text-sub); line-height: 1.4; display: -webkit-box; -webkit-line-clamp: 1; -webkit-box-orient: vertical; overflow: hidden; margin-bottom: 4px; }
        .subtitle:empty { display: none; margin: 0; }
        
        .stats-and-download { display: flex; justify-content: space-between; align-items: center; gap: 10px; width: 100%; margin-bottom: 4px; }
        .stats-row { font-size: 12px; color: var(--text-sub); display: flex; align-items: center; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; flex: 1; -webkit-overflow-scrolling: touch; }
        .stats-row::-webkit-scrollbar { display: none; }
        .stat-item { display: flex; align-items: center; gap: 2px; }
        .seed-icon { color: #22c55e; font-size: 10px; }
        .leech-icon { color: #ef4444; font-size: 10px; }
        
        .tags-wrapper { display: flex; flex-direction: column; gap: 6px; margin-top: auto; width: 100%; overflow: hidden; padding-top: 2px; }
        .tags-top { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; }
        .tags-top::-webkit-scrollbar { display: none; }
        .tags-top:empty { display: none; }
        
        .tags-bottom { display: flex; gap: 6px; overflow-x: auto; white-space: nowrap; width: 100%; scrollbar-width: none; align-items: center; -webkit-overflow-scrolling: touch; padding-bottom: 2px; }
        .tags-bottom::-webkit-scrollbar { display: none; }
        .tags-bottom:empty { display: none; }
        
        .secondary-tag { display: none !important; }
        .show-extra .secondary-tag { display: inline-flex !important; }
        
        .badge-toggle { background: #e5e7eb; color: #6b7280; font-size: 10px; padding: 2px 8px; border-radius: 4px; flex-shrink: 0; cursor: pointer; display: inline-flex; align-items: center; letter-spacing: 1px; font-weight: bold; transition: all 0.2s ease;}
        [data-theme="dark"] .badge-toggle { background: #374151; color: #9ca3af; }
        .badge-toggle:active { opacity: 0.7; }
        
        .badge { background: #8b5cf6; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-discount { background: #ef4444; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-tech { background: #0ea5e9; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-lang { background: #10b981; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; }
        .badge-group { background: #ec4899; color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; letter-spacing: 0.5px;} 
        
        .badge-zmpt { background: linear-gradient(135deg, #f59e0b 0%, #fbbf24 100%); color: #fff; font-size: 10px; padding: 2px 6px; border-radius: 4px; flex-shrink: 0; display: inline-flex; align-items: center; font-weight: bold; letter-spacing: 0.5px; box-shadow: 0 1px 3px rgba(245,158,11,0.3);}

        .download-btn { background: var(--primary-color); color: #fff; font-size: 10px; padding: 2px 8px; border-radius: 4px; font-weight: bold; text-decoration: none; flex-shrink: 0; display: inline-flex; align-items: center; letter-spacing: 1px; min-width: 36px; justify-content: center;}
        .download-btn:active { opacity: 0.8; }
        
        .account-box { background: var(--card-bg); border-radius: 12px; padding: 20px; text-align: center; border: 1px solid var(--border-color); }
        .avatar { width: 70px; height: 70px; background: var(--primary-color); color: #fff; border-radius: 35px; line-height: 70px; font-size: 24px; margin: 0 auto 10px; }
        .account-name { font-size: 18px; font-weight: bold; }
        .stat-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }
        .stat-val { font-size: 18px; font-weight: bold; color: var(--primary-color); margin-bottom: 4px; }
        .stat-lbl { font-size: 12px; color: var(--text-sub); }
        .btn-signin { background: #22c55e; color: #fff; width: 100%; padding: 12px; font-size: 16px; border-radius: 8px; font-weight: bold; margin-top: 25px; border: none; cursor: pointer; }
    </style>
</head>
<body>
    <script>
        function setTheme(theme) {
            const html = document.documentElement;
            if (theme === 'dark') { html.setAttribute('data-theme', 'dark'); }
            else if (theme === 'light') { html.removeAttribute('data-theme'); }
            else { 
                if (window.matchMedia('(prefers-color-scheme: dark)').matches) html.setAttribute('data-theme', 'dark');
                else html.removeAttribute('data-theme');
            }
        }
        let currentTheme = localStorage.getItem('zmpt_theme') || 'auto';
        setTheme(currentTheme);
        window.matchMedia('(prefers-color-scheme: dark)').addEventListener('change', () => {
            if (localStorage.getItem('zmpt_theme') === 'auto') setTheme('auto');
        });
        
        window.toggleTags = function(btn) {
            const wrapper = btn.closest('.tags-wrapper');
            wrapper.classList.toggle('show-extra');
            if (wrapper.classList.contains('show-extra')) {
                btn.innerText = '收起 ▴';
            } else {
                btn.innerText = '... ▾';
            }
        };
    </script>

    <div class="top-bar">
        <div class="menu-btn" id="menuBtn" onclick="toggleMenu()">
            <span></span><span></span><span></span>
        </div>
        <div class="logo"><span class="logo-icon">🌟</span> 织梦PT</div>
        <div class="top-placeholder"></div>
    </div>

    <div class="overlay" id="overlay" onclick="toggleMenu()"></div>
    <div class="sidebar" id="sidebar">
        <a href="/?tab=recent" class="menu-item {% if tab == 'recent' %}active{% endif %}">最近发布</a>
        <a href="/?tab=official" class="menu-item {% if tab == 'official' %}active{% endif %}">官种发布</a>
        <a href="/?tab=account" class="menu-item {% if tab == 'account' %}active{% endif %}">我的帐号</a>
        <div class="theme-toggle-icon" id="themeIcon" onclick="cycleTheme()">🔄</div>
    </div>

    <div class="container" id="list-container">
        {% if tab in ['official', 'recent', 'search'] %}
        <form class="global-search-container" action="/" method="GET">
            <input type="hidden" name="tab" value="search">
            <input type="text" name="q" class="search-input" placeholder="输入关键词搜全站..." value="{{ query }}">
            <button type="submit" class="search-btn">🔍</button>
        </form>
        {% endif %}

        {% if error %}<div style="text-align:center; color:#ef4444; margin-top:30px; font-weight:bold;">{{ error }}</div>{% endif %}

        {% if tab == 'account' and user_info %}
        <div class="account-box">
            <div class="avatar">{{ user_info.username[:1] | upper }}</div>
            <div class="account-name">{{ user_info.username }}</div>
            <div class="stat-grid">
                <div><div class="stat-val">{{ user_info.bonus }}</div><div class="stat-lbl">电力值</div></div>
                <div><div class="stat-val">{{ user_info.ratio }}</div><div class="stat-lbl">分享率</div></div>
                <div><div class="stat-val">{{ user_info.upload }}</div><div class="stat-lbl">总上传</div></div>
                <div><div class="stat-val">{{ user_info.download }}</div><div class="stat-lbl">总下载</div></div>
            </div>
            <button id="signinBtn" class="btn-signin" onclick="doSignIn()">✅ 手动签到</button>
        </div>
        <script>
            function doSignIn() {
                const btn = document.getElementById('signinBtn');
                btn.innerText = '签到中...'; btn.disabled = true;
                fetch('/api/signin').then(r => r.json()).then(data => {
                    alert(data.msg); btn.innerText = '✅ 手动签到'; btn.disabled = false;
                }).catch(e => { alert('请求失败'); btn.innerText = '✅ 手动签到'; btn.disabled = false; });
            }
        </script>
        
        {% elif tab != 'account' %}
            {% for item in torrents %}
            <div class="list-item" id="item-{{ item.id }}">
                <div class="poster-wrapper">
                    {% set needs_lazy = not item.img or ('data:image/svg+xml' in item.img) %}
                    <img class="poster {% if needs_lazy %}lazy-load{% endif %}" 
                         src="{{ item.img if item.img else 'data:image/svg+xml;utf8,<svg xmlns=\\'http://www.w3.org/2000/svg\\' width=\\'100\\' height=\\'140\\'><rect width=\\'100%\\' height=\\'100%\\' fill=\\'%23e6e6e6\\'/><text x=\\'50%\\' y=\\'50%\\' fill=\\'%23999\\' font-size=\\'12\\' font-family=\\'sans-serif\\' text-anchor=\\'middle\\' dominant-baseline=\\'middle\\'>加载海报</text></svg>' }}" 
                         {% if needs_lazy %}data-tid="{{ item.id }}"{% endif %}
                         referrerpolicy="no-referrer" alt="海报" onclick="window.open('{{ item.link }}', '_blank')">
                    {% if 'FREE' in item.tags|join(' ')|upper %}
                        <div class="tag-free">FREE</div>
                    {% endif %}
                </div>
                <div class="info">
                    <div class="title" onclick="window.open('{{ item.link }}', '_blank')">{{ item.title }}</div>
                    <div class="subtitle">{{ item.subtitle }}</div>
                    
                    <div class="stats-and-download">
                        <div class="stats-row">
                            <span class="stat-item" id="size-{{ item.id }}">📄 {{ item.size }}</span>
                            <span class="stat-item"><span class="seed-icon">▲</span> {{ item.seeders }}</span>
                            <span class="stat-item"><span class="leech-icon">▼</span> {{ item.leechers }}</span>
                            {% if item.group %}
                                {% if item.group|upper == 'ZMPT' %}
                                    <span class="badge-zmpt">💎 ZmPT</span>
                                {% else %}
                                    <span class="badge-group">{{ item.group }}</span>
                                {% endif %}
                            {% endif %}
                        </div>
                        <a href="/download/{{ item.id }}" class="download-btn">下载</a>
                    </div>

                    <div class="tags-wrapper">
                        <div class="tags-top">
                            {% for tag in item.top_pri %}
                                <span class="badge-tech">{{ tag }}</span>
                            {% endfor %}
                            {% for tag in item.top_sec %}
                                <span class="badge-tech secondary-tag">{{ tag }}</span>
                            {% endfor %}
                        </div>

                        <div class="tags-bottom">
                            {% if item.time_tag %}
                                <span class="badge-discount">{{ item.time_tag }}</span>
                            {% endif %}
                            
                            {% if item.clock_tag %}
                                <span class="badge" style="background:#64748b;">{{ item.clock_tag }}</span>
                            {% endif %}
                            
                            {% for tag in item.bot_disc %}
                                <span class="badge-discount">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_pri %}
                                <span class="badge">{{ tag }}</span>
                            {% endfor %}
                            
                            {% for tag in item.bot_sec %}
                                <span class="badge-lang secondary-tag">{{ tag }}</span>
                            {% endfor %}
                            
                            {% if item.top_sec|length > 0 or item.bot_sec|length > 0 %}
                                <span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>
                            {% endif %}
                        </div>
                    </div>
                </div>
            </div>
            {% endfor %}
            
            {% if tab == 'search' and not torrents and query %}
                <div style="text-align:center; color:var(--text-sub); margin-top:30px;">找到 0 个结果</div>
            {% elif tab == 'search' and torrents %}
                <div style="text-align:center; color:var(--text-sub); margin: 20px 0; font-size:12px;">已展示当前页全部结果</div>
            {% elif tab in ['recent', 'official'] and not error %}
                <!-- 💡添加了触发无限滚动的观察点 -->
                <div id="load-more-sentinel" style="text-align:center; padding: 20px; color:var(--text-sub); font-size:12px; clear:both;">
                    向下划动，无痕加载更多... ⚡
                </div>
            {% endif %}
        {% endif %}
    </div>

    <script>
        const sidebar = document.getElementById('sidebar');
        const overlay = document.getElementById('overlay');
        const menuBtn = document.getElementById('menuBtn');
        const themeIcon = document.getElementById('themeIcon');

        function toggleMenu() { 
            sidebar.classList.toggle('open'); 
            overlay.classList.toggle('open'); 
            menuBtn.classList.toggle('open');
        }

        const icons = { 'auto': '🔄', 'dark': '🌙', 'light': '☀️' };
        themeIcon.innerText = icons[currentTheme];

        function cycleTheme() {
            if (currentTheme === 'auto') currentTheme = 'dark';
            else if (currentTheme === 'dark') currentTheme = 'light';
            else currentTheme = 'auto';
            localStorage.setItem('zmpt_theme', currentTheme);
            setTheme(currentTheme);
            themeIcon.innerText = icons[currentTheme];
        }

        // 暴露给全局，方便追加新渲染的数据使用
        window.lazyObserver = null;

        document.addEventListener("DOMContentLoaded", function() {
            if ('IntersectionObserver' in window) {
                window.lazyObserver = new IntersectionObserver((entries, obs) => {
                    entries.forEach(entry => {
                        if (entry.isIntersecting) {
                            const img = entry.target;
                            const tid = img.getAttribute('data-tid');
                            if (tid) {
                                fetch('/api/extra/' + tid)
                                    .then(r => r.json())
                                    .then(data => {
                                        if (data.img) {
                                            img.src = data.img;
                                            img.classList.remove('lazy-load');
                                        }
                                        if (data.size && data.size !== '未知' && data.size !== '加载中...') {
                                            const sizeSpan = document.getElementById('size-' + tid);
                                            if (sizeSpan) sizeSpan.innerHTML = '📄 ' + data.size;
                                        }
                                        
                                        const itemDiv = document.getElementById('item-' + tid);
                                        if (itemDiv && data.tags) {
                                            if (data.group && !itemDiv.querySelector('.badge-group') && !itemDiv.querySelector('.badge-zmpt')) {
                                                const statsRow = itemDiv.querySelector('.stats-row');
                                                if (statsRow) {
                                                    if (data.group.toUpperCase() === 'ZMPT') {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-zmpt">💎 ZmPT</span>`);
                                                    } else {
                                                        statsRow.insertAdjacentHTML('beforeend', `<span class="badge-group">${data.group}</span>`);
                                                    }
                                                }
                                            }
                                            
                                            const tagsWrapper = itemDiv.querySelector('.tags-wrapper');
                                            if (tagsWrapper && data.top_pri) {
                                                let hasFree = !!itemDiv.querySelector('.tag-free');
                                                if (data.tags.some(t => t.toUpperCase() === 'FREE' || t === '免费') && !hasFree) {
                                                    itemDiv.querySelector('.poster-wrapper').insertAdjacentHTML('beforeend', '<div class="tag-free">FREE</div>');
                                                }
                                                
                                                let html = '<div class="tags-top">';
                                                data.top_pri.forEach(t => html += `<span class="badge-tech">${t}</span>`);
                                                data.top_sec.forEach(t => html += `<span class="badge-tech secondary-tag">${t}</span>`);
                                                html += '</div><div class="tags-bottom">';
                                                
                                                if (data.time_tag) html += `<span class="badge-discount">${data.time_tag}</span>`;
                                                if (data.clock_tag) html += `<span class="badge" style="background:#64748b;">${data.clock_tag}</span>`;
                                                
                                                data.bot_disc.forEach(t => { 
                                                    if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge-discount">${t}</span>`; 
                                                });
                                                data.bot_pri.forEach(t => { 
                                                    if(t.toUpperCase() !== 'FREE' && t !== '免费') html += `<span class="badge">${t}</span>`; 
                                                });
                                                data.bot_sec.forEach(t => html += `<span class="badge-lang secondary-tag">${t}</span>`);
                                                
                                                if (data.top_sec.length > 0 || data.bot_sec.length > 0) {
                                                    html += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                                }
                                                html += '</div>';
                                                tagsWrapper.innerHTML = html;
                                            }
                                        }
                                    })
                                    .catch(e => console.error('懒加载失败:', e));
                                    
                                img.removeAttribute('data-tid');
                                obs.unobserve(img);
                            }
                        }
                    });
                }, { rootMargin: "300px" });

                document.querySelectorAll('.lazy-load').forEach(img => {
                    window.lazyObserver.observe(img);
                });

                // 💡无痕无限滑动监听器逻辑
                const sentinel = document.getElementById('load-more-sentinel');
                if (sentinel) {
                    let currentPage = 1; 
                    let isLoadingMore = false;
                    let hasMore = true;
                    // 初始化去重缓存（保证加载时屏幕上绝不出现重复种子）
                    const seenIds = new Set();
                    document.querySelectorAll('.list-item').forEach(el => seenIds.add(el.id.replace('item-', '')));

                    const loadMoreObserver = new IntersectionObserver((entries) => {
                        const entry = entries[0];
                        if (entry.isIntersecting && !isLoadingMore && hasMore) {
                            loadMore();
                        }
                    }, { rootMargin: "200px" });
                    
                    loadMoreObserver.observe(sentinel);

                    function loadMore() {
                        isLoadingMore = true;
                        sentinel.innerText = "正在无痕加载下一页...";
                        
                        const params = new URLSearchParams(window.location.search);
                        const tab = params.get('tab') || 'recent';
                        const q = params.get('q') || '';
                        
                        fetch(`/api/load_more?tab=${tab}&q=${encodeURIComponent(q)}&page=${currentPage}`)
                            .then(r => r.json())
                            .then(data => {
                                currentPage++;
                                isLoadingMore = false;
                                
                                if (data.error) {
                                    sentinel.innerText = "加载失败: " + data.error + " (点击重试)";
                                    return;
                                }
                                
                                if (data.torrents.length === 0) {
                                    // 遇到满页置顶情况，自动翻下一页
                                    if (currentPage < 10) {
                                        loadMore();
                                    } else {
                                        hasMore = false;
                                        sentinel.innerText = "没有更多数据了";
                                    }
                                    return;
                                }
                                
                                let addedCount = 0;
                                data.torrents.forEach(item => {
                                    if (seenIds.has(item.id)) return; // 去重过滤
                                    seenIds.add(item.id);
                                    addedCount++;
                                    
                                    // 前端构建 DOM 树
                                    let tagsTopHtml = '';
                                    item.top_pri.forEach(t => tagsTopHtml += `<span class="badge-tech">${t}</span>`);
                                    item.top_sec.forEach(t => tagsTopHtml += `<span class="badge-tech secondary-tag">${t}</span>`);
                                    
                                    let tagsBotHtml = '';
                                    if (item.time_tag) tagsBotHtml += `<span class="badge-discount">${item.time_tag}</span>`;
                                    if (item.clock_tag) tagsBotHtml += `<span class="badge" style="background:#64748b;">${item.clock_tag}</span>`;
                                    item.bot_disc.forEach(t => tagsBotHtml += `<span class="badge-discount">${t}</span>`);
                                    item.bot_pri.forEach(t => tagsBotHtml += `<span class="badge">${t}</span>`);
                                    item.bot_sec.forEach(t => tagsBotHtml += `<span class="badge-lang secondary-tag">${t}</span>`);
                                    
                                    if (item.top_sec.length > 0 || item.bot_sec.length > 0) {
                                        tagsBotHtml += `<span class="badge-toggle" onclick="toggleTags(this)">... ▾</span>`;
                                    }

                                    let groupHtml = '';
                                    if (item.group) {
                                        if (item.group.toUpperCase() === 'ZMPT') {
                                            groupHtml = `<span class="badge-zmpt">💎 ZmPT</span>`;
                                        } else {
                                            groupHtml = `<span class="badge-group">${item.group}</span>`;
                                        }
                                    }
                                    
                                    let freeTagHtml = '';
                                    if (item.tags.join(' ').toUpperCase().includes('FREE')) {
                                        freeTagHtml = `<div class="tag-free">FREE</div>`;
                                    }

                                    const imgPlaceholder = "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' width='100' height='140'><rect width='100%' height='100%' fill='%23e6e6e6'/><text x='50%' y='50%' fill='%23999' font-size='12' font-family='sans-serif' text-anchor='middle' dominant-baseline='middle'>加载海报</text></svg>";
                                    const imgSrc = item.img ? item.img : imgPlaceholder;
                                    const needsLazy = !item.img || item.img.includes('data:image/svg+xml');

                                    const html = `
                                    <div class="list-item" id="item-${item.id}">
                                        <div class="poster-wrapper">
                                            <img class="poster ${needsLazy ? 'lazy-load' : ''}" 
                                                 src="${imgSrc}" 
                                                 ${needsLazy ? `data-tid="${item.id}"` : ''}
                                                 referrerpolicy="no-referrer" alt="海报" onclick="window.open('${item.link}', '_blank')">
                                            ${freeTagHtml}
                                        </div>
                                        <div class="info">
                                            <div class="title" onclick="window.open('${item.link}', '_blank')">${item.title}</div>
                                            <div class="subtitle">${item.subtitle || ''}</div>
                                            <div class="stats-and-download">
                                                <div class="stats-row">
                                                    <span class="stat-item" id="size-${item.id}">📄 ${item.size}</span>
                                                    <span class="stat-item"><span class="seed-icon">▲</span> ${item.seeders}</span>
                                                    <span class="stat-item"><span class="leech-icon">▼</span> ${item.leechers}</span>
                                                    ${groupHtml}
                                                </div>
                                                <a href="/download/${item.id}" class="download-btn">下载</a>
                                            </div>
                                            <div class="tags-wrapper">
                                                <div class="tags-top">${tagsTopHtml}</div>
                                                <div class="tags-bottom">${tagsBotHtml}</div>
                                            </div>
                                        </div>
                                    </div>
                                    `;
                                    sentinel.insertAdjacentHTML('beforebegin', html);
                                });
                                
                                document.querySelectorAll('.lazy-load:not([data-observed])').forEach(img => {
                                    img.setAttribute('data-observed', '1');
                                    window.lazyObserver.observe(img);
                                });
                                
                                if (addedCount === 0) {
                                    loadMore(); // 如果抓到的全被前端去重了，自动加载下一批
                                } else {
                                    sentinel.innerText = "向下划动，无痕加载更多... ⚡";
                                }
                            })
                            .catch(e => {
                                isLoadingMore = false;
                                sentinel.innerText = "网络错误，点击重试";
                                sentinel.onclick = loadMore;
                            });
                    }
                }
            }
        });
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    tab = request.args.get('tab', 'recent')
    query = request.args.get('q', '')
    torrents = []
    user_info = {}
    seen_ids = set() 
    error_msg = None
    
    try:
        if tab == 'account':
            resp = requests.get(INDEX_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
            if "登录" in resp.text and "index.php" not in resp.url:
                error_msg = "⚠️ Cookie 已失效，请在代码中更新 Cookie！"
            else:
                soup = BeautifulSoup(resp.text, 'html.parser')
                for tag in soup.find_all(True):
                    title_attr = tag.get('title') or tag.get('alt') or tag.get('data-original-title')
                    if title_attr and isinstance(title_attr, str) and len(title_attr) < 20:
                        tag.append(f"{title_attr}")
                page_text = soup.get_text(separator='', strip=True)
                
                user_match = re.search(r'([A-Za-z0-9_]+)\s*\[.*?签到|欢迎回来[\s,，]*([A-Za-z0-9_]+)', page_text)
                username = user_match.group(1) or user_match.group(2) if user_match else "织梦会员"
                
                bonus_match = re.search(r'(?:魔力值|电力值)[:：\s]*([\d\.,]+)', page_text)
                ratio_match = re.search(r'分享率[:：\s]*([无限Inf]+|[\d\.,]+)', page_text, re.IGNORECASE)
                up_match = re.search(r'(?:上传量?|上传|上行)[:：\s]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                down_match = re.search(r'(?:下载量?|下载|下行)[:：\s]*([\d\.,]+\s*[KMGTP]B)', page_text, re.IGNORECASE)
                
                user_info = {
                    'username': username,
                    'bonus': bonus_match.group(1) if bonus_match else "0",
                    'ratio': ratio_match.group(1) if ratio_match else "0",
                    'upload': up_match.group(1) if up_match else "0 GB",
                    'download': down_match.group(1) if down_match else "0 GB"
                }

        elif tab in ['official', 'search']:
            if tab == 'search' and not query: pass
            else:
                if tab == 'search':
                    safe_query = urllib.parse.quote(query)
                    target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
                else: 
                    target_url = OFFICIAL_URL
                    
                fetched_torrents, err = parse_torrents_page(target_url, seen_ids, is_search=(tab == 'search'), skip_sticky=False, start_page=0, max_pages=2)
                if fetched_torrents is not None:
                    history_dict = load_cache('official')
                    for item in fetched_torrents:
                        tid_str = str(item['id'])
                        if tid_str in history_dict:
                            cached = history_dict[tid_str]
                            if 'data:image' not in cached.get('img','') and 'data:image' in item.get('img',''):
                                item['img'] = cached['img']
                            if cached.get('size') not in ['未知', '加载中...'] and item.get('size') in ['未知', '-']:
                                item['size'] = cached['size']
                            if not item.get('group') and cached.get('group'):
                                item['group'] = cached['group']
                            item['tags'] = purify_tags(merge_smart_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                        
                        history_dict[tid_str] = item
                    if tab != 'search':
                        save_cache(history_dict, 'official')
                    torrents = fetched_torrents
                else:
                    if err: error_msg = err

        elif tab == 'recent':
            fetched_torrents, err_msg = parse_torrents_page(COMPREHENSIVE_URL, seen_ids, is_search=False, skip_sticky=True, start_page=0, max_pages=4)
            if fetched_torrents is not None:
                history_dict = load_cache('recent')
                for item in fetched_torrents:
                    tid_str = str(item['id'])
                    if tid_str in history_dict:
                        cached = history_dict[tid_str]
                        if cached.get('img') and 'data:image/svg+xml' not in cached['img']: item['img'] = cached['img']
                        if cached.get('size') and cached.get('size') not in ['未知', '加载中...']: item['size'] = cached['size']
                        if cached.get('group') and not item.get('group'): item['group'] = cached['group']
                        item['tags'] = purify_tags(merge_smart_tags_from_list(item.get('tags', []), cached.get('tags', [])))
                    history_dict[tid_str] = item
                    
                save_cache(history_dict, 'recent')
                merged = list(history_dict.values())
                merged.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = merged[:50]
            else:
                history_data = list(load_cache('recent').values())
                history_data.sort(key=lambda x: int(x['id']), reverse=True)
                torrents = history_data[:50]
                if err_msg: error_msg = err_msg

        if torrents:
            for item in torrents:
                tpri, tsec, bdisc, bpri, bsec = sort_tags(item.get('tags', []))
                item['top_pri'] = tpri
                item['top_sec'] = tsec
                item['bot_disc'] = bdisc
                item['bot_pri'] = bpri
                item['bot_sec'] = bsec
                item['time_tag'] = ""
                item['clock_tag'] = ""
                
                for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                    for t in list(arr):
                        if '剩余' in t.upper():
                            item['time_tag'] = t
                            arr.remove(t)
                        elif '🕒' in t:
                            item['clock_tag'] = t
                            arr.remove(t)

    except Exception as e:
        error_msg = f"服务器内部请求错误: {e}"
        torrents = torrents or []
        
    return render_template_string(HTML_TEMPLATE, tab=tab, query=query, torrents=torrents or [], user_info=user_info, error=error_msg)

# 💡无痕下拉加载核心接口：绝对不保存数据到 json 缓存文件中！
@app.route('/api/load_more')
def api_load_more():
    tab = request.args.get('tab', 'recent')
    query = request.args.get('q', '')
    try:
        page = int(request.args.get('page', 1))
    except:
        page = 1
        
    seen_ids = set()
    
    if tab == 'search':
        safe_query = urllib.parse.quote(query)
        target_url = f"{BASE_URL}/torrents.php?incldead=1&spstate=0&inclbookmarked=0&search={safe_query}&search_area=0&search_mode=0"
        fetched, err = parse_torrents_page(target_url, seen_ids, skip_sticky=False, start_page=page, max_pages=1)
    elif tab == 'official':
        fetched, err = parse_torrents_page(OFFICIAL_URL, seen_ids, skip_sticky=False, start_page=page, max_pages=1)
    else: 
        fetched, err = parse_torrents_page(COMPREHENSIVE_URL, seen_ids, skip_sticky=True, start_page=page, max_pages=1)
        
    if fetched:
        for item in fetched:
            tpri, tsec, bdisc, bpri, bsec = sort_tags(item.get('tags', []))
            item['top_pri'] = tpri
            item['top_sec'] = tsec
            item['bot_disc'] = bdisc
            item['bot_pri'] = bpri
            item['bot_sec'] = bsec
            item['time_tag'] = ""
            item['clock_tag'] = ""
            
            for arr in [item['top_pri'], item['top_sec'], item['bot_disc'], item['bot_pri'], item['bot_sec']]:
                for t in list(arr):
                    if '剩余' in t.upper():
                        item['time_tag'] = t
                        arr.remove(t)
                    elif '🕒' in t:
                        item['clock_tag'] = t
                        arr.remove(t)
                        
    return jsonify({"torrents": fetched or [], "error": err})

@app.route('/api/extra/<int:tid>')
def api_extra(tid):
    img_src, size, extra_tags, release_group = fetch_details_info(tid)
    merged_tags = purify_tags(extra_tags)
    
    tid_str = str(tid)
    # 这一步极其安全：只会在 top50 的缓存库中查找。如果是下拉加载出来的新种子，根本不在库里，直接跳过保存！
    for c_type in ['recent', 'official']:
        history_dict = load_cache(c_type)
        if tid_str in history_dict:
            cached_item = history_dict[tid_str]
            final_tags = purify_tags(merge_smart_tags_from_details(merged_tags, cached_item.get('tags', [])))
            cached_item['tags'] = final_tags
            merged_tags = final_tags
            
            if img_src: cached_item['img'] = img_src
            if size and size != "未知": cached_item['size'] = size
            if release_group and not cached_item.get('group'): cached_item['group'] = release_group
            
            history_dict[tid_str] = cached_item
            save_cache(history_dict, c_type)
        
    tpri, tsec, bdisc, bpri, bsec = sort_tags(merged_tags)
    time_tag = ""
    clock_tag = ""
    for arr in [tpri, tsec, bdisc, bpri, bsec]:
        for t in list(arr):
            if '剩余' in t.upper():
                time_tag = t
                arr.remove(t)
            elif '🕒' in t:
                clock_tag = t
                arr.remove(t)
    
    return jsonify({
        "img": img_src, 
        "size": size, 
        "tags": merged_tags,
        "top_pri": tpri,
        "top_sec": tsec,
        "bot_disc": bdisc,
        "bot_pri": bpri,
        "bot_sec": bsec,
        "time_tag": time_tag,
        "clock_tag": clock_tag,
        "group": release_group
    })

@app.route('/api/signin')
def api_signin():
    try:
        r = requests.get(SIGNIN_URL, headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        r.encoding = 'utf-8'
        res_text = r.text
        if r.status_code == 200:
            if any(kw in res_text for kw in ["已经", "退下", "已签到", "请明天再来"]):
                return jsonify({"msg": "☕ 今天已经签到过了，明天再来吧。"})
            elif any(kw in res_text for kw in ["这是您的第", "签到成功", "魔力值", "电力值"]):
                match = re.search(r'(这是您的第.*?(?:魔力值|电力值)|签到成功.*?(?:魔力值|电力值))', BeautifulSoup(res_text, 'html.parser').get_text(separator=' ', strip=True))
                msg = match.group(1).replace('  ', ' ') if match else "签到成功！"
                return jsonify({"msg": f"🎉 {msg}"})
            elif "登录" in res_text:
                return jsonify({"msg": "⚠️ 签到失败：Cookie 已失效，请更新！"})
            else:
                return jsonify({"msg": "✅ 签到请求已发送，请留意积分变化。"})
        else:
            return jsonify({"msg": f"❌ 签到接口报错，状态码: {r.status_code}"})
    except Exception as e:
        return jsonify({"msg": f"❌ 网络请求出错: {str(e)}"})

@app.route('/download/<int:tid>')
def download_torrent(tid):
    try:
        r = requests.get(f"{BASE_URL}/download.php?id={tid}", headers=HEADERS, proxies=PROXIES, timeout=TIMEOUT)
        if "application/x-bittorrent" in r.headers.get('Content-Type', '') or "octet-stream" in r.headers.get('Content-Type', ''):
            return Response(r.content, mimetype="application/x-bittorrent", headers={"Content-Disposition": f"attachment;filename=ZmPT_{tid}.torrent"})
        else: return "下载失败，Cookie 可能失效或无权下载该种子。"
    except Exception as e: return f"请求下载出错: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5054)
```