---

title: "Hugo建站、主题修改与Cloudflare部署全流程手册"

author: "xxsky"

type: "posts"

date: 2025-12-07T16:49:26+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

beautifulhugo主题配置、修改以及部署过程

<!--more-->
### 1. 安装Hugo
1.1 macOS (推荐使用 Homebrew):
```bash
brew install hugo
```
1.2 Windows (推荐使用 Chocolatey 或 Scoop):
```bash
choco install hugo -confirm
# 或者下载官方 exe 文件配置环境变量
```
1.3 Linux:
```bash
sudo apt-get install hugo
```
1.4 验证安装
```bash
hugo version
# 输出示例: hugo v0.130.0 ...
```
### 2. 创建项目
2.1 创建新站点
```bash
hugo new hugo_simple
```
2.2 进入目录
```bash
cd hugo_simple
```
2.3 初始化 Git 仓库（部署必须）
```bash
git init
```
### 3. 安装Beautiful Hugo主题
3.1 使用Git Submodule方式安装，这样方便后续更新：
```bash
git submodule add https://github.com/halogenica/beautifulhugo.git themes/beautifulhugo
```
3.2 复制默认配置
将主题自带的示例配置复制出来作为基础：
```bash
cp themes/beautifulhugo/exampleSite/config.toml .
```
### 4. 配置修改
hugo.toml内容：
```bash
# -----------------------------------------------------------
# 基础配置 (Basic Configuration)
# -----------------------------------------------------------

# 你的网站完整域名。在本地运行(hugo server)时这里不起作用，但部署到 GitHub Pages 时必须填对
baseurl = "" 

# 网站默认语言。如果你主要写中文，建议改成 "zh-cn"
DefaultContentLanguage = "zh-cn" 
languageCode = "zh-cn"

# 网站标题，会显示在浏览器标签页和网页头部
title = "Beautiful Hugo" 

# 主题名称，必须和你 themes 文件夹下的名字完全一致
theme = "beautifulhugo" 

# 如果你是用 Hugo Module 方式安装的，取消下面这行的注释（新手一般用上面那个）
# theme = "github.com/halogenica/beautifulhugo"

# -----------------------------------------------------------
# 代码高亮设置 (Code Highlighting)
# -----------------------------------------------------------
pygmentsStyle = "trac"                 # 代码高亮的配色风格，可选值很多，如 monokai, dracula 等
pygmentsUseClasses = true              # 推荐 true，生成 CSS 类而不是内联样式，方便自定义
pygmentsCodeFences = true              # 启用 Markdown 的 ``` 代码块语法
pygmentsCodefencesGuessSyntax = true   # 自动猜测代码语言（如果你没写 ```python 这种标记）
#pygmentsUseClassic = true             # 是否使用旧版高亮引擎（一般不用开）
#pygmentOptions = "linenos=inline"     # 是否显示行号，去掉注释即可开启

# -----------------------------------------------------------
# 第三方服务 (Services) - 如果不需要可以留空
# -----------------------------------------------------------
[Services]
  [Services.Disqus]
    # Shortname = "XXX"      # 你的 Disqus 评论系统 ID，填了就会在文章下显示评论框
  [Services.googleAnalytics]
    # id = "XXX"             # 你的 Google Analytics 统计 ID (G-xxxxxx)

# -----------------------------------------------------------
# 主题特定参数 (Theme Params)
# -----------------------------------------------------------
[Params]
#  homeTitle = "Beautiful Hugo"  # 首页的大标题。如果不填，默认使用上面的 title 变量
  subtitle = "beautiful and simple website" # 副标题，显示在标题下方
  
  mainSections = ["post","posts"]      # 指定哪些文件夹里的内容算作“博客文章”
  
  logo = "img/avatar.png"         # 导航栏左上角的头像/Logo。文件需放在 static/img/ 目录下
  favicon = "img/favicon.svg"          # 浏览器标签页的小图标。文件需放在 static/img/ 目录下
  
  # 日期格式。注意：Go语言必须用 2006-01-02 代表 YYYY-MM-DD，不能乱改数字
  dateFormat = "January 2, 2006"       
  
  commit = false                       # 是否在文章底部显示 Git 提交信息（一般不开）
  rss = true                           # 是否生成 RSS 订阅源
  comments = true                      # 是否全局开启评论功能
  readingTime = true                   # 是否显示“阅读需 X 分钟”
  wordCount = true                     # 是否显示文章字数
  useHLJS = true                       # 是否使用 highlight.js (前端高亮库)，建议开启
  socialShare = true                   # 是否在文章底部显示分享按钮（Twitter, FB等）
  delayDisqus = true                   # 延迟加载 Disqus 评论，能加快网页打开速度
  showRelatedPosts = true              # 是否在文章底部显示“相关文章”推荐
  
