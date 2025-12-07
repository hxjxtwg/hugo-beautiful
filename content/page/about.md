---
title: "关于我"
date: 2025-12-05
layout: "standard" # 使用标准布局

# 👇 关键：隐藏左侧边栏，让内容居中显示，更像个人主页
sidebar:
  enable: false

# 👇 关键：自动把它加到顶部导航栏
menu:
  main:
    weight: 50 # 数字越大越靠右
    params: 
      icon: user # 给菜单加个小图标 (可选)

# 👇 如果不想让关于页面显示评论，设为 false
comments: false 
---

###  hello xxsky

记录生活
专注世界

---

{{< callout type="info" emoji="🛠️" >}}
**我的技术栈：**
* **前端：** HTML, CSS, Hugo
* **后端：** Python, Go
{{< /callout >}}

---

###  我的经历

-------
{{< steps >}}
2025.12建立了hugo站点-----------------------------------------------------------------

2025.08建立了tpyecheo-----------------------------------------------------------------

2025.06建立了hexo博客-----------------------------------------------------------------
-------

{{< /steps >}}

---

###  联系我

{{< cards >}}
  {{< card link="https://github.com/xuhxjx" title="GitHub" icon="github" >}}
  {{< card link="mailto:hxjxtw@gmail.com" title="Email" icon="mail" >}}
{{< /cards >}}