---
title: vps搭建typecho博客
cover: 
swiper_index: 10
top_group_index: 10
background: '#fff'
date: 2025-09-29 15:32:36
updated:
tags:
- 技术
- 学习
categories: 技术,学习
keywords: typecho
description:
top:
top_img:
comments:
toc:
toc_number:
toc_style_simple:
copyright:
copyright_author:
copyright_author_href:
copyright_url:
copyright_info:
mathjax:
katex:
aplayer:
highlight_shrink:
aside:
ai:
---

typecho轻量级博客搭建教程
<!--more-->

<div class="video-container">
<iframe width="560" height="315" src="https://www.youtube.com/embed/AHpGphE-XC8?si=stg73B9WBwCKLdby" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>
</div>

<style>
.video-container {
    position: relative;
    width: 100%;
    padding-top: 56.25%; /* 16:9 aspect ratio (height/width = 9/16 * 100%) */
}

.video-container iframe {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
</style>
## 一、准备条件
### 1. 一台服务器或者NAS(理论上只有其他NAS都可以)
### 2. 本项目使用到的开源项目
https://github.com/typecho/typecho
### 3. 域名(可选)
## 二、vps上搭建
### 1. docker环境安装
1.1 docker安装脚本
```bash
bash <(curl -sSL https://cdn.jsdelivr.net/gh/SuperManito/LinuxMirrors@main/DockerInstallation.sh)
```
1.2 docker-compose安装脚本
```bash
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose && chmod +x /usr/local/bin/docker-compose
```
### 2. 创建docker-compose.yml文件
2.1 安装方法1：
- 创建docker-compose.yml文件
  ```bash
  mkdir typecho;cd typecho #创建一个目录，并进入此目录
  ```
- 编辑docker-compose.yml  
  ```bash
  vim docker-compose.yml
  ```
  按i进入编辑模式，粘贴如下内容：
  ```bash
  services:
  typecho:  # Typecho 博客服务
    image: joyqi/typecho:nightly-php8.2-apache  # 官方 Apache 镜像
    container_name: typecho
    ports:
      - "8383:80"  # 宿主机 8080 -> 容器 80
    environment:
      TZ: Asia/Shanghai  # 设置时区为上海
    volumes:
      - ./typecho/app/usr:/app/usr  # 当前目录存放 Typecho 文件
    depends_on:
      - db  # 依赖数据库
    restart: always  # 自动重启策略

  db:  # 数据库服务
    image: mariadb:10.6  # MariaDB 镜像
    container_name: typecho-db
    environment:
      MYSQL_ROOT_PASSWORD: root_password  # 数据库 root 密码（请修改）
      MYSQL_DATABASE: typecho  # 默认数据库
      MYSQL_USER: typecho  # 数据库用户
      MYSQL_PASSWORD: typecho_password  # 用户密码（请修改）
      TZ: Asia/Shanghai  # 时区
    volumes:
      - ./db:/var/lib/mysql  # 数据库数据存放当前目录
    restart: always
  ```

2.2 安装方法2：
```bash
cat > docker-compose.yml <<EOF
services:
  typecho:  # Typecho 博客服务
    image: joyqi/typecho:nightly-php8.2-apache  # 官方 Apache 镜像
    container_name: typecho
    ports:
      - "8383:80"  # 宿主机 8080 -> 容器 80
    environment:
      TZ: Asia/Shanghai  # 设置时区为上海
    volumes:
      - ./typecho/app/usr:/app/usr  # 当前目录存放 Typecho 文件
    depends_on:
      - db  # 依赖数据库
    restart: always  # 自动重启策略

  db:  # 数据库服务
    image: mariadb:10.6  # MariaDB 镜像
    container_name: typecho-db
    environment:
      MYSQL_ROOT_PASSWORD: root_password  # 数据库 root 密码（请修改）
      MYSQL_DATABASE: typecho  # 默认数据库
      MYSQL_USER: typecho  # 数据库用户
      MYSQL_PASSWORD: typecho_password  # 用户密码（请修改）
      TZ: Asia/Shanghai  # 时区
    volumes:
      - ./db:/var/lib/mysql  # 数据库数据存放当前目录
    restart: always
EOF
```
### 3.执行容器运行命令
```bash
docker compose up -d #运行容器
```
```bash
docker compose ps  #查看是否开启成功
```
正常显示如下：
```bash
docker compose ps
NAME         IMAGE                                 COMMAND                  SERVICE   CREATED         STATUS         PORTS
typecho      joyqi/typecho:nightly-php8.2-apache   "docker-php-entrypoi…"   typecho   8 minutes ago   Up 8 minutes   0.0.0.0:8383->80/tcp, [::]:8383->80/tcp
typecho-db   mariadb:10.6                   
```
### 4. cloudflare配置域名端口
4.1 域名dns添加AAAA记录或A记录
4.2 创建规则
- 规则名称 

- 当传入请求匹配时
  | 字段 | 运算符 | 值 |
  |---|---|---|
  | 主机名 | 等于 | 域名 |
- 目标端口重写到

  8383

### 5. 打开web页面使用
成功以后需要打开自己相应的端口(8383)防火墙就可以web端访问了
ip:8383或端口回源域名访问

5.1 初始化开始
![1](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/1.png)

5.2 下一步-根据图示配置数据库信息
![2](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/2.png)

5.3 配置管理员信息
![3](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/3.png)

5.4 安装成功
![4](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/4.png)

5.5 管理后台
![5](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/5.png)

5.6 站点logo

控制台-外观-外观设置
![6](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/6.jpg)

5.7 网站favicon
编辑 header.php 文件

在您的服务器上，找到并编辑这个文件：./typecho/app/usr/themes/你正在使用的主题名/header.php。

在文件的`<head>`和`</head>`标签之间，找到一个合适的位置（比如在 `<title>` 标签下面），添加以下代码：

```html
<link rel="icon" href="https://img.hxjx.hidns.co/file/BQACAgUAAyEGAASfZU2vAAIBYGjaLdVus_kuz1NBlSKeEiC7BFQaAAK2GAACboHRVs15w3-rQqIZNgQ.ico" type="image/png">
```

保存 header.php 文件的修改。现在您的网站应该就会显示新的 Favicon 了。
![7](https://raw.githubusercontent.com/xuhxjx/myimg/refs/heads/main/typecho/7.jpg)

5.8 代码块添加复制按钮

5.8.1 编辑 header.php 文件
在文件的`<head>`和`</head>`标签之间，找到一个合适的位置（比如在 `<title>` 标签下面），添加以下代码：
```html
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
```
5.8.2 编辑 footer.php 文件
在文件的`</body>`之前添加如下代码：
```html
<style>
/* 样式：让按钮显示在代码块的右上角 */
.code-container {
    position: relative; 
}
.code-copy-btn {
    position: absolute;
    top: 0.5em;
    right: 0.5em;
    padding: 0.3em 0.6em;
    /* 保持背景色与文字色，让它看起来更像一个图标 */
    background: none; 
    color: #fff; 
    border: none;
    border-radius: 4px;
    cursor: pointer;
    font-size: 1.1em; /* 增大图标尺寸 */
    opacity: 0.6; /* 默认半透明 */
    transition: opacity 0.3s;
    z-index: 10;
}
.code-copy-btn:hover {
    opacity: 1; /* 鼠标悬停时完全显示 */
}
/* 复制成功的提示样式 */
.code-copy-btn.success {
    color: #28a745; /* 成功后图标变绿 */
    opacity: 1;
}
</style>

<script>
document.addEventListener('DOMContentLoaded', function() {
    var pres = document.querySelectorAll('pre');

    pres.forEach(function(pre) {
        pre.classList.add('code-container');

        // 创建复制按钮，使用 Font Awesome 图标
        var button = document.createElement('button');
        button.className = 'code-copy-btn';
        // 使用 Font Awesome 剪贴板图标
        button.innerHTML = '<i class="fa-solid fa-copy"></i>'; 
        
        pre.appendChild(button);

        button.addEventListener('click', function() {
            var codeElement = pre.querySelector('code');
            
            if (codeElement) {
                var textarea = document.createElement('textarea');
                textarea.value = codeElement.innerText;
                
                textarea.style.position = 'fixed'; 
                textarea.style.top = 0;
                textarea.style.left = 0;
                
                document.body.appendChild(textarea);
                textarea.select();

                try {
                    document.execCommand('copy');
                    
                    // 复制成功：显示成功图标/文字并短暂变绿
                    var originalHTML = button.innerHTML;
                    button.classList.add('success');
                    button.innerHTML = '<i class="fa-solid fa-check"></i>'; // 换成打勾图标
                    
                    setTimeout(function() {
                        button.classList.remove('success');
                        button.innerHTML = originalHTML; // 恢复剪贴板图标
                    }, 2000);

                } catch (err) {
                    // 复制失败：恢复图标
                    button.innerHTML = '<i class="fa-solid fa-copy"></i>';
                }
                
                document.body.removeChild(textarea);
            }
        });
    });
});
</script>
```
### 6. 优化功能

[typechor插件](https://prismjs.com/download)

经过调试，网站favicon、代码块高亮、代码块复制功能完美适配默认主题

6.1 打开网站下载插件，官方默认的勾选不要修改，只选择需要增加代码块高亮的的插件。点击页面下方下载prism.js和prism.css。

6.2 修改prism.css，内容如下 ：
```css
/* PrismJS 1.30.0 (Clean Version) */
/* This file is responsible ONLY for syntax highlighting colors and basic font settings. */
/* All container styles (background, border, padding, etc.) are handled separately in the theme. */

code[class*="language-"],
pre[class*="language-"] {
	color: #000;
	font-family: Consolas, Monaco, 'Andale Mono', 'Ubuntu Mono', monospace;
	font-size: 1em;
	text-align: left;
	white-space: pre;
	word-spacing: normal;
	word-break: normal;
	word-wrap: normal;
	line-height: 1.5;

	-moz-tab-size: 4;
	-o-tab-size: 4;
	tab-size: 4;

	-webkit-hyphens: none;
	-moz-hyphens: none;
	-ms-hyphens: none;
	hyphens: none;
}

pre[class*="language-"] {
	overflow: auto; /* 保留滚动条功能 */
}

/* --- Selection & Print --- */
code[class*="language-"]::-moz-selection, code[class*="language-"] ::-moz-selection,
pre[class*="language-"]::-moz-selection, pre[class*="language-"] ::-moz-selection {
	text-shadow: none;
	background: #b3d4fc;
}
code[class*="language-"]::selection, code[class*="language-"] ::selection,
pre[class*="language-"]::selection, pre[class*="language-"] ::selection {
	text-shadow: none;
	background: #b3d4fc;
}
@media print {
	code[class*="language-"],
	pre[class*="language-"] {
		text-shadow: none;
	}
}


/* --- ALL TOKEN COLOR RULES --- */

.token.comment,
.token.prolog,
.token.doctype,
.token.cdata {
	color: #708090;
}

.token.punctuation {
	color: #999;
}

.token.namespace {
	opacity: .7;
}

.token.property,
.token.tag,
.token.boolean,
.token.number,
.token.constant,
.token.symbol,
.token.deleted {
	color: #905;
}

.token.selector,
.token.attr-name,
.token.string,
.token.char,
.token.builtin,
.token.inserted {
	color: #690;
}

.token.operator,
.token.entity,
.token.url,
.language-css .token.string,
.style .token.string {
	color: #9a6e3a;
	background: hsla(0, 0%, 100%, .5);
}

.token.atrule,
.token.attr-value,
.token.keyword {
	color: #07a;
}

.token.function,
.token.class-name {
	color: #dd4a68;
}

.token.regex,
.token.important,
.token.variable {
	color: #e90;
}

.token.important,
.token.bold {
	font-weight: bold;
}

.token.italic {
	font-style: italic;
}

.token.entity {
	cursor: help;
}
```
6.3 上传prism.js和prism.css至./typecho/app/usr/themes/default/assets文件夹下

<span style="color:rgba(164, 45, 74, 1)">assets文件夹需新建</span>

6.4 编辑 header.php 文件

在您的服务器上，找到并编辑这个文件，路径：./typecho/app/usr/themes/default/header.php

在文件的`<head>`和`</head>`标签之间，找到一个合适的位置（比如在 `<title>` 标签下面），添加以下代码：
```html
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.1/css/all.min.css">
    <link rel="icon" href="https://cdn.jsdelivr.net/gh/xuhxjx/myimg@main/favicon.ico" type="image/png"> 
<style>
    pre[class*="language-"] {
        background: #f3f3f3;        /* 一个干净的淡灰色背景 */
        /* border: 1px solid #e3e3e3; */  /* 已移除边框 */
        padding: 1em;               /* 舒适的内边距 */
        margin: 1.5em 0;            /* 和上下文的间距 */
        overflow: auto;             /* 代码过长时显示滚动条 */
        border-radius: 5px; */         /* 柔和的圆角 */
        /* box-shadow: 0 1px 2px rgba(0,0,0,0.05); */ /* 已移除阴影 */
    }

    /* 确保行内代码也有一个匹配的样式 */
    :not(pre) > code[class*="language-"] {
        padding: .2em .4em;
        margin: 0;
        font-size: 85%;
        background-color: rgba(27,31,35,.05);
        border-radius: 3px;
    }
</style>
```
6.5 编辑 footer.php 文件
在您的服务器上，找到并编辑这个文件，路径：./typecho/app/usr/themes/default/header.php
在文件的`</body>`之前添加如下代码：
```html
<style>
    /* 1. 这是新的“外层包裹容器”的样式 */
    .code-copy-wrapper {
        position: relative; /* 关键：作为按钮的定位父级 */
    }

    /* 2. 复制按钮的样式 (基本不变) */
    .code-copy-btn {
        position: absolute; /* 关键：相对于上面的 wrapper 定位 */
        top: 8px;
        right: 8px;

        padding: 5px 10px;
        font-size: 14px;

        background: transparent; /* 将背景设置为透明#e0e0e0 */
        color: #555;            /* 图标颜色可以稍浅一些，更柔和 */
        border: none;           /* 移除边框 1px solid #ccc */
        /*border-radius: 4px; */

        cursor: pointer;
        opacity: 0;
        transition: opacity 0.2s ease-in-out;
        z-index: 10; /* 确保按钮在最上层 */
    }

    /* 3. 当鼠标移动到“外层包裹容器”上时，让按钮浮现 */
    .code-copy-wrapper:hover .code-copy-btn {
        opacity: 1;
    }

    /* 4. 鼠标悬停在按钮上时的效果 */
    .code-copy-btn:hover {
    /* background: #ccc; */ /* 背景已移除，此行不再需要 */
        color: #000; /* 悬停时图标变黑，反馈更清晰 */
    }

    /* 5. 复制成功的提示样式 */
    .code-copy-btn.success {
         /* background: #D6EAF8; */ /* 移除背景色 */
        color: #2E86C1;
        /* border-color: #85C1E9; */ /* 背景已移除，此行不再需要 */
    }
</style>

<script>
document.addEventListener('DOMContentLoaded', function() {
    const pres = document.querySelectorAll('pre');

    pres.forEach(pre => {
        // 【关键修改】开始：创建并替换结构
        // 1. 创建一个新的 div 作为包裹容器
        const wrapper = document.createElement('div');
        wrapper.className = 'code-copy-wrapper';

        // 2. 在 DOM 中，将 pre 替换为 wrapper
        pre.parentNode.replaceChild(wrapper, pre);

        // 3. 将 pre 移动到 wrapper 内部
        wrapper.appendChild(pre);
        // 【关键修改】结束

        // 创建按钮
        const button = document.createElement('button');
        button.className = 'code-copy-btn';
        button.innerHTML = '<i class="fa-solid fa-copy"></i>';
        button.setAttribute('title', '复制代码');
        
        // 将按钮添加到 wrapper 中，而不是 pre 中
        wrapper.appendChild(button);

        // --- 点击事件的逻辑保持不变 ---
        button.addEventListener('click', () => {
            const codeElement = pre.querySelector('code');
            if (!codeElement) return;

            const codeToCopy = codeElement.innerText;

            if (navigator.clipboard && window.isSecureContext) {
                navigator.clipboard.writeText(codeToCopy).then(() => {
                    showSuccess(button);
                }).catch(err => {
                    copyFallback(codeToCopy, button);
                });
            } else {
                copyFallback(codeToCopy, button);
            }
        });
    });

    function showSuccess(button) {
        const originalHTML = button.innerHTML;
        button.classList.add('success');
        button.innerHTML = '<i class="fa-solid fa-check"></i> 已复制';
        
        setTimeout(() => {
            button.classList.remove('success');
            button.innerHTML = originalHTML;
        }, 2000);
    }
    
    function copyFallback(text, button) {
        const textarea = document.createElement('textarea');
        textarea.value = text;
        textarea.style.position = 'fixed';
        textarea.style.opacity = 0;
        document.body.appendChild(textarea);
        textarea.select();
        
        try {
            if (document.execCommand('copy')) {
                showSuccess(button);
            }
        } catch (err) {
            console.error('Fallback copy failed', err);
        }
        
        document.body.removeChild(textarea);
    }
});
</script>
<script src="<?php $this->options->themeUrl('assets/prism.js'); ?>"></script>
```

### 7. 加载动画
Web 前端代码（HTML/CSS/JS）是通用的，它不分 Hugo 还是 Typecho。只要是网页，这段代码都能跑。

在 Typecho 里添加这个动画，甚至比 Hugo 更简单，因为你可以在后台直接改。

请按照以下 3 步 操作：

第一步：登录 Typecho 后台
1. 进入你的 Typecho 博客后台。

2. 在顶部菜单找到 “控制台” -> “外观”。

3. 点击 “编辑当前外观”。

第二步：找到 header.php
1. 在右侧的文件列表中，找到名为 header.php (通常叫“公共头部”或“页头”) 的文件。

2. 点击它，中间会出现代码编辑框。

第三步：粘贴代码
1. 在代码框里，按 Ctrl + F 搜索 <body。

1. 找到 <body> 标签（或者 <body<?php ... ?>> 这种）。

2. 在 <body> 标签的下一行，直接粘贴我刚才给你的那段最终定稿版代码。
```html
<div id="halo-racer-loader">
  <div class="racer-container">
    <div class="racer-bar pale-1"></div>
    <div class="racer-bar pale-2"></div>
    <div class="racer-bar red"></div>
  </div>
</div>

<style>
  /* 1. 全屏遮罩 */
  #halo-racer-loader {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    z-index: 999999999 !important;
    background: #ffffff;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: opacity 0.3s ease, visibility 0.3s ease;
  }
  
  /* 深色模式背景适配 (支持常见 Typecho 主题的深色类名) */
  [data-theme='dark'] #halo-racer-loader,
  html.dark #halo-racer-loader, 
  body.dark #halo-racer-loader { background: #1a1a1a; }
  
  #halo-racer-loader.hidden { opacity: 0; visibility: hidden; }

  /* 2. 容器 */
  .racer-container {
    position: relative;
    width: 80px;
    height: 20px;
  }

  /* 3. 方块通用样式 (已移除边框) */
  .racer-bar {
    position: absolute;
    top: 0;
    left: 0;
    width: 12px;
    height: 20px;
    border-radius: 0;
    
    /* 【核心修改】删掉了 border 和 box-sizing */
    /* 现在是纯粹的色块，深色模式下也不会有白边 */
  }

  /* 4. 最淡的块 */
  .pale-1 {
    background-color: rgba(255, 71, 87, 0.2);
    z-index: 1; 
    animation: round-trip 1.5s cubic-bezier(0.4, 0, 0.2, 1) infinite;
    animation-delay: 0.15s;
  }

  /* 5. 中间淡的块 */
  .pale-2 {
    background-color: rgba(255, 71, 87, 0.5);
    z-index: 2;
    animation: round-trip 1.5s cubic-bezier(0.4, 0, 0.2, 1) infinite;
    animation-delay: 0.08s;
  }

  /* 6. 红色主角块 */
  .red {
    background-color: #ff4757;
    z-index: 10;
    animation: round-trip 1.5s cubic-bezier(0.4, 0, 0.2, 1) infinite;
    animation-delay: 0s;
  }

  /* 7. 移动轨迹 */
  @keyframes round-trip {
    0% { left: 0; }
    30% { left: 68px; }
    50% { left: 68px; }
    80% { left: 0; }
    100% { left: 0; }
  }
</style>

<script>
  (function() {
    const loader = document.getElementById('halo-racer-loader');
    const startTime = Date.now();

    function removeLoader() {
        const elapsedTime = Date.now() - startTime;
        // 极速模式判断：如果小于 500ms 加载完，则 0.1s 极速消失
        if (elapsedTime < 500) {
            loader.style.transition = 'opacity 0.1s ease';
            loader.classList.add('hidden');
            setTimeout(() => { loader.style.display = 'none'; }, 100);
        } else {
            // 正常模式：0.3s 优雅消失
            loader.classList.add('hidden');
            setTimeout(() => { loader.style.display = 'none'; }, 300);
        }
    }

    window.addEventListener('load', removeLoader);
    
    // Typecho PJAX 兼容
    document.addEventListener('pjax:complete', removeLoader);
    document.addEventListener('pjax:end', removeLoader);

    // 5秒超时强制移除
    setTimeout(function() { 
        if (!loader.classList.contains('hidden')) {
            removeLoader();
        }
    }, 5000);
  })();
</script>
```
### 8. 在原来的基础上添加折叠功能
编辑 footer.php 文件
在您的服务器上，找到并编辑这个文件，路径：./typecho/app/usr/themes/default/header.php
在文件的</body>之前添加如下代码：
```html
<style>
    /* 1. 外层包裹容器 */
    .code-copy-wrapper {
        position: relative;
        transition: all 0.3s ease;
    }

    /* 2. 按钮组通用样式 (复制 & 折叠) */
    .code-btn {
        position: absolute;
        top: 6px;                  /* 离顶部稍微近一点 */
        padding: 6px;              /* 增加内边距方便点击 */
        line-height: 1;
        font-size: 16px;           /* 图标稍微大一点 */
        
        background: transparent !important; /* 强制无背景 */
        border: none !important;            /* 强制无边框 */
        color: #999;               /* 默认颜色：浅灰，不抢眼 */
        
        cursor: pointer;
        opacity: 0;                /* 默认隐藏，鼠标放上去才显示 */
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        z-index: 20;
        border-radius: 4px;
    }
    
    /* 3. 鼠标悬停交互：好看的蓝色 */
    .code-btn:hover {
        background: transparent !important; /* 悬停也不要背景色 */
        color: #409EFF !important;          /* 悬停变亮蓝 */
        transform: scale(1.1);              /* 微微放大，更有灵动感 */
    }

    /* 4. 按钮位置 */
    /* 复制按钮在最右边 */
    .code-copy-btn {
        right: 8px;
    }
    /* 折叠按钮在复制按钮左侧 (因为没了文字，间距可以缩小) */
    .code-fold-btn {
        right: 40px; 
    }

    /* 5. 鼠标移入代码块时显示按钮 */
    .code-copy-wrapper:hover .code-btn {
        opacity: 1;
    }

    /* 6. 复制成功状态 (保持绿色反馈，或者也改成蓝色看你喜好) */
    .code-copy-btn.success {
        color: #67c23a !important; /* 成功时变绿，符合直觉 */
        transform: scale(1.2);
    }

    /* 7. 折叠状态核心样式 */
    .code-collapsed pre {
        max-height: 300px;           /* 折叠高度 */
        overflow: hidden !important; 
        
        /* 底部渐变遮罩：让代码看起来是“渐隐”的 */
        mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        -webkit-mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        
        /* 底部边框提示还有内容 */
        border-bottom: 1px dashed rgba(0,0,0,0.1); 
    }
    
    /* 8. 展开状态 */
    .code-expanded pre {
        max-height: none;
        mask-image: none;
        -webkit-mask-image: none;
        border-bottom: none;
        animation: expandAnim 0.5s ease; /* 加个简单的展开动画 */
    }

    @keyframes expandAnim {
        from { max-height: 300px; }
        to { max-height: 1000px; } /* 近似值动画 */
    }
</style>
<script>
document.addEventListener('DOMContentLoaded', function() {
    const pres = document.querySelectorAll('pre');

    pres.forEach(pre => {
        // 1. 创建 wrapper
        const wrapper = document.createElement('div');
        wrapper.className = 'code-copy-wrapper';
        pre.parentNode.replaceChild(wrapper, pre);
        wrapper.appendChild(pre);

        // --- A. 复制按钮 ---
        const copyBtn = document.createElement('button');
        copyBtn.className = 'code-btn code-copy-btn';
        
        // 【修正】改回实心图标 fa-solid
        copyBtn.innerHTML = '<i class="fa-solid fa-copy"></i>'; 
        
        copyBtn.setAttribute('title', '复制代码'); 
        wrapper.appendChild(copyBtn);

        copyBtn.addEventListener('click', () => {
            const code = pre.querySelector('code');
            if (!code) return;
            const text = code.innerText;
            
            if (navigator.clipboard && window.isSecureContext) {
                navigator.clipboard.writeText(text).then(() => showSuccess(copyBtn))
                    .catch(() => copyFallback(text, copyBtn));
            } else {
                copyFallback(text, copyBtn);
            }
        });

        // --- B. 折叠按钮 ---
        const codeHeight = pre.offsetHeight;
        const foldHeightThreshold = 350; 

        if (codeHeight > foldHeightThreshold) {
            wrapper.classList.add('code-collapsed');

            const foldBtn = document.createElement('button');
            foldBtn.className = 'code-btn code-fold-btn';
            // 保持双箭头图标
            foldBtn.innerHTML = '<i class="fa-solid fa-angles-down"></i>';
            foldBtn.setAttribute('title', '展开代码');
            wrapper.appendChild(foldBtn);

            foldBtn.addEventListener('click', () => {
                const isCollapsed = wrapper.classList.contains('code-collapsed');
                
                if (isCollapsed) {
                    // 展开
                    wrapper.classList.remove('code-collapsed');
                    wrapper.classList.add('code-expanded');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angles-up"></i>';
                    foldBtn.setAttribute('title', '收起代码');
                } else {
                    // 收起
                    wrapper.classList.remove('code-expanded');
                    wrapper.classList.add('code-collapsed');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angles-down"></i>';
                    foldBtn.setAttribute('title', '展开代码');
                    
                    wrapper.scrollIntoView({behavior: "smooth", block: "nearest"});
                }
            });
        }
    });

    // --- 辅助函数 ---
    function showSuccess(btn) {
        const originalHTML = btn.innerHTML;
        btn.classList.add('success');
        btn.innerHTML = '<i class="fa-solid fa-check"></i>';
        setTimeout(() => {
            btn.classList.remove('success');
            btn.innerHTML = originalHTML;
        }, 2000);
    }

    function copyFallback(text, btn) {
        const textarea = document.createElement('textarea');
        textarea.value = text;
        textarea.style.position = 'fixed';
        textarea.style.opacity = 0;
        document.body.appendChild(textarea);
        textarea.select();
        try {
            if (document.execCommand('copy')) showSuccess(btn);
        } catch (err) {
            console.error('Fallback failed', err);
        }
        document.body.removeChild(textarea);
    }
});
</script>
<script src="<?php $this->options->themeUrl('assets/prism.js'); ?>"></script>
```