#  cusdisID = "XXX"                    # 另一种评论系统 Cusdis 的 ID
#  hideAuthor = true                   # 是否隐藏作者名字
#  gcse = "0123..."                    # Google 自定义搜索 ID，填了会把搜索框变成 Google 搜索
#  disclaimerText = "..."              # 页脚的免责声明文字

# -----------------------------------------------------------
# 首页大图轮播 (Big Image / Hero Image)
# -----------------------------------------------------------
# 下面这些是首页顶部的大图设置。如果不想要大图，就把这几行都删掉或者注释掉

#[[Params.bigimg]]
#  src = "img/triangle.jpg"  # 图片路径 (static/img/...)
#  desc = "Triangle"         # 图片描述
#[[Params.bigimg]]
#  src = "img/sphere.jpg"
#  desc = "Sphere"
#  position = "center top"   # CSS 定位参数
#[[Params.bigimg]]
#  src = "img/hexagon.jpg"
#  desc = "Hexagon"

# -----------------------------------------------------------
# 作者社交链接 (Author Social Links)
# -----------------------------------------------------------
[Params.author]
  name = "xxsky"       # 你的名字（用于页脚版权声明）
  website = "https://nav.hxjx.hidns.co" # 你的个人主页链接
  email = "xuhxjxhk@gmail.com" # 你的邮箱，填了会在页脚显示邮箱图标
  
  # 下面这些社交账号，有哪个填哪个，没有的直接留空或删除即可，图标会自动显示
  facebook = "username"
  github = "username"        # 比如填 halogenica，不要填完整 URL
  # gitlab = "username"
  # bitbucket = "username"
  # twitter = "username"
  # reddit = "username"
  # linkedin = "username"
  # stackoverflow = "users/XXXXXXX/username"
  # instagram = "username"
  # youtube = "user/username" 
  # weibo = "username"       # 微博
  # zhihu = "username"       # 知乎 (部分版本支持)

# -----------------------------------------------------------
# 菜单栏设置 (Menu)
# -----------------------------------------------------------

# 1. 普通菜单链接
[[menu.main]]
    name = "Blog"            # 菜单显示的文字
    url = ""                 # 点击跳转的链接，"" 代表首页
    weight = 1               # 排序权重，数字越小越靠左

[[menu.main]]
    name = "About"
    url = "page/about/"      # 对应 content/page/about.md
    weight = 3

# 2. 下拉菜单示例 (父级)
[[menu.main]]
    identifier = "samples"   # 唯一标识符，用于被子菜单引用
    name = "Samples"         # 显示文字
    weight = 2

# 2.1 下拉菜单 (子级)
[[menu.main]]
    parent = "samples"       # 必须填上面的 identifier
    name = "Big Image Sample"
    url = "post/2017-03-07-bigimg-sample"
    weight = 1

[[menu.main]]
    parent = "samples"
    name = "Math Sample"
    url = "post/2017-03-05-math-sample"
    weight = 2

[[menu.main]]
    parent = "samples"
    name = "Code Sample"
    url = "post/2016-03-08-code-sample"
    weight = 3

# 3. 标签云页面链接
[[menu.main]]
    name = "Tags"
    url = "tags"
    weight = 3
```
### 5. 主题模板核心修复 (代码修改)
5.1 修复post_preview.html(防止内容泄露)
解决 “首页错误显示第一篇文章完整内容” 的 Bug

文件位置: themes/beautifulhugo/layouts/partials/post_preview.html

操作: 找到 post-entry 部分，将错误的 .Content 替换为 .Summary。
修改后的代码片段：
```html
<div class="post-entry">
    {{ if or (.Truncated) (.Params.summary) }}
        {{ .Summary }}
        <a href="{{ .Permalink }}" class="post-read-more">[{{ i18n "readMore" }}]</a>
    {{ else }}
        {{ .Summary }}  {{ end }}
