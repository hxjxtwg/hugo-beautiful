---

title: "谷歌云typecho配置"

author: "xxsky"

type: "posts"

date: 2025-12-10T17:42:30+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

添加加载动画、代码框复制折叠功能、代码高亮、文章置顶权重。

<!--more-->

### 1. 图片、css、js文件上传
使用谷歌自带的网页SSH上传logo.png、favicon.ico、prism.css、prism.js文件。
上传完后，用命令把它移动到你的网站目录。assets需新建。
```bash
sudo mv logo.png /root/typecho/app/usr/themes/default/img
sudo mv favicon.ico /root/typecho/app/usr/themes/default/img
sudo mv prism.css /root/typecho/app/usr/themes/default/assets
sudo mv prism.js /root/typecho/app/usr/themes/default/assets
```
### 2. 站点LOGO地址
```bash
https://tych.cc.cd/usr/themes/default/img/logo.png
```
### 3. 修改header.php
在`</head>`之前添加站点图标、css样式的代码：
```html
    <link rel="icon" href="/usr/themes/default/img/favicon.ico" type="image/png">
    <link rel="stylesheet" href="<?php $this->options->themeUrl('assets/prism.css'); ?>">
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
在`<body>`下面添加加载动画代码：
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
### 4. 修改footer.php
在`</body>`之前添加复制、折叠功能代码：

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
或者气泡提示带背景的：
```html
<style>
    /* 1. 外层包裹容器 */
    .code-copy-wrapper {
        position: relative;
        transition: all 0.3s ease;
    }

    /* 2. 按钮组通用样式 */
    .code-btn {
        position: absolute;
        top: 6px;
        padding: 6px;
        line-height: 1;
        font-size: 16px; 
        
        background: transparent !important;
        border: none !important;
        color: #999;
        
        cursor: pointer;
        opacity: 0; 
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        z-index: 20;
        border-radius: 4px;
        
        /* 关键：去掉浏览器默认点击后的轮廓线 */
        outline: none !important; 
    }
    
    /* 3. 鼠标悬停交互：好看的蓝色 */
    .code-btn:hover {
        background: transparent !important;
        color: #409EFF !important;
        transform: scale(1.1);
    }

    /* --- 新增：自定义无边框提示词 (Tooltip) 样式 --- */
    .code-btn[data-tooltip] {
        position: absolute; 
        /* 确保伪元素定位参考该按钮 */
    }
    
    /* 提示词气泡 */
    .code-btn[data-tooltip]:hover::before {
        content: attr(data-tooltip); /*以此获取JS中设置的文字*/
        position: absolute;
        bottom: 100%;        /* 显示在按钮上方 */
        right: 50%;          /* 居中调整 */
        transform: translateX(50%);
        margin-bottom: 5px;  /* 距离按钮的间距 */
        
        padding: 4px 8px;
        background: rgba(71, 99, 255, 0); /* 浅色背景 */
        color: #000;
        font-size: 12px;
        white-space: nowrap; /* 不换行 */
        border-radius: 4px;
        pointer-events: none; /* 鼠标穿透，防止挡住点击 */
        
        /* 关键：无边框 */
        border: none !important; 
        box-shadow: 0 2px 4px rgba(0,0,0,0.2);
        
        /* 淡入动画 */
        opacity: 0;
        animation: tooltipFadeIn 0.2s forwards;
    }
    
    @keyframes tooltipFadeIn {
        to { opacity: 1; }
    }

    /* 4. 按钮位置 */
    .code-copy-btn {
        right: 8px;
    }
    .code-fold-btn {
        right: 40px; 
    }

    /* 5. 鼠标移入代码块时显示按钮 */
    .code-copy-wrapper:hover .code-btn {
        opacity: 1;
    }

    /* 6. 复制成功状态 */
    .code-copy-btn.success {
        color: #67c23a !important; 
        transform: scale(1.2);
    }

    /* 7. 折叠状态核心样式 */
    .code-collapsed pre {
        max-height: 100px; 
        overflow: hidden !important; 
        
        mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        -webkit-mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        
        border-bottom: 1px dashed rgba(0,0,0,0.1); 
    }
    
    /* 8. 展开状态 */
    .code-expanded pre {
        max-height: none;
        mask-image: none;
        -webkit-mask-image: none;
        border-bottom: none;
        animation: expandAnim 0.5s ease; 
    }

    @keyframes expandAnim {
        from { max-height: 300px; }
        to { max-height: 1000px; } 
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
        
        // 【修改1】使用 fa-regular (常规线条) 代替 fa-solid，看起来更细
        // 如果您的网站没引入 Regular 版字体，可能需要改回 fa-solid 并通过 font-size 调小
        copyBtn.innerHTML = '<i class="fa-regular fa-copy"></i>'; 
        
        // 【修改2】使用 data-tooltip 代替 title，去掉系统自带的边框提示
        copyBtn.setAttribute('data-tooltip', '复制'); 
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
        const foldHeightThreshold = 150; 

        if (codeHeight > foldHeightThreshold) {
            wrapper.classList.add('code-collapsed');

            const foldBtn = document.createElement('button');
            foldBtn.className = 'code-btn code-fold-btn';
            
            // 【修改3】使用 fa-angle-down (单箭头) 且用 fa-solid 配以较小的字重，或者 fa-regular
            // 相比原来的 fa-angles-down (双箭头)，这个更简洁纤细
            foldBtn.innerHTML = '<i class="fa-solid fa-angle-down"></i>';
            foldBtn.setAttribute('data-tooltip', '展开');
            wrapper.appendChild(foldBtn);

            foldBtn.addEventListener('click', () => {
                const isCollapsed = wrapper.classList.contains('code-collapsed');
                
                if (isCollapsed) {
                    // 展开
                    wrapper.classList.remove('code-collapsed');
                    wrapper.classList.add('code-expanded');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angle-up"></i>';
                    foldBtn.setAttribute('data-tooltip', '收起');
                } else {
                    // 收起
                    wrapper.classList.remove('code-expanded');
                    wrapper.classList.add('code-collapsed');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angle-down"></i>';
                    foldBtn.setAttribute('data-tooltip', '展开');
                    
                    wrapper.scrollIntoView({behavior: "smooth", block: "nearest"});
                }
            });
        }
    });

    // --- 辅助函数 ---
    function showSuccess(btn) {
        // 缓存原来的图标
        const originalIcon = btn.innerHTML;
        const originalTooltip = btn.getAttribute('data-tooltip');

        btn.classList.add('success');
        btn.innerHTML = '<i class="fa-solid fa-check"></i>';
        btn.setAttribute('data-tooltip', 'copied'); // 复制成功时的提示

        setTimeout(() => {
            btn.classList.remove('success');
            btn.innerHTML = originalIcon;
            btn.setAttribute('data-tooltip', originalTooltip); // 恢复原来的提示
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
或者提示词无背景的
```bash
<style>
    /* 1. 外层包裹容器 */
    .code-copy-wrapper {
        position: relative;
        transition: all 0.3s ease;
    }

    /* 2. 按钮组通用样式 */
    .code-btn {
        position: absolute;
        top: 6px;
        padding: 6px;
        line-height: 1;
        font-size: 16px; 
        
        background: transparent !important;
        border: none !important;
        color: #999;
        
        cursor: pointer;
        opacity: 0; 
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        z-index: 20;
        border-radius: 4px;
        
        outline: none !important; 
    }
    
    /* 3. 鼠标悬停交互 */
    .code-btn:hover {
        background: transparent !important;
        color: #409EFF !important;
        transform: scale(1.1);
    }

    /* --- 新增：自定义无边框、无背景提示词 (Tooltip) 样式 --- */
    .code-btn[data-tooltip] {
        position: absolute; 
    }
    
    /* 提示词文字 */
    .code-btn[data-tooltip]:hover::before {
        content: attr(data-tooltip);
        position: absolute;
        bottom: 100%;        
        right: 50%;          
        transform: translateX(50%);
        margin-bottom: 5px;  /*稍微离图标近一点*/
        
        /* 核心修改：无背景风格 */
        background: transparent !important; /* 不要背景色 */
        color: #409EFF;      /* 文字改成蓝色，防止背景没了看不清 */
        padding: 0;          /* 去掉内边距 */
        box-shadow: none;    /* 去掉阴影 */
        
        font-size: 11px;     /* 稍微大一点点让文字清楚 */
        font-weight: extra light;   /* 细一点 */
        white-space: nowrap; 
        pointer-events: none; 
        
        opacity: 0;
        animation: tooltipFadeIn 0.2s forwards;
    }
    
    @keyframes tooltipFadeIn {
        to { opacity: 1; }
    }

    /* 4. 按钮位置 */
    .code-copy-btn {
        right: 8px;
    }
    .code-fold-btn {
        right: 40px; 
    }

    /* 5. 鼠标移入代码块时显示按钮 */
    .code-copy-wrapper:hover .code-btn {
        opacity: 1;
    }

    /* 6. 复制成功状态 */
    .code-copy-btn.success {
        color: #67c23a !important; 
        transform: scale(1.2);
    }
    
    /* 复制成功时的提示词颜色也变成绿色 */
    .code-copy-btn.success[data-tooltip]:hover::before {
        color: #67c23a;
    }

    /* 7. 折叠状态核心样式 */
    .code-collapsed pre {
        max-height: 100px; 
        overflow: hidden !important; 
        
        mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        -webkit-mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        
        border-bottom: 1px dashed rgba(0,0,0,0.1); 
    }
    
    /* 8. 展开状态 */
    .code-expanded pre {
        max-height: none;
        mask-image: none;
        -webkit-mask-image: none;
        border-bottom: none;
        animation: expandAnim 0.5s ease; 
    }

    @keyframes expandAnim {
        from { max-height: 300px; }
        to { max-height: 1000px; } 
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
        
        // 使用 fa-regular 细线条
        copyBtn.innerHTML = '<i class="fa-regular fa-copy"></i>'; 
        copyBtn.setAttribute('data-tooltip', '复制'); 
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
        const foldHeightThreshold = 150; 

        if (codeHeight > foldHeightThreshold) {
            wrapper.classList.add('code-collapsed');

            const foldBtn = document.createElement('button');
            foldBtn.className = 'code-btn code-fold-btn';
            
            // 使用 fa-angle-down 单箭头
            foldBtn.innerHTML = '<i class="fa-solid fa-angle-down"></i>';
            foldBtn.setAttribute('data-tooltip', '展开');
            wrapper.appendChild(foldBtn);

            foldBtn.addEventListener('click', () => {
                const isCollapsed = wrapper.classList.contains('code-collapsed');
                
                if (isCollapsed) {
                    // 展开
                    wrapper.classList.remove('code-collapsed');
                    wrapper.classList.add('code-expanded');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angle-up"></i>';
                    foldBtn.setAttribute('data-tooltip', '收起');
                } else {
                    // 收起
                    wrapper.classList.remove('code-expanded');
                    wrapper.classList.add('code-collapsed');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angle-down"></i>';
                    foldBtn.setAttribute('data-tooltip', '展开');
                    
                    wrapper.scrollIntoView({behavior: "smooth", block: "nearest"});
                }
            });
        }
    });

    // --- 辅助函数 ---
    function showSuccess(btn) {
        const originalIcon = btn.innerHTML;
        const originalTooltip = btn.getAttribute('data-tooltip');

        btn.classList.add('success');
        btn.innerHTML = '<i class="fa-solid fa-check"></i>';
        btn.setAttribute('data-tooltip', '已复制！'); 

        setTimeout(() => {
            btn.classList.remove('success');
            btn.innerHTML = originalIcon;
            btn.setAttribute('data-tooltip', originalTooltip); 
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
或者没有提示词
```html
<style>
    /* 1. 外层包裹容器 */
    .code-copy-wrapper {
        position: relative;
        transition: all 0.3s ease;
    }

    /* 2. 按钮组通用样式 */
    .code-btn {
        position: absolute;
        top: 6px;
        padding: 6px;
        line-height: 1;
        font-size: 16px; 
        
        background: transparent !important;
        border: none !important;
        color: #999;
        
        cursor: pointer;
        opacity: 0; 
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
        z-index: 20;
        border-radius: 4px;
        outline: none !important; 
    }
    
    /* 3. 鼠标悬停交互 */
    .code-btn:hover {
        background: transparent !important;
        color: #409EFF !important; /* 悬停变蓝 */
        transform: scale(1.1);
    }

    /* 4. 按钮位置 */
    .code-copy-btn {
        right: 8px;
    }
    .code-fold-btn {
        right: 40px; 
    }

    /* 5. 鼠标移入代码块时显示按钮 */
    .code-copy-wrapper:hover .code-btn {
        opacity: 1;
    }

    /* 6. 复制成功状态 */
    .code-copy-btn.success {
        color: #67c23a !important; 
        transform: scale(1.2);
    }

    /* 7. 折叠状态核心样式 */
    .code-collapsed pre {
        max-height: 300px; 
        overflow: hidden !important; 
        mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        -webkit-mask-image: linear-gradient(to bottom, black 80%, transparent 100%);
        border-bottom: 1px dashed rgba(0,0,0,0.1); 
    }
    
    /* 8. 展开状态 */
    .code-expanded pre {
        max-height: none;
        mask-image: none;
        -webkit-mask-image: none;
        border-bottom: none;
        animation: expandAnim 0.5s ease; 
    }

    @keyframes expandAnim {
        from { max-height: 300px; }
        to { max-height: 1000px; } 
    }
    
    /* 9. 【双重保险】强制隐藏所有伪元素提示词 */
    .code-btn::before, .code-btn::after {
        display: none !important;
        content: "" !important;
    }
</style>

<script>
document.addEventListener('DOMContentLoaded', function() {
    const pres = document.querySelectorAll('pre');

    pres.forEach(pre => {
        const wrapper = document.createElement('div');
        wrapper.className = 'code-copy-wrapper';
        pre.parentNode.replaceChild(wrapper, pre);
        wrapper.appendChild(pre);

        // --- A. 复制按钮 ---
        const copyBtn = document.createElement('button');
        copyBtn.className = 'code-btn code-copy-btn';
        // 这里删除了 data-tooltip 属性
        copyBtn.innerHTML = '<i class="fa-regular fa-copy"></i>'; 
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
            // 这里删除了 data-tooltip 属性
            foldBtn.innerHTML = '<i class="fa-solid fa-angle-down"></i>';
            wrapper.appendChild(foldBtn);

            foldBtn.addEventListener('click', () => {
                const isCollapsed = wrapper.classList.contains('code-collapsed');
                if (isCollapsed) {
                    wrapper.classList.remove('code-collapsed');
                    wrapper.classList.add('code-expanded');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angle-up"></i>';
                    // 这里删除了 data-tooltip 设置
                } else {
                    wrapper.classList.remove('code-expanded');
                    wrapper.classList.add('code-collapsed');
                    foldBtn.innerHTML = '<i class="fa-solid fa-angle-down"></i>';
                    // 这里删除了 data-tooltip 设置
                    wrapper.scrollIntoView({behavior: "smooth", block: "nearest"});
                }
            });
        }
    });

    // --- 辅助函数 ---
    function showSuccess(btn) {
        const originalIcon = btn.innerHTML;
        btn.classList.add('success');
        btn.innerHTML = '<i class="fa-solid fa-check"></i>';
        // 这里删除了 data-tooltip 更改

        setTimeout(() => {
            btn.classList.remove('success');
            btn.innerHTML = originalIcon;
            // 这里删除了 data-tooltip 恢复
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
### 5. 文章置顶功能
5.1 修改functions.php
最底添加代码：
```html
function themeFields($layout)
{
    // --- 3. 文章排序权重 (提示语已美化) ---
    $weight = new Typecho_Widget_Helper_Form_Element_Text(
        'article_weight', 
        NULL, 
        '0', 
        _t('文章置顶权重'), 
        // 使用 div 包裹进行样式美化，防止错位
        _t('<div style="margin-top: 8px; padding: 10px; background: #f9f9f9; border: 1px solid #eee; border-radius: 4px; color: #666; font-size: 13px; line-height: 1.6;">
            <strong>设置规则：</strong><br>
            1. 输入数字，<b>数字越大越靠前</b>。<br>
            2. 默认为 0 或留空，即不置顶（按时间排序）。<br>
            3. 建议填写 100、99 等大数字。
            </div>')
    );
    $layout->addItem($weight);
}
```
也可以把logo的功能加进去。

5.2 修改index.php
只有首页也就是第1页去重。在`<?php if ($this->have()): ?>`与`<?php while ($this->next()): ?>`添加如下代码：
```html
    <?php 
    /** =============================================
     * 置顶逻辑 (最终纯净版)
     * ============================================= */
    $stickyCids = array(); 
    
    if ($this->is('index') && $this->_currentPage == 1) {
        $db = Typecho_Db::get();
        
        // 1. SQL 全量查询
        $sql = $db->select('table.contents.*', 'table.fields.str_value AS sticky_weight')
            ->from('table.contents')
            ->join('table.fields', 'table.contents.cid = table.fields.cid', Typecho_Db::INNER_JOIN)
            ->where('table.fields.name = ?', 'article_weight') 
            ->where('table.contents.type = ?', 'post')
            ->where('table.contents.status = ?', 'publish');
            
        $rawPosts = $db->fetchAll($sql);

        // 2. 过滤和排序
        $validSticky = array();
        foreach ($rawPosts as $row) {
            $weight = intval($row['sticky_weight']);
            if ($weight > 0) {
                $row['sticky_weight'] = $weight;
                $validSticky[] = $row;
            }
        }

        usort($validSticky, function($a, $b) {
            return $b['sticky_weight'] - $a['sticky_weight'];
        });

        // 3. 循环输出
        if (!empty($validSticky)) {
            foreach($validSticky as $row) {
                $stickyCids[] = $row['cid']; 
                
                // 初始化组件并手动填充数据，防止查询错误
                $stickyPost = $this->widget('Widget_Abstract_Contents', 'sticky_manual_'.$row['cid']);
                $stickyPost->push($row);
    ?>
        <article class="post sticky-post" itemscope itemtype="http://schema.org/BlogPosting" style="border-bottom: 1px solid #f0f0f0; margin-bottom: 25px; padding-bottom: 20px;">
            <h2 class="post-title" itemprop="name headline">
                <span style="background:#ff4757; color:#fff; font-size:12px; padding:3px 6px; border-radius:3px; vertical-align:middle; margin-right:6px; font-weight:normal;">
                    TOP <?php echo $row['sticky_weight']; ?>
                </span>
                <a itemprop="url" href="<?php $stickyPost->permalink(); ?>"><?php $stickyPost->title(); ?></a>
            </h2>
            
            <ul class="post-meta">
                <li><?php _e('作者: '); ?><a href="<?php $stickyPost->author->permalink(); ?>"><?php $stickyPost->author(); ?></a></li>
                <li><?php _e('时间: '); ?><time><?php $stickyPost->date(); ?></time></li>
                <li><?php _e('分类: '); ?><?php $stickyPost->category(','); ?></li>
            </ul>
            
            <div class="post-content" itemprop="articleBody">
                <?php $stickyPost->content(_t('阅读剩余部分')); ?>
            </div>
        </article>
    <?php
            }
        }
    }
    ?>
```
或者都去重，但第1页以后的页面会缺失文章数量。在`<?php if ($this->have()): ?>`与`<?php while ($this->next()): ?>`添加如下代码：
```html
<?php 
/** =============================================
 * 置顶逻辑 (修复分页重复显示版)
 * ============================================= */
$stickyCids = array(); 
$stickyHTML = ""; // 用于缓存置顶文章的HTML，稍后输出

// 1. 【无条件执行】查询数据库，找出所有置顶文章ID
// 只要是首页(index)，不管第几页，都要查出来，为了后续过滤
if ($this->is('index')) {
    $db = Typecho_Db::get();
    
    $sql = $db->select('table.contents.*', 'table.fields.str_value AS sticky_weight')
        ->from('table.contents')
        ->join('table.fields', 'table.contents.cid = table.fields.cid', Typecho_Db::INNER_JOIN)
        ->where('table.fields.name = ?', 'article_weight') 
        ->where('table.contents.type = ?', 'post')
        ->where('table.contents.status = ?', 'publish');
        
    $rawPosts = $db->fetchAll($sql);

    // 2. 提取数据并排序
    $validSticky = array();
    foreach ($rawPosts as $row) {
        $weight = intval($row['sticky_weight']);
        if ($weight > 0) {
            $row['sticky_weight'] = $weight;
            $validSticky[] = $row;
            // 【关键】无论第几页，都要把ID记下来，用于下面主循环的去重
            $stickyCids[] = $row['cid']; 
        }
    }

    usort($validSticky, function($a, $b) {
        return $b['sticky_weight'] - $a['sticky_weight'];
    });

    // 3. 【有条件输出】只在第一页渲染HTML
    if ($this->_currentPage == 1 && !empty($validSticky)) {
        foreach($validSticky as $row) {
            // 初始化组件
            $stickyPost = $this->widget('Widget_Abstract_Contents', 'sticky_manual_'.$row['cid']);
            $stickyPost->push($row);
            
            // 下面是输出样式，您可以根据需要微调
?>
    <article class="post sticky-post" itemscope itemtype="http://schema.org/BlogPosting" style="border-bottom: 1px solid #f0f0f0; margin-bottom: 25px; padding-bottom: 20px;">
        <h2 class="post-title" itemprop="name headline">
            <span style="background:#ff4757; color:#fff; font-size:12px; padding:3px 6px; border-radius:3px; vertical-align:middle; margin-right:6px; font-weight:normal;">
                TOP <?php echo $row['sticky_weight']; ?>
            </span>
            <a itemprop="url" href="<?php $stickyPost->permalink(); ?>"><?php $stickyPost->title(); ?></a>
        </h2>
        
        <ul class="post-meta">
            <li><?php _e('作者: '); ?><a href="<?php $stickyPost->author->permalink(); ?>"><?php $stickyPost->author(); ?></a></li>
            <li><?php _e('时间: '); ?><time><?php $stickyPost->date(); ?></time></li>
            <li><?php _e('分类: '); ?><?php $stickyPost->category(','); ?></li>
        </ul>
        
        <div class="post-content" itemprop="articleBody">
            <?php $stickyPost->content(_t('阅读剩余部分')); ?>
        </div>
    </article>
<?php
        } // end foreach
    } // end if page==1
} // end if index
?>
```
过滤置顶文章还需在<?php while ($this->next()): ?>下加上代码：
```html
<?php if (isset($stickyCids) && in_array($this->cid, $stickyCids)) continue; ?>
```
### 6.prism.css、prism.js文件
6.1 prism.css内容：
```css
/* ====================================================== */
/* 第一部分：PrismJS 高亮样式 (保持不变) */
/* ====================================================== */

/* PrismJS 1.30.0 (Clean Version) */
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
	overflow: auto;
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
.token.comment, .token.prolog, .token.doctype, .token.cdata { color: #708090; }
.token.punctuation { color: #999; }
.token.namespace { opacity: .7; }
.token.property, .token.tag, .token.boolean, .token.number, .token.constant, .token.symbol, .token.deleted { color: #905; }
.token.selector, .token.attr-name, .token.string, .token.char, .token.builtin, .token.inserted { color: #690; }
.token.operator, .token.entity, .token.url, .language-css .token.string, .style .token.string { color: #9a6e3a; background: hsla(0, 0%, 100%, .5); }
.token.atrule, .token.attr-value, .token.keyword { color: #07a; }
.token.function, .token.class-name { color: #dd4a68; }
.token.regex, .token.important, .token.variable { color: #e90; }
.token.important, .token.bold { font-weight: bold; }
.token.italic { font-style: italic; }
.token.entity { cursor: help; }


/* ====================================================== */
/* 第二部分：提示词样式 (无背景、蓝色文字) */
/* ====================================================== */

/* 必须显式定义，否则会被下面的清除样式影响 */
.code-btn[data-tooltip]:hover::before {
    content: attr(data-tooltip) !important; /* 强制显示内容 */
    display: block !important;              /* 强制显示区块 */
    visibility: visible !important;
    opacity: 1 !important;
    
    position: absolute;
    bottom: 100%;        
    right: 50%;          
    transform: translateX(50%);
    margin-bottom: 2px;
    
    /* 无背景风格 */
    background: transparent !important; 
    color: #409EFF !important;      
    padding: 0;          
    box-shadow: none;
    border: none !important;
    
    font-size: 13px;     
    font-weight: bold;   
    white-space: nowrap; 
    pointer-events: none; 
    z-index: 100;
}


/* ====================================================== */
/* 第三部分：超细图标样式 (SVG 替换 FontAwesome) */
/* ====================================================== */

/* 1. 图标基础重置 (注意：删除了对 .code-btn::before 的误伤) */
.fa-solid, [class*="fa-"], .fa-regular {
    font-family: sans-serif !important;
    font-style: normal;
    display: inline-block !important;
    width: 18px !important;    
    height: 18px !important;
    background-repeat: no-repeat !important;
    background-position: center !important;
    background-size: contain !important;
    vertical-align: middle;
    color: transparent !important; /* 让原来的字体图标透明 */
    transition: all 0.2s ease;
}

/* 清空 FontAwesome 自身的伪元素，但不清空按钮的伪元素 */
.fa-solid::before, [class*="fa-"]::before,
.fa-solid::after, [class*="fa-"]::after {
    content: "" !important;
    display: none !important;
}

/* --------------------------------------------------- */
/* 2. 定义图标 (SVG stroke-width=1.2 超细线条) */
/* --------------------------------------------------- */

/* [复制图标] 灰色超细线条 */
.fa-copy, .fa-clipboard, .fa-clone {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23666' stroke-width='1.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Crect x='9' y='9' width='13' height='13' rx='2' ry='2'/%3E%3Cpath d='M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1'/%3E%3C/svg%3E") !important;
    opacity: 0.8;
}

/* 悬停时变蓝 */
.code-btn:hover .fa-copy, .code-btn:hover .fa-clipboard {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23409EFF' stroke-width='1.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Crect x='9' y='9' width='13' height='13' rx='2' ry='2'/%3E%3Cpath d='M5 15H4a2 2 0 0 1-2-2V4a2 2 0 0 1 2-2h9a2 2 0 0 1 2 2v1'/%3E%3C/svg%3E") !important;
    opacity: 1;
    transform: scale(1.05);
}

/* [复制成功] 绿色超细对勾 */
.fa-check {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%2367c23a' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='20 6 9 17 4 12'/%3E%3C/svg%3E") !important;
    opacity: 1;
    animation: bounce 0.3s;
}

/* [折叠/收起] 向上超细 V 型 */
.fa-angles-up, .fa-angle-up, .fa-chevron-up {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23666' stroke-width='1.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='18 15 12 9 6 15'/%3E%3C/svg%3E") !important;
    opacity: 0.8;
}
.code-btn:hover .fa-angles-up, .code-btn:hover .fa-angle-up {
    opacity: 1;
    /* 悬停变蓝，保持一致 */
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23409EFF' stroke-width='1.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='18 15 12 9 6 15'/%3E%3C/svg%3E") !important; 
}

/* [展开/向下] 向下超细 V 型 */
.fa-angles-down, .fa-angle-down, .fa-chevron-down {
    background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23666' stroke-width='1.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E") !important;
    opacity: 0.8;
}

/* 展开图标悬停变蓝 */
.code-btn:hover .fa-angles-down, .code-btn:hover .fa-angle-down {
     background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 24 24' fill='none' stroke='%23409EFF' stroke-width='1.2' stroke-linecap='round' stroke-linejoin='round'%3E%3Cpolyline points='6 9 12 15 18 9'/%3E%3C/svg%3E") !important;
}

@keyframes bounce {
    0% { transform: scale(0.8); }
    50% { transform: scale(1.2); }
    100% { transform: scale(1); }
}
```
6.2 prism.js内容：
```javascript
/* PrismJS 1.30.0
https://prismjs.com/download.html#themes=prism&languages=markup+css+clike+javascript+bash+c+cpp+docker+markup-templating+nginx+php */
var _self="undefined"!=typeof window?window:"undefined"!=typeof WorkerGlobalScope&&self instanceof WorkerGlobalScope?self:{},Prism=function(e){var n=/(?:^|\s)lang(?:uage)?-([\w-]+)(?=\s|$)/i,t=0,r={},a={manual:e.Prism&&e.Prism.manual,disableWorkerMessageHandler:e.Prism&&e.Prism.disableWorkerMessageHandler,util:{encode:function e(n){return n instanceof i?new i(n.type,e(n.content),n.alias):Array.isArray(n)?n.map(e):n.replace(/&/g,"&amp;").replace(/</g,"&lt;").replace(/\u00a0/g," ")},type:function(e){return Object.prototype.toString.call(e).slice(8,-1)},objId:function(e){return e.__id||Object.defineProperty(e,"__id",{value:++t}),e.__id},clone:function e(n,t){var r,i;switch(t=t||{},a.util.type(n)){case"Object":if(i=a.util.objId(n),t[i])return t[i];for(var l in r={},t[i]=r,n)n.hasOwnProperty(l)&&(r[l]=e(n[l],t));return r;case"Array":return i=a.util.objId(n),t[i]?t[i]:(r=[],t[i]=r,n.forEach((function(n,a){r[a]=e(n,t)})),r);default:return n}},getLanguage:function(e){for(;e;){var t=n.exec(e.className);if(t)return t[1].toLowerCase();e=e.parentElement}return"none"},setLanguage:function(e,t){e.className=e.className.replace(RegExp(n,"gi"),""),e.classList.add("language-"+t)},currentScript:function(){if("undefined"==typeof document)return null;if(document.currentScript&&"SCRIPT"===document.currentScript.tagName)return document.currentScript;try{throw new Error}catch(r){var e=(/at [^(\r\n]*\((.*):[^:]+:[^:]+\)$/i.exec(r.stack)||[])[1];if(e){var n=document.getElementsByTagName("script");for(var t in n)if(n[t].src==e)return n[t]}return null}},isActive:function(e,n,t){for(var r="no-"+n;e;){var a=e.classList;if(a.contains(n))return!0;if(a.contains(r))return!1;e=e.parentElement}return!!t}},languages:{plain:r,plaintext:r,text:r,txt:r,extend:function(e,n){var t=a.util.clone(a.languages[e]);for(var r in n)t[r]=n[r];return t},insertBefore:function(e,n,t,r){var i=(r=r||a.languages)[e],l={};for(var o in i)if(i.hasOwnProperty(o)){if(o==n)for(var s in t)t.hasOwnProperty(s)&&(l[s]=t[s]);t.hasOwnProperty(o)||(l[o]=i[o])}var u=r[e];return r[e]=l,a.languages.DFS(a.languages,(function(n,t){t===u&&n!=e&&(this[n]=l)})),l},DFS:function e(n,t,r,i){i=i||{};var l=a.util.objId;for(var o in n)if(n.hasOwnProperty(o)){t.call(n,o,n[o],r||o);var s=n[o],u=a.util.type(s);"Object"!==u||i[l(s)]?"Array"!==u||i[l(s)]||(i[l(s)]=!0,e(s,t,o,i)):(i[l(s)]=!0,e(s,t,null,i))}}},plugins:{},highlightAll:function(e,n){a.highlightAllUnder(document,e,n)},highlightAllUnder:function(e,n,t){var r={callback:t,container:e,selector:'code[class*="language-"], [class*="language-"] code, code[class*="lang-"], [class*="lang-"] code'};a.hooks.run("before-highlightall",r),r.elements=Array.prototype.slice.apply(r.container.querySelectorAll(r.selector)),a.hooks.run("before-all-elements-highlight",r);for(var i,l=0;i=r.elements[l++];)a.highlightElement(i,!0===n,r.callback)},highlightElement:function(n,t,r){var i=a.util.getLanguage(n),l=a.languages[i];a.util.setLanguage(n,i);var o=n.parentElement;o&&"pre"===o.nodeName.toLowerCase()&&a.util.setLanguage(o,i);var s={element:n,language:i,grammar:l,code:n.textContent};function u(e){s.highlightedCode=e,a.hooks.run("before-insert",s),s.element.innerHTML=s.highlightedCode,a.hooks.run("after-highlight",s),a.hooks.run("complete",s),r&&r.call(s.element)}if(a.hooks.run("before-sanity-check",s),(o=s.element.parentElement)&&"pre"===o.nodeName.toLowerCase()&&!o.hasAttribute("tabindex")&&o.setAttribute("tabindex","0"),!s.code)return a.hooks.run("complete",s),void(r&&r.call(s.element));if(a.hooks.run("before-highlight",s),s.grammar)if(t&&e.Worker){var c=new Worker(a.filename);c.onmessage=function(e){u(e.data)},c.postMessage(JSON.stringify({language:s.language,code:s.code,immediateClose:!0}))}else u(a.highlight(s.code,s.grammar,s.language));else u(a.util.encode(s.code))},highlight:function(e,n,t){var r={code:e,grammar:n,language:t};if(a.hooks.run("before-tokenize",r),!r.grammar)throw new Error('The language "'+r.language+'" has no grammar.');return r.tokens=a.tokenize(r.code,r.grammar),a.hooks.run("after-tokenize",r),i.stringify(a.util.encode(r.tokens),r.language)},tokenize:function(e,n){var t=n.rest;if(t){for(var r in t)n[r]=t[r];delete n.rest}var a=new s;return u(a,a.head,e),o(e,a,n,a.head,0),function(e){for(var n=[],t=e.head.next;t!==e.tail;)n.push(t.value),t=t.next;return n}(a)},hooks:{all:{},add:function(e,n){var t=a.hooks.all;t[e]=t[e]||[],t[e].push(n)},run:function(e,n){var t=a.hooks.all[e];if(t&&t.length)for(var r,i=0;r=t[i++];)r(n)}},Token:i};function i(e,n,t,r){this.type=e,this.content=n,this.alias=t,this.length=0|(r||"").length}function l(e,n,t,r){e.lastIndex=n;var a=e.exec(t);if(a&&r&&a[1]){var i=a[1].length;a.index+=i,a[0]=a[0].slice(i)}return a}function o(e,n,t,r,s,g){for(var f in t)if(t.hasOwnProperty(f)&&t[f]){var h=t[f];h=Array.isArray(h)?h:[h];for(var d=0;d<h.length;++d){if(g&&g.cause==f+","+d)return;var v=h[d],p=v.inside,m=!!v.lookbehind,y=!!v.greedy,k=v.alias;if(y&&!v.pattern.global){var x=v.pattern.toString().match(/[imsuy]*$/)[0];v.pattern=RegExp(v.pattern.source,x+"g")}for(var b=v.pattern||v,w=r.next,A=s;w!==n.tail&&!(g&&A>=g.reach);A+=w.value.length,w=w.next){var P=w.value;if(n.length>e.length)return;if(!(P instanceof i)){var E,S=1;if(y){if(!(E=l(b,A,e,m))||E.index>=e.length)break;var L=E.index,O=E.index+E[0].length,C=A;for(C+=w.value.length;L>=C;)C+=(w=w.next).value.length;if(A=C-=w.value.length,w.value instanceof i)continue;for(var j=w;j!==n.tail&&(C<O||"string"==typeof j.value);j=j.next)S++,C+=j.value.length;S--,P=e.slice(A,C),E.index-=A}else if(!(E=l(b,0,P,m)))continue;L=E.index;var N=E[0],_=P.slice(0,L),M=P.slice(L+N.length),W=A+P.length;g&&W>g.reach&&(g.reach=W);var I=w.prev;if(_&&(I=u(n,I,_),A+=_.length),c(n,I,S),w=u(n,I,new i(f,p?a.tokenize(N,p):N,k,N)),M&&u(n,w,M),S>1){var T={cause:f+","+d,reach:W};o(e,n,t,w.prev,A,T),g&&T.reach>g.reach&&(g.reach=T.reach)}}}}}}function s(){var e={value:null,prev:null,next:null},n={value:null,prev:e,next:null};e.next=n,this.head=e,this.tail=n,this.length=0}function u(e,n,t){var r=n.next,a={value:t,prev:n,next:r};return n.next=a,r.prev=a,e.length++,a}function c(e,n,t){for(var r=n.next,a=0;a<t&&r!==e.tail;a++)r=r.next;n.next=r,r.prev=n,e.length-=a}if(e.Prism=a,i.stringify=function e(n,t){if("string"==typeof n)return n;if(Array.isArray(n)){var r="";return n.forEach((function(n){r+=e(n,t)})),r}var i={type:n.type,content:e(n.content,t),tag:"span",classes:["token",n.type],attributes:{},language:t},l=n.alias;l&&(Array.isArray(l)?Array.prototype.push.apply(i.classes,l):i.classes.push(l)),a.hooks.run("wrap",i);var o="";for(var s in i.attributes)o+=" "+s+'="'+(i.attributes[s]||"").replace(/"/g,"&quot;")+'"';return"<"+i.tag+' class="'+i.classes.join(" ")+'"'+o+">"+i.content+"</"+i.tag+">"},!e.document)return e.addEventListener?(a.disableWorkerMessageHandler||e.addEventListener("message",(function(n){var t=JSON.parse(n.data),r=t.language,i=t.code,l=t.immediateClose;e.postMessage(a.highlight(i,a.languages[r],r)),l&&e.close()}),!1),a):a;var g=a.util.currentScript();function f(){a.manual||a.highlightAll()}if(g&&(a.filename=g.src,g.hasAttribute("data-manual")&&(a.manual=!0)),!a.manual){var h=document.readyState;"loading"===h||"interactive"===h&&g&&g.defer?document.addEventListener("DOMContentLoaded",f):window.requestAnimationFrame?window.requestAnimationFrame(f):window.setTimeout(f,16)}return a}(_self);"undefined"!=typeof module&&module.exports&&(module.exports=Prism),"undefined"!=typeof global&&(global.Prism=Prism);
Prism.languages.markup={comment:{pattern:/<!--(?:(?!<!--)[\s\S])*?-->/,greedy:!0},prolog:{pattern:/<\?[\s\S]+?\?>/,greedy:!0},doctype:{pattern:/<!DOCTYPE(?:[^>"'[\]]|"[^"]*"|'[^']*')+(?:\[(?:[^<"'\]]|"[^"]*"|'[^']*'|<(?!!--)|<!--(?:[^-]|-(?!->))*-->)*\]\s*)?>/i,greedy:!0,inside:{"internal-subset":{pattern:/(^[^\[]*\[)[\s\S]+(?=\]>$)/,lookbehind:!0,greedy:!0,inside:null},string:{pattern:/"[^"]*"|'[^']*'/,greedy:!0},punctuation:/^<!|>$|[[\]]/,"doctype-tag":/^DOCTYPE/i,name:/[^\s<>'"]+/}},cdata:{pattern:/<!\[CDATA\[[\s\S]*?\]\]>/i,greedy:!0},tag:{pattern:/<\/?(?!\d)[^\s>\/=$<%]+(?:\s(?:\s*[^\s>\/=]+(?:\s*=\s*(?:"[^"]*"|'[^']*'|[^\s'">=]+(?=[\s>]))|(?=[\s/>])))+)?\s*\/?>/,greedy:!0,inside:{tag:{pattern:/^<\/?[^\s>\/]+/,inside:{punctuation:/^<\/?/,namespace:/^[^\s>\/:]+:/}},"special-attr":[],"attr-value":{pattern:/=\s*(?:"[^"]*"|'[^']*'|[^\s'">=]+)/,inside:{punctuation:[{pattern:/^=/,alias:"attr-equals"},{pattern:/^(\s*)["']|["']$/,lookbehind:!0}]}},punctuation:/\/?>/,"attr-name":{pattern:/[^\s>\/]+/,inside:{namespace:/^[^\s>\/:]+:/}}}},entity:[{pattern:/&[\da-z]{1,8};/i,alias:"named-entity"},/&#x?[\da-f]{1,8};/i]},Prism.languages.markup.tag.inside["attr-value"].inside.entity=Prism.languages.markup.entity,Prism.languages.markup.doctype.inside["internal-subset"].inside=Prism.languages.markup,Prism.hooks.add("wrap",(function(a){"entity"===a.type&&(a.attributes.title=a.content.replace(/&amp;/,"&"))})),Object.defineProperty(Prism.languages.markup.tag,"addInlined",{value:function(a,e){var s={};s["language-"+e]={pattern:/(^<!\[CDATA\[)[\s\S]+?(?=\]\]>$)/i,lookbehind:!0,inside:Prism.languages[e]},s.cdata=/^<!\[CDATA\[|\]\]>$/i;var t={"included-cdata":{pattern:/<!\[CDATA\[[\s\S]*?\]\]>/i,inside:s}};t["language-"+e]={pattern:/[\s\S]+/,inside:Prism.languages[e]};var n={};n[a]={pattern:RegExp("(<__[^>]*>)(?:<!\\[CDATA\\[(?:[^\\]]|\\](?!\\]>))*\\]\\]>|(?!<!\\[CDATA\\[)[^])*?(?=</__>)".replace(/__/g,(function(){return a})),"i"),lookbehind:!0,greedy:!0,inside:t},Prism.languages.insertBefore("markup","cdata",n)}}),Object.defineProperty(Prism.languages.markup.tag,"addAttribute",{value:function(a,e){Prism.languages.markup.tag.inside["special-attr"].push({pattern:RegExp("(^|[\"'\\s])(?:"+a+")\\s*=\\s*(?:\"[^\"]*\"|'[^']*'|[^\\s'\">=]+(?=[\\s>]))","i"),lookbehind:!0,inside:{"attr-name":/^[^\s=]+/,"attr-value":{pattern:/=[\s\S]+/,inside:{value:{pattern:/(^=\s*(["']|(?!["'])))\S[\s\S]*(?=\2$)/,lookbehind:!0,alias:[e,"language-"+e],inside:Prism.languages[e]},punctuation:[{pattern:/^=/,alias:"attr-equals"},/"|'/]}}}})}}),Prism.languages.html=Prism.languages.markup,Prism.languages.mathml=Prism.languages.markup,Prism.languages.svg=Prism.languages.markup,Prism.languages.xml=Prism.languages.extend("markup",{}),Prism.languages.ssml=Prism.languages.xml,Prism.languages.atom=Prism.languages.xml,Prism.languages.rss=Prism.languages.xml;
!function(s){var e=/(?:"(?:\\(?:\r\n|[\s\S])|[^"\\\r\n])*"|'(?:\\(?:\r\n|[\s\S])|[^'\\\r\n])*')/;s.languages.css={comment:/\/\*[\s\S]*?\*\//,atrule:{pattern:RegExp("@[\\w-](?:[^;{\\s\"']|\\s+(?!\\s)|"+e.source+")*?(?:;|(?=\\s*\\{))"),inside:{rule:/^@[\w-]+/,"selector-function-argument":{pattern:/(\bselector\s*\(\s*(?![\s)]))(?:[^()\s]|\s+(?![\s)])|\((?:[^()]|\([^()]*\))*\))+(?=\s*\))/,lookbehind:!0,alias:"selector"},keyword:{pattern:/(^|[^\w-])(?:and|not|only|or)(?![\w-])/,lookbehind:!0}}},url:{pattern:RegExp("\\burl\\((?:"+e.source+"|(?:[^\\\\\r\n()\"']|\\\\[^])*)\\)","i"),greedy:!0,inside:{function:/^url/i,punctuation:/^\(|\)$/,string:{pattern:RegExp("^"+e.source+"$"),alias:"url"}}},selector:{pattern:RegExp("(^|[{}\\s])[^{}\\s](?:[^{};\"'\\s]|\\s+(?![\\s{])|"+e.source+")*(?=\\s*\\{)"),lookbehind:!0},string:{pattern:e,greedy:!0},property:{pattern:/(^|[^-\w\xA0-\uFFFF])(?!\s)[-_a-z\xA0-\uFFFF](?:(?!\s)[-\w\xA0-\uFFFF])*(?=\s*:)/i,lookbehind:!0},important:/!important\b/i,function:{pattern:/(^|[^-a-z0-9])[-a-z0-9]+(?=\()/i,lookbehind:!0},punctuation:/[(){};:,]/},s.languages.css.atrule.inside.rest=s.languages.css;var t=s.languages.markup;t&&(t.tag.addInlined("style","css"),t.tag.addAttribute("style","css"))}(Prism);
Prism.languages.clike={comment:[{pattern:/(^|[^\\])\/\*[\s\S]*?(?:\*\/|$)/,lookbehind:!0,greedy:!0},{pattern:/(^|[^\\:])\/\/.*/,lookbehind:!0,greedy:!0}],string:{pattern:/(["'])(?:\\(?:\r\n|[\s\S])|(?!\1)[^\\\r\n])*\1/,greedy:!0},"class-name":{pattern:/(\b(?:class|extends|implements|instanceof|interface|new|trait)\s+|\bcatch\s+\()[\w.\\]+/i,lookbehind:!0,inside:{punctuation:/[.\\]/}},keyword:/\b(?:break|catch|continue|do|else|finally|for|function|if|in|instanceof|new|null|return|throw|try|while)\b/,boolean:/\b(?:false|true)\b/,function:/\b\w+(?=\()/,number:/\b0x[\da-f]+\b|(?:\b\d+(?:\.\d*)?|\B\.\d+)(?:e[+-]?\d+)?/i,operator:/[<>]=?|[!=]=?=?|--?|\+\+?|&&?|\|\|?|[?*/~^%]/,punctuation:/[{}[\];(),.:]/};
Prism.languages.javascript=Prism.languages.extend("clike",{"class-name":[Prism.languages.clike["class-name"],{pattern:/(^|[^$\w\xA0-\uFFFF])(?!\s)[_$A-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*(?=\.(?:constructor|prototype))/,lookbehind:!0}],keyword:[{pattern:/((?:^|\})\s*)catch\b/,lookbehind:!0},{pattern:/(^|[^.]|\.\.\.\s*)\b(?:as|assert(?=\s*\{)|async(?=\s*(?:function\b|\(|[$\w\xA0-\uFFFF]|$))|await|break|case|class|const|continue|debugger|default|delete|do|else|enum|export|extends|finally(?=\s*(?:\{|$))|for|from(?=\s*(?:['"]|$))|function|(?:get|set)(?=\s*(?:[#\[$\w\xA0-\uFFFF]|$))|if|implements|import|in|instanceof|interface|let|new|null|of|package|private|protected|public|return|static|super|switch|this|throw|try|typeof|undefined|var|void|while|with|yield)\b/,lookbehind:!0}],function:/#?(?!\s)[_$a-zA-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*(?=\s*(?:\.\s*(?:apply|bind|call)\s*)?\()/,number:{pattern:RegExp("(^|[^\\w$])(?:NaN|Infinity|0[bB][01]+(?:_[01]+)*n?|0[oO][0-7]+(?:_[0-7]+)*n?|0[xX][\\dA-Fa-f]+(?:_[\\dA-Fa-f]+)*n?|\\d+(?:_\\d+)*n|(?:\\d+(?:_\\d+)*(?:\\.(?:\\d+(?:_\\d+)*)?)?|\\.\\d+(?:_\\d+)*)(?:[Ee][+-]?\\d+(?:_\\d+)*)?)(?![\\w$])"),lookbehind:!0},operator:/--|\+\+|\*\*=?|=>|&&=?|\|\|=?|[!=]==|<<=?|>>>?=?|[-+*/%&|^!=<>]=?|\.{3}|\?\?=?|\?\.?|[~:]/}),Prism.languages.javascript["class-name"][0].pattern=/(\b(?:class|extends|implements|instanceof|interface|new)\s+)[\w.\\]+/,Prism.languages.insertBefore("javascript","keyword",{regex:{pattern:RegExp("((?:^|[^$\\w\\xA0-\\uFFFF.\"'\\])\\s]|\\b(?:return|yield))\\s*)/(?:(?:\\[(?:[^\\]\\\\\r\n]|\\\\.)*\\]|\\\\.|[^/\\\\\\[\r\n])+/[dgimyus]{0,7}|(?:\\[(?:[^[\\]\\\\\r\n]|\\\\.|\\[(?:[^[\\]\\\\\r\n]|\\\\.|\\[(?:[^[\\]\\\\\r\n]|\\\\.)*\\])*\\])*\\]|\\\\.|[^/\\\\\\[\r\n])+/[dgimyus]{0,7}v[dgimyus]{0,7})(?=(?:\\s|/\\*(?:[^*]|\\*(?!/))*\\*/)*(?:$|[\r\n,.;:})\\]]|//))"),lookbehind:!0,greedy:!0,inside:{"regex-source":{pattern:/^(\/)[\s\S]+(?=\/[a-z]*$)/,lookbehind:!0,alias:"language-regex",inside:Prism.languages.regex},"regex-delimiter":/^\/|\/$/,"regex-flags":/^[a-z]+$/}},"function-variable":{pattern:/#?(?!\s)[_$a-zA-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*(?=\s*[=:]\s*(?:async\s*)?(?:\bfunction\b|(?:\((?:[^()]|\([^()]*\))*\)|(?!\s)[_$a-zA-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*)\s*=>))/,alias:"function"},parameter:[{pattern:/(function(?:\s+(?!\s)[_$a-zA-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*)?\s*\(\s*)(?!\s)(?:[^()\s]|\s+(?![\s)])|\([^()]*\))+(?=\s*\))/,lookbehind:!0,inside:Prism.languages.javascript},{pattern:/(^|[^$\w\xA0-\uFFFF])(?!\s)[_$a-z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*(?=\s*=>)/i,lookbehind:!0,inside:Prism.languages.javascript},{pattern:/(\(\s*)(?!\s)(?:[^()\s]|\s+(?![\s)])|\([^()]*\))+(?=\s*\)\s*=>)/,lookbehind:!0,inside:Prism.languages.javascript},{pattern:/((?:\b|\s|^)(?!(?:as|async|await|break|case|catch|class|const|continue|debugger|default|delete|do|else|enum|export|extends|finally|for|from|function|get|if|implements|import|in|instanceof|interface|let|new|null|of|package|private|protected|public|return|set|static|super|switch|this|throw|try|typeof|undefined|var|void|while|with|yield)(?![$\w\xA0-\uFFFF]))(?:(?!\s)[_$a-zA-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*\s*)\(\s*|\]\s*\(\s*)(?!\s)(?:[^()\s]|\s+(?![\s)])|\([^()]*\))+(?=\s*\)\s*\{)/,lookbehind:!0,inside:Prism.languages.javascript}],constant:/\b[A-Z](?:[A-Z_]|\dx?)*\b/}),Prism.languages.insertBefore("javascript","string",{hashbang:{pattern:/^#!.*/,greedy:!0,alias:"comment"},"template-string":{pattern:/`(?:\\[\s\S]|\$\{(?:[^{}]|\{(?:[^{}]|\{[^}]*\})*\})+\}|(?!\$\{)[^\\`])*`/,greedy:!0,inside:{"template-punctuation":{pattern:/^`|`$/,alias:"string"},interpolation:{pattern:/((?:^|[^\\])(?:\\{2})*)\$\{(?:[^{}]|\{(?:[^{}]|\{[^}]*\})*\})+\}/,lookbehind:!0,inside:{"interpolation-punctuation":{pattern:/^\$\{|\}$/,alias:"punctuation"},rest:Prism.languages.javascript}},string:/[\s\S]+/}},"string-property":{pattern:/((?:^|[,{])[ \t]*)(["'])(?:\\(?:\r\n|[\s\S])|(?!\2)[^\\\r\n])*\2(?=\s*:)/m,lookbehind:!0,greedy:!0,alias:"property"}}),Prism.languages.insertBefore("javascript","operator",{"literal-property":{pattern:/((?:^|[,{])[ \t]*)(?!\s)[_$a-zA-Z\xA0-\uFFFF](?:(?!\s)[$\w\xA0-\uFFFF])*(?=\s*:)/m,lookbehind:!0,alias:"property"}}),Prism.languages.markup&&(Prism.languages.markup.tag.addInlined("script","javascript"),Prism.languages.markup.tag.addAttribute("on(?:abort|blur|change|click|composition(?:end|start|update)|dblclick|error|focus(?:in|out)?|key(?:down|up)|load|mouse(?:down|enter|leave|move|out|over|up)|reset|resize|scroll|select|slotchange|submit|unload|wheel)","javascript")),Prism.languages.js=Prism.languages.javascript;
!function(e){var t="\\b(?:BASH|BASHOPTS|BASH_ALIASES|BASH_ARGC|BASH_ARGV|BASH_CMDS|BASH_COMPLETION_COMPAT_DIR|BASH_LINENO|BASH_REMATCH|BASH_SOURCE|BASH_VERSINFO|BASH_VERSION|COLORTERM|COLUMNS|COMP_WORDBREAKS|DBUS_SESSION_BUS_ADDRESS|DEFAULTS_PATH|DESKTOP_SESSION|DIRSTACK|DISPLAY|EUID|GDMSESSION|GDM_LANG|GNOME_KEYRING_CONTROL|GNOME_KEYRING_PID|GPG_AGENT_INFO|GROUPS|HISTCONTROL|HISTFILE|HISTFILESIZE|HISTSIZE|HOME|HOSTNAME|HOSTTYPE|IFS|INSTANCE|JOB|LANG|LANGUAGE|LC_ADDRESS|LC_ALL|LC_IDENTIFICATION|LC_MEASUREMENT|LC_MONETARY|LC_NAME|LC_NUMERIC|LC_PAPER|LC_TELEPHONE|LC_TIME|LESSCLOSE|LESSOPEN|LINES|LOGNAME|LS_COLORS|MACHTYPE|MAILCHECK|MANDATORY_PATH|NO_AT_BRIDGE|OLDPWD|OPTERR|OPTIND|ORBIT_SOCKETDIR|OSTYPE|PAPERSIZE|PATH|PIPESTATUS|PPID|PS1|PS2|PS3|PS4|PWD|RANDOM|REPLY|SECONDS|SELINUX_INIT|SESSION|SESSIONTYPE|SESSION_MANAGER|SHELL|SHELLOPTS|SHLVL|SSH_AUTH_SOCK|TERM|UID|UPSTART_EVENTS|UPSTART_INSTANCE|UPSTART_JOB|UPSTART_SESSION|USER|WINDOWID|XAUTHORITY|XDG_CONFIG_DIRS|XDG_CURRENT_DESKTOP|XDG_DATA_DIRS|XDG_GREETER_DATA_DIR|XDG_MENU_PREFIX|XDG_RUNTIME_DIR|XDG_SEAT|XDG_SEAT_PATH|XDG_SESSION_DESKTOP|XDG_SESSION_ID|XDG_SESSION_PATH|XDG_SESSION_TYPE|XDG_VTNR|XMODIFIERS)\\b",a={pattern:/(^(["']?)\w+\2)[ \t]+\S.*/,lookbehind:!0,alias:"punctuation",inside:null},n={bash:a,environment:{pattern:RegExp("\\$"+t),alias:"constant"},variable:[{pattern:/\$?\(\([\s\S]+?\)\)/,greedy:!0,inside:{variable:[{pattern:/(^\$\(\([\s\S]+)\)\)/,lookbehind:!0},/^\$\(\(/],number:/\b0x[\dA-Fa-f]+\b|(?:\b\d+(?:\.\d*)?|\B\.\d+)(?:[Ee]-?\d+)?/,operator:/--|\+\+|\*\*=?|<<=?|>>=?|&&|\|\||[=!+\-*/%<>^&|]=?|[?~:]/,punctuation:/\(\(?|\)\)?|,|;/}},{pattern:/\$\((?:\([^)]+\)|[^()])+\)|`[^`]+`/,greedy:!0,inside:{variable:/^\$\(|^`|\)$|`$/}},{pattern:/\$\{[^}]+\}/,greedy:!0,inside:{operator:/:[-=?+]?|[!\/]|##?|%%?|\^\^?|,,?/,punctuation:/[\[\]]/,environment:{pattern:RegExp("(\\{)"+t),lookbehind:!0,alias:"constant"}}},/\$(?:\w+|[#?*!@$])/],entity:/\\(?:[abceEfnrtv\\"]|O?[0-7]{1,3}|U[0-9a-fA-F]{8}|u[0-9a-fA-F]{4}|x[0-9a-fA-F]{1,2})/};e.languages.bash={shebang:{pattern:/^#!\s*\/.*/,alias:"important"},comment:{pattern:/(^|[^"{\\$])#.*/,lookbehind:!0},"function-name":[{pattern:/(\bfunction\s+)[\w-]+(?=(?:\s*\(?:\s*\))?\s*\{)/,lookbehind:!0,alias:"function"},{pattern:/\b[\w-]+(?=\s*\(\s*\)\s*\{)/,alias:"function"}],"for-or-select":{pattern:/(\b(?:for|select)\s+)\w+(?=\s+in\s)/,alias:"variable",lookbehind:!0},"assign-left":{pattern:/(^|[\s;|&]|[<>]\()\w+(?:\.\w+)*(?=\+?=)/,inside:{environment:{pattern:RegExp("(^|[\\s;|&]|[<>]\\()"+t),lookbehind:!0,alias:"constant"}},alias:"variable",lookbehind:!0},parameter:{pattern:/(^|\s)-{1,2}(?:\w+:[+-]?)?\w+(?:\.\w+)*(?=[=\s]|$)/,alias:"variable",lookbehind:!0},string:[{pattern:/((?:^|[^<])<<-?\s*)(\w+)\s[\s\S]*?(?:\r?\n|\r)\2/,lookbehind:!0,greedy:!0,inside:n},{pattern:/((?:^|[^<])<<-?\s*)(["'])(\w+)\2\s[\s\S]*?(?:\r?\n|\r)\3/,lookbehind:!0,greedy:!0,inside:{bash:a}},{pattern:/(^|[^\\](?:\\\\)*)"(?:\\[\s\S]|\$\([^)]+\)|\$(?!\()|`[^`]+`|[^"\\`$])*"/,lookbehind:!0,greedy:!0,inside:n},{pattern:/(^|[^$\\])'[^']*'/,lookbehind:!0,greedy:!0},{pattern:/\$'(?:[^'\\]|\\[\s\S])*'/,greedy:!0,inside:{entity:n.entity}}],environment:{pattern:RegExp("\\$?"+t),alias:"constant"},variable:n.variable,function:{pattern:/(^|[\s;|&]|[<>]\()(?:add|apropos|apt|apt-cache|apt-get|aptitude|aspell|automysqlbackup|awk|basename|bash|bc|bconsole|bg|bzip2|cal|cargo|cat|cfdisk|chgrp|chkconfig|chmod|chown|chroot|cksum|clear|cmp|column|comm|composer|cp|cron|crontab|csplit|curl|cut|date|dc|dd|ddrescue|debootstrap|df|diff|diff3|dig|dir|dircolors|dirname|dirs|dmesg|docker|docker-compose|du|egrep|eject|env|ethtool|expand|expect|expr|fdformat|fdisk|fg|fgrep|file|find|fmt|fold|format|free|fsck|ftp|fuser|gawk|git|gparted|grep|groupadd|groupdel|groupmod|groups|grub-mkconfig|gzip|halt|head|hg|history|host|hostname|htop|iconv|id|ifconfig|ifdown|ifup|import|install|ip|java|jobs|join|kill|killall|less|link|ln|locate|logname|logrotate|look|lpc|lpr|lprint|lprintd|lprintq|lprm|ls|lsof|lynx|make|man|mc|mdadm|mkconfig|mkdir|mke2fs|mkfifo|mkfs|mkisofs|mknod|mkswap|mmv|more|most|mount|mtools|mtr|mutt|mv|nano|nc|netstat|nice|nl|node|nohup|notify-send|npm|nslookup|op|open|parted|passwd|paste|pathchk|ping|pkill|pnpm|podman|podman-compose|popd|pr|printcap|printenv|ps|pushd|pv|quota|quotacheck|quotactl|ram|rar|rcp|reboot|remsync|rename|renice|rev|rm|rmdir|rpm|rsync|scp|screen|sdiff|sed|sendmail|seq|service|sftp|sh|shellcheck|shuf|shutdown|sleep|slocate|sort|split|ssh|stat|strace|su|sudo|sum|suspend|swapon|sync|sysctl|tac|tail|tar|tee|time|timeout|top|touch|tr|traceroute|tsort|tty|umount|uname|unexpand|uniq|units|unrar|unshar|unzip|update-grub|uptime|useradd|userdel|usermod|users|uudecode|uuencode|v|vcpkg|vdir|vi|vim|virsh|vmstat|wait|watch|wc|wget|whereis|which|who|whoami|write|xargs|xdg-open|yarn|yes|zenity|zip|zsh|zypper)(?=$|[)\s;|&])/,lookbehind:!0},keyword:{pattern:/(^|[\s;|&]|[<>]\()(?:case|do|done|elif|else|esac|fi|for|function|if|in|select|then|until|while)(?=$|[)\s;|&])/,lookbehind:!0},builtin:{pattern:/(^|[\s;|&]|[<>]\()(?:\.|:|alias|bind|break|builtin|caller|cd|command|continue|declare|echo|enable|eval|exec|exit|export|getopts|hash|help|let|local|logout|mapfile|printf|pwd|read|readarray|readonly|return|set|shift|shopt|source|test|times|trap|type|typeset|ulimit|umask|unalias|unset)(?=$|[)\s;|&])/,lookbehind:!0,alias:"class-name"},boolean:{pattern:/(^|[\s;|&]|[<>]\()(?:false|true)(?=$|[)\s;|&])/,lookbehind:!0},"file-descriptor":{pattern:/\B&\d\b/,alias:"important"},operator:{pattern:/\d?<>|>\||\+=|=[=~]?|!=?|<<[<-]?|[&\d]?>>|\d[<>]&?|[<>][&=]?|&[>&]?|\|[&|]?/,inside:{"file-descriptor":{pattern:/^\d/,alias:"important"}}},punctuation:/\$?\(\(?|\)\)?|\.\.|[{}[\];\\]/,number:{pattern:/(^|\s)(?:[1-9]\d*|0)(?:[.,]\d+)?\b/,lookbehind:!0}},a.inside=e.languages.bash;for(var s=["comment","function-name","for-or-select","assign-left","parameter","string","environment","function","keyword","builtin","boolean","file-descriptor","operator","punctuation","number"],o=n.variable[1].inside,i=0;i<s.length;i++)o[s[i]]=e.languages.bash[s[i]];e.languages.sh=e.languages.bash,e.languages.shell=e.languages.bash}(Prism);
Prism.languages.c=Prism.languages.extend("clike",{comment:{pattern:/\/\/(?:[^\r\n\\]|\\(?:\r\n?|\n|(?![\r\n])))*|\/\*[\s\S]*?(?:\*\/|$)/,greedy:!0},string:{pattern:/"(?:\\(?:\r\n|[\s\S])|[^"\\\r\n])*"/,greedy:!0},"class-name":{pattern:/(\b(?:enum|struct)\s+(?:__attribute__\s*\(\([\s\S]*?\)\)\s*)?)\w+|\b[a-z]\w*_t\b/,lookbehind:!0},keyword:/\b(?:_Alignas|_Alignof|_Atomic|_Bool|_Complex|_Generic|_Imaginary|_Noreturn|_Static_assert|_Thread_local|__attribute__|asm|auto|break|case|char|const|continue|default|do|double|else|enum|extern|float|for|goto|if|inline|int|long|register|return|short|signed|sizeof|static|struct|switch|typedef|typeof|union|unsigned|void|volatile|while)\b/,function:/\b[a-z_]\w*(?=\s*\()/i,number:/(?:\b0x(?:[\da-f]+(?:\.[\da-f]*)?|\.[\da-f]+)(?:p[+-]?\d+)?|(?:\b\d+(?:\.\d*)?|\B\.\d+)(?:e[+-]?\d+)?)[ful]{0,4}/i,operator:/>>=?|<<=?|->|([-+&|:])\1|[?:~]|[-+*/%&|^!=<>]=?/}),Prism.languages.insertBefore("c","string",{char:{pattern:/'(?:\\(?:\r\n|[\s\S])|[^'\\\r\n]){0,32}'/,greedy:!0}}),Prism.languages.insertBefore("c","string",{macro:{pattern:/(^[\t ]*)#\s*[a-z](?:[^\r\n\\/]|\/(?!\*)|\/\*(?:[^*]|\*(?!\/))*\*\/|\\(?:\r\n|[\s\S]))*/im,lookbehind:!0,greedy:!0,alias:"property",inside:{string:[{pattern:/^(#\s*include\s*)<[^>]+>/,lookbehind:!0},Prism.languages.c.string],char:Prism.languages.c.char,comment:Prism.languages.c.comment,"macro-name":[{pattern:/(^#\s*define\s+)\w+\b(?!\()/i,lookbehind:!0},{pattern:/(^#\s*define\s+)\w+\b(?=\()/i,lookbehind:!0,alias:"function"}],directive:{pattern:/^(#\s*)[a-z]+/,lookbehind:!0,alias:"keyword"},"directive-hash":/^#/,punctuation:/##|\\(?=[\r\n])/,expression:{pattern:/\S[\s\S]*/,inside:Prism.languages.c}}}}),Prism.languages.insertBefore("c","function",{constant:/\b(?:EOF|NULL|SEEK_CUR|SEEK_END|SEEK_SET|__DATE__|__FILE__|__LINE__|__TIMESTAMP__|__TIME__|__func__|stderr|stdin|stdout)\b/}),delete Prism.languages.c.boolean;
!function(e){var t=/\b(?:alignas|alignof|asm|auto|bool|break|case|catch|char|char16_t|char32_t|char8_t|class|co_await|co_return|co_yield|compl|concept|const|const_cast|consteval|constexpr|constinit|continue|decltype|default|delete|do|double|dynamic_cast|else|enum|explicit|export|extern|final|float|for|friend|goto|if|import|inline|int|int16_t|int32_t|int64_t|int8_t|long|module|mutable|namespace|new|noexcept|nullptr|operator|override|private|protected|public|register|reinterpret_cast|requires|return|short|signed|sizeof|static|static_assert|static_cast|struct|switch|template|this|thread_local|throw|try|typedef|typeid|typename|uint16_t|uint32_t|uint64_t|uint8_t|union|unsigned|using|virtual|void|volatile|wchar_t|while)\b/,n="\\b(?!<keyword>)\\w+(?:\\s*\\.\\s*\\w+)*\\b".replace(/<keyword>/g,(function(){return t.source}));e.languages.cpp=e.languages.extend("c",{"class-name":[{pattern:RegExp("(\\b(?:class|concept|enum|struct|typename)\\s+)(?!<keyword>)\\w+".replace(/<keyword>/g,(function(){return t.source}))),lookbehind:!0},/\b[A-Z]\w*(?=\s*::\s*\w+\s*\()/,/\b[A-Z_]\w*(?=\s*::\s*~\w+\s*\()/i,/\b\w+(?=\s*<(?:[^<>]|<(?:[^<>]|<[^<>]*>)*>)*>\s*::\s*\w+\s*\()/],keyword:t,number:{pattern:/(?:\b0b[01']+|\b0x(?:[\da-f']+(?:\.[\da-f']*)?|\.[\da-f']+)(?:p[+-]?[\d']+)?|(?:\b[\d']+(?:\.[\d']*)?|\B\.[\d']+)(?:e[+-]?[\d']+)?)[ful]{0,4}/i,greedy:!0},operator:/>>=?|<<=?|->|--|\+\+|&&|\|\||[?:~]|<=>|[-+*/%&|^!=<>]=?|\b(?:and|and_eq|bitand|bitor|not|not_eq|or|or_eq|xor|xor_eq)\b/,boolean:/\b(?:false|true)\b/}),e.languages.insertBefore("cpp","string",{module:{pattern:RegExp('(\\b(?:import|module)\\s+)(?:"(?:\\\\(?:\r\n|[^])|[^"\\\\\r\n])*"|<[^<>\r\n]*>|'+"<mod-name>(?:\\s*:\\s*<mod-name>)?|:\\s*<mod-name>".replace(/<mod-name>/g,(function(){return n}))+")"),lookbehind:!0,greedy:!0,inside:{string:/^[<"][\s\S]+/,operator:/:/,punctuation:/\./}},"raw-string":{pattern:/R"([^()\\ ]{0,16})\([\s\S]*?\)\1"/,alias:"string",greedy:!0}}),e.languages.insertBefore("cpp","keyword",{"generic-function":{pattern:/\b(?!operator\b)[a-z_]\w*\s*<(?:[^<>]|<[^<>]*>)*>(?=\s*\()/i,inside:{function:/^\w+/,generic:{pattern:/<[\s\S]+/,alias:"class-name",inside:e.languages.cpp}}}}),e.languages.insertBefore("cpp","operator",{"double-colon":{pattern:/::/,alias:"punctuation"}}),e.languages.insertBefore("cpp","class-name",{"base-clause":{pattern:/(\b(?:class|struct)\s+\w+\s*:\s*)[^;{}"'\s]+(?:\s+[^;{}"'\s]+)*(?=\s*[;{])/,lookbehind:!0,greedy:!0,inside:e.languages.extend("cpp",{})}}),e.languages.insertBefore("inside","double-colon",{"class-name":/\b[a-z_]\w*\b(?!\s*::)/i},e.languages.cpp["base-clause"])}(Prism);
!function(e){var n="(?:[ \t]+(?![ \t])(?:<SP_BS>)?|<SP_BS>)".replace(/<SP_BS>/g,(function(){return"\\\\[\r\n](?:\\s|\\\\[\r\n]|#.*(?!.))*(?![\\s#]|\\\\[\r\n])"})),r="\"(?:[^\"\\\\\r\n]|\\\\(?:\r\n|[^]))*\"|'(?:[^'\\\\\r\n]|\\\\(?:\r\n|[^]))*'",t="--[\\w-]+=(?:<STR>|(?![\"'])(?:[^\\s\\\\]|\\\\.)+)".replace(/<STR>/g,(function(){return r})),o={pattern:RegExp(r),greedy:!0},i={pattern:/(^[ \t]*)#.*/m,lookbehind:!0,greedy:!0};function a(e,r){return e=e.replace(/<OPT>/g,(function(){return t})).replace(/<SP>/g,(function(){return n})),RegExp(e,r)}e.languages.docker={instruction:{pattern:/(^[ \t]*)(?:ADD|ARG|CMD|COPY|ENTRYPOINT|ENV|EXPOSE|FROM|HEALTHCHECK|LABEL|MAINTAINER|ONBUILD|RUN|SHELL|STOPSIGNAL|USER|VOLUME|WORKDIR)(?=\s)(?:\\.|[^\r\n\\])*(?:\\$(?:\s|#.*$)*(?![\s#])(?:\\.|[^\r\n\\])*)*/im,lookbehind:!0,greedy:!0,inside:{options:{pattern:a("(^(?:ONBUILD<SP>)?\\w+<SP>)<OPT>(?:<SP><OPT>)*","i"),lookbehind:!0,greedy:!0,inside:{property:{pattern:/(^|\s)--[\w-]+/,lookbehind:!0},string:[o,{pattern:/(=)(?!["'])(?:[^\s\\]|\\.)+/,lookbehind:!0}],operator:/\\$/m,punctuation:/=/}},keyword:[{pattern:a("(^(?:ONBUILD<SP>)?HEALTHCHECK<SP>(?:<OPT><SP>)*)(?:CMD|NONE)\\b","i"),lookbehind:!0,greedy:!0},{pattern:a("(^(?:ONBUILD<SP>)?FROM<SP>(?:<OPT><SP>)*(?!--)[^ \t\\\\]+<SP>)AS","i"),lookbehind:!0,greedy:!0},{pattern:a("(^ONBUILD<SP>)\\w+","i"),lookbehind:!0,greedy:!0},{pattern:/^\w+/,greedy:!0}],comment:i,string:o,variable:/\$(?:\w+|\{[^{}"'\\]*\})/,operator:/\\$/m}},comment:i},e.languages.dockerfile=e.languages.docker}(Prism);
!function(e){function n(e,n){return"___"+e.toUpperCase()+n+"___"}Object.defineProperties(e.languages["markup-templating"]={},{buildPlaceholders:{value:function(t,a,r,o){if(t.language===a){var c=t.tokenStack=[];t.code=t.code.replace(r,(function(e){if("function"==typeof o&&!o(e))return e;for(var r,i=c.length;-1!==t.code.indexOf(r=n(a,i));)++i;return c[i]=e,r})),t.grammar=e.languages.markup}}},tokenizePlaceholders:{value:function(t,a){if(t.language===a&&t.tokenStack){t.grammar=e.languages[a];var r=0,o=Object.keys(t.tokenStack);!function c(i){for(var u=0;u<i.length&&!(r>=o.length);u++){var g=i[u];if("string"==typeof g||g.content&&"string"==typeof g.content){var l=o[r],s=t.tokenStack[l],f="string"==typeof g?g:g.content,p=n(a,l),k=f.indexOf(p);if(k>-1){++r;var m=f.substring(0,k),d=new e.Token(a,e.tokenize(s,t.grammar),"language-"+a,s),h=f.substring(k+p.length),v=[];m&&v.push.apply(v,c([m])),v.push(d),h&&v.push.apply(v,c([h])),"string"==typeof g?i.splice.apply(i,[u,1].concat(v)):g.content=v}}else g.content&&c(g.content)}return i}(t.tokens)}}}})}(Prism);
!function(e){var n=/\$(?:\w[a-z\d]*(?:_[^\x00-\x1F\s"'\\()$]*)?|\{[^}\s"'\\]+\})/i;e.languages.nginx={comment:{pattern:/(^|[\s{};])#.*/,lookbehind:!0,greedy:!0},directive:{pattern:/(^|\s)\w(?:[^;{}"'\\\s]|\\.|"(?:[^"\\]|\\.)*"|'(?:[^'\\]|\\.)*'|\s+(?:#.*(?!.)|(?![#\s])))*?(?=\s*[;{])/,lookbehind:!0,greedy:!0,inside:{string:{pattern:/((?:^|[^\\])(?:\\\\)*)(?:"(?:[^"\\]|\\.)*"|'(?:[^'\\]|\\.)*')/,lookbehind:!0,greedy:!0,inside:{escape:{pattern:/\\["'\\nrt]/,alias:"entity"},variable:n}},comment:{pattern:/(\s)#.*/,lookbehind:!0,greedy:!0},keyword:{pattern:/^\S+/,greedy:!0},boolean:{pattern:/(\s)(?:off|on)(?!\S)/,lookbehind:!0},number:{pattern:/(\s)\d+[a-z]*(?!\S)/i,lookbehind:!0},variable:n}},punctuation:/[{};]/}}(Prism);
!function(e){var a=/\/\*[\s\S]*?\*\/|\/\/.*|#(?!\[).*/,t=[{pattern:/\b(?:false|true)\b/i,alias:"boolean"},{pattern:/(::\s*)\b[a-z_]\w*\b(?!\s*\()/i,greedy:!0,lookbehind:!0},{pattern:/(\b(?:case|const)\s+)\b[a-z_]\w*(?=\s*[;=])/i,greedy:!0,lookbehind:!0},/\b(?:null)\b/i,/\b[A-Z_][A-Z0-9_]*\b(?!\s*\()/],i=/\b0b[01]+(?:_[01]+)*\b|\b0o[0-7]+(?:_[0-7]+)*\b|\b0x[\da-f]+(?:_[\da-f]+)*\b|(?:\b\d+(?:_\d+)*\.?(?:\d+(?:_\d+)*)?|\B\.\d+)(?:e[+-]?\d+)?/i,n=/<?=>|\?\?=?|\.{3}|\??->|[!=]=?=?|::|\*\*=?|--|\+\+|&&|\|\||<<|>>|[?~]|[/^|%*&<>.+-]=?/,s=/[{}\[\](),:;]/;e.languages.php={delimiter:{pattern:/\?>$|^<\?(?:php(?=\s)|=)?/i,alias:"important"},comment:a,variable:/\$+(?:\w+\b|(?=\{))/,package:{pattern:/(namespace\s+|use\s+(?:function\s+)?)(?:\\?\b[a-z_]\w*)+\b(?!\\)/i,lookbehind:!0,inside:{punctuation:/\\/}},"class-name-definition":{pattern:/(\b(?:class|enum|interface|trait)\s+)\b[a-z_]\w*(?!\\)\b/i,lookbehind:!0,alias:"class-name"},"function-definition":{pattern:/(\bfunction\s+)[a-z_]\w*(?=\s*\()/i,lookbehind:!0,alias:"function"},keyword:[{pattern:/(\(\s*)\b(?:array|bool|boolean|float|int|integer|object|string)\b(?=\s*\))/i,alias:"type-casting",greedy:!0,lookbehind:!0},{pattern:/([(,?]\s*)\b(?:array(?!\s*\()|bool|callable|(?:false|null)(?=\s*\|)|float|int|iterable|mixed|object|self|static|string)\b(?=\s*\$)/i,alias:"type-hint",greedy:!0,lookbehind:!0},{pattern:/(\)\s*:\s*(?:\?\s*)?)\b(?:array(?!\s*\()|bool|callable|(?:false|null)(?=\s*\|)|float|int|iterable|mixed|never|object|self|static|string|void)\b/i,alias:"return-type",greedy:!0,lookbehind:!0},{pattern:/\b(?:array(?!\s*\()|bool|float|int|iterable|mixed|object|string|void)\b/i,alias:"type-declaration",greedy:!0},{pattern:/(\|\s*)(?:false|null)\b|\b(?:false|null)(?=\s*\|)/i,alias:"type-declaration",greedy:!0,lookbehind:!0},{pattern:/\b(?:parent|self|static)(?=\s*::)/i,alias:"static-context",greedy:!0},{pattern:/(\byield\s+)from\b/i,lookbehind:!0},/\bclass\b/i,{pattern:/((?:^|[^\s>:]|(?:^|[^-])>|(?:^|[^:]):)\s*)\b(?:abstract|and|array|as|break|callable|case|catch|clone|const|continue|declare|default|die|do|echo|else|elseif|empty|enddeclare|endfor|endforeach|endif|endswitch|endwhile|enum|eval|exit|extends|final|finally|fn|for|foreach|function|global|goto|if|implements|include|include_once|instanceof|insteadof|interface|isset|list|match|namespace|never|new|or|parent|print|private|protected|public|readonly|require|require_once|return|self|static|switch|throw|trait|try|unset|use|var|while|xor|yield|__halt_compiler)\b/i,lookbehind:!0}],"argument-name":{pattern:/([(,]\s*)\b[a-z_]\w*(?=\s*:(?!:))/i,lookbehind:!0},"class-name":[{pattern:/(\b(?:extends|implements|instanceof|new(?!\s+self|\s+static))\s+|\bcatch\s*\()\b[a-z_]\w*(?!\\)\b/i,greedy:!0,lookbehind:!0},{pattern:/(\|\s*)\b[a-z_]\w*(?!\\)\b/i,greedy:!0,lookbehind:!0},{pattern:/\b[a-z_]\w*(?!\\)\b(?=\s*\|)/i,greedy:!0},{pattern:/(\|\s*)(?:\\?\b[a-z_]\w*)+\b/i,alias:"class-name-fully-qualified",greedy:!0,lookbehind:!0,inside:{punctuation:/\\/}},{pattern:/(?:\\?\b[a-z_]\w*)+\b(?=\s*\|)/i,alias:"class-name-fully-qualified",greedy:!0,inside:{punctuation:/\\/}},{pattern:/(\b(?:extends|implements|instanceof|new(?!\s+self\b|\s+static\b))\s+|\bcatch\s*\()(?:\\?\b[a-z_]\w*)+\b(?!\\)/i,alias:"class-name-fully-qualified",greedy:!0,lookbehind:!0,inside:{punctuation:/\\/}},{pattern:/\b[a-z_]\w*(?=\s*\$)/i,alias:"type-declaration",greedy:!0},{pattern:/(?:\\?\b[a-z_]\w*)+(?=\s*\$)/i,alias:["class-name-fully-qualified","type-declaration"],greedy:!0,inside:{punctuation:/\\/}},{pattern:/\b[a-z_]\w*(?=\s*::)/i,alias:"static-context",greedy:!0},{pattern:/(?:\\?\b[a-z_]\w*)+(?=\s*::)/i,alias:["class-name-fully-qualified","static-context"],greedy:!0,inside:{punctuation:/\\/}},{pattern:/([(,?]\s*)[a-z_]\w*(?=\s*\$)/i,alias:"type-hint",greedy:!0,lookbehind:!0},{pattern:/([(,?]\s*)(?:\\?\b[a-z_]\w*)+(?=\s*\$)/i,alias:["class-name-fully-qualified","type-hint"],greedy:!0,lookbehind:!0,inside:{punctuation:/\\/}},{pattern:/(\)\s*:\s*(?:\?\s*)?)\b[a-z_]\w*(?!\\)\b/i,alias:"return-type",greedy:!0,lookbehind:!0},{pattern:/(\)\s*:\s*(?:\?\s*)?)(?:\\?\b[a-z_]\w*)+\b(?!\\)/i,alias:["class-name-fully-qualified","return-type"],greedy:!0,lookbehind:!0,inside:{punctuation:/\\/}}],constant:t,function:{pattern:/(^|[^\\\w])\\?[a-z_](?:[\w\\]*\w)?(?=\s*\()/i,lookbehind:!0,inside:{punctuation:/\\/}},property:{pattern:/(->\s*)\w+/,lookbehind:!0},number:i,operator:n,punctuation:s};var l={pattern:/\{\$(?:\{(?:\{[^{}]+\}|[^{}]+)\}|[^{}])+\}|(^|[^\\{])\$+(?:\w+(?:\[[^\r\n\[\]]+\]|->\w+)?)/,lookbehind:!0,inside:e.languages.php},r=[{pattern:/<<<'([^']+)'[\r\n](?:.*[\r\n])*?\1;/,alias:"nowdoc-string",greedy:!0,inside:{delimiter:{pattern:/^<<<'[^']+'|[a-z_]\w*;$/i,alias:"symbol",inside:{punctuation:/^<<<'?|[';]$/}}}},{pattern:/<<<(?:"([^"]+)"[\r\n](?:.*[\r\n])*?\1;|([a-z_]\w*)[\r\n](?:.*[\r\n])*?\2;)/i,alias:"heredoc-string",greedy:!0,inside:{delimiter:{pattern:/^<<<(?:"[^"]+"|[a-z_]\w*)|[a-z_]\w*;$/i,alias:"symbol",inside:{punctuation:/^<<<"?|[";]$/}},interpolation:l}},{pattern:/`(?:\\[\s\S]|[^\\`])*`/,alias:"backtick-quoted-string",greedy:!0},{pattern:/'(?:\\[\s\S]|[^\\'])*'/,alias:"single-quoted-string",greedy:!0},{pattern:/"(?:\\[\s\S]|[^\\"])*"/,alias:"double-quoted-string",greedy:!0,inside:{interpolation:l}}];e.languages.insertBefore("php","variable",{string:r,attribute:{pattern:/#\[(?:[^"'\/#]|\/(?![*/])|\/\/.*$|#(?!\[).*$|\/\*(?:[^*]|\*(?!\/))*\*\/|"(?:\\[\s\S]|[^\\"])*"|'(?:\\[\s\S]|[^\\'])*')+\](?=\s*[a-z$#])/im,greedy:!0,inside:{"attribute-content":{pattern:/^(#\[)[\s\S]+(?=\]$)/,lookbehind:!0,inside:{comment:a,string:r,"attribute-class-name":[{pattern:/([^:]|^)\b[a-z_]\w*(?!\\)\b/i,alias:"class-name",greedy:!0,lookbehind:!0},{pattern:/([^:]|^)(?:\\?\b[a-z_]\w*)+/i,alias:["class-name","class-name-fully-qualified"],greedy:!0,lookbehind:!0,inside:{punctuation:/\\/}}],constant:t,number:i,operator:n,punctuation:s}},delimiter:{pattern:/^#\[|\]$/,alias:"punctuation"}}}}),e.hooks.add("before-tokenize",(function(a){/<\?/.test(a.code)&&e.languages["markup-templating"].buildPlaceholders(a,"php",/<\?(?:[^"'/#]|\/(?![*/])|("|')(?:\\[\s\S]|(?!\1)[^\\])*\1|(?:\/\/|#(?!\[))(?:[^?\n\r]|\?(?!>))*(?=$|\?>|[\r\n])|#\[|\/\*(?:[^*]|\*(?!\/))*(?:\*\/|$))*?(?:\?>|$)/g)})),e.hooks.add("after-tokenize",(function(a){e.languages["markup-templating"].tokenizePlaceholders(a,"php")}))}(Prism);
```