</div>
```

5.2. 修复 index.html (首页安全过滤)
* 文件位置: layouts/index.html (如果没有，请从 themes/beautifulhugo/layouts/index.html 复制一份到项目根目录的 layouts 文件夹)

* 操作: 在循环中增加 .IsHome 判断

修改后的代码片段：
```html
<div class="posts-list">
  {{ $pag := .Paginate (where site.RegularPages "Type" "in" site.Params.mainSections) }}
  {{ range $pag.Pages }}
    {{/* 确保跳过被错误识别为页面的首页索引 */}}
    {{ if not .IsHome }}
      {{ partial "post_preview" . }}
    {{ end }}
  {{ end }}
</div>
```
5.3 自定义页脚版权 (可选)
* 文件位置: themes/beautifulhugo/layouts/partials/footer.html

* 操作: 搜索 copyright 修改版权文字，搜索 poweredBy 删除或修改“由 Hugo 驱动”的字样

5.4 修改版权与驱动信息
目的： 去掉 "Powered by Hugo" 和 "Theme by Beautiful Hugo"。 操作： 复制主题文件到 layouts/partials/footer.html，然后修改：
```html
<p class="credits theme-by text-muted">
  {{ i18n "poweredBy" . | safeHTML }}
  {{ if $.GitInfo }}...{{ end }}
</p>

<p class="credits theme-by text-muted">
  Copyright &copy; {{ now.Format "2006" }} {{ .Site.Params.author.name }}
</p>
```
也可以直接删掉p段

5.5 修改文章元信息
目的： 文章标题下不显示“By [作者名]”，只显示日期。 操作： 复制主题文件到 layouts/partials/post_meta.html，然后修改：
```bash
{{ if .Params.author }}
  {{ i18n "postedBy" }} <span class="post-meta-user">{{ .Params.author }}</span>
{{ end }}
```
5.6 网站头像
位置：layouts/partials/nav.html
去掉头像圆圈，替换成下面字段
```bash
<a class="navbar-brand" href="{{ "" | absLangURL }}">{{ .Site.Title }}</a>
```
或强制指向首页
```bash
<a class="navbar-brand" href="/">
```
5.7 日期显示
位置：layouts/partials/post_meta.html
```bash
  {{ $lastmodstr := .Lastmod | time.Format ":2006-1-2" }}
  {{ $datestr := .Date | time.Format ":2006-1-2" }}
```
或文章标题下不显示作者名字，删除作者判断逻辑
```bash
{{ if .Params.author }}
  {{ i18n "postedBy" }} <span class="post-meta-user">{{ .Params.author }}</span>
{{ end }}
```

### 6. 创建内容与本地测试
6.1 创建文章
```bash
hugo new posts/hello-world.md
```
6.2 本地预览
```bash
hugo server
```
访问 http://localhost:1313，确保首页显示正常（只有标题和摘要，没有全文）

### 7. 推送到 GitHub 并部署 Cloudflare Pages
7.1 提交代码到 GitHub
在项目根目录下：
```bash
# 1. 创建 .gitignore 文件（如果没有）
echo "public" >> .gitignore
echo "resources" >> .gitignore

# 2. 提交所有更改
git add .
git commit -m "完成站点配置与Bug修复"

# 3. 推送到远程仓库 (假设您已在 GitHub 创建了仓库)
git branch -M main
git remote add origin https://github.com/您的用户名/您的仓库名.git
git push -u origin main
```
7.2 Cloudflare Pages 部署设置
* 登录 Cloudflare Dash: 进入 Workers & Pages -> Create Application -> Connect to Git。

* 选择仓库: 选中刚才推送的 my-blog 仓库。

* 构建配置 (Build Settings):

  * Framework preset: 选择 Hugo。

  * Build command: 保持 hugo。

  * Build output directory: 保持 public。

* 开始部署: 点击 Save and Deploy。

### 8. 后续维护
* 更新文章: 在本地 hugo new post/xxx.md，写完后 git add . -> git commit -> git push，Cloudflare 会自动触发更新。

* 排错: 如果部署后样式错乱或不更新，去 Cloudflare 页面点击 Purge Cache (清除缓存)。