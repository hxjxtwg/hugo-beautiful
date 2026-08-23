---

title: "Danmu Api"

author: "xxsky"

type: "posts"

date: 2026-08-23T11:11:44+08:00

subtitle: ""

image: ""

tags:
  - 技术
  - 学习

---

修改弹幕支持版本标签识别

<!--more-->

一、LogVar弹幕
[API项目地址](https://github.com/huangxd-/danmu_api "danmu_api")

二、修改方法

1.路径

/danmu_api/apis/dandan-api.js

2.针对手动搜索接口 (searchAnimeBody)

* 在文件里使用 Ctrl + F 搜索 async function searchAnimeBody（大概在代码的第 432 行左右）。

* 找到获取 queryTitle 的那一行。

* 在它正下方插入我们自定义的过滤规则，修改后如下：

```
async function searchAnimeBody(url, preferAnimeId = null, preferSource = null, detailStore = null, targetPlatform = null, forceRefresh = false) {
  let queryTitle = url.searchParams.get("keyword");

  // 【在此处插入这3行】过滤你自定义的画质/帧率后缀
  if (queryTitle) {
    queryTitle = queryTitle.replace(/\(DV\)|\(HQ\)|\(HDR\)|\(HFR\)|\(SDR\)|\(4K\)|\(2K\)/gi, '').trim();
  }

  // 搜索词杂音清理：移除画质/配音/版本等杂音词后再提交源站搜索
  if (globals.titleNoiseFilter) {
    queryTitle = queryTitle.replace(globals.titleNoiseFilter, '').trim();
  }
```
3.针对自动匹配接口 (normalizeMatchTitle) —— 最关键的一步

Emby 播放器自动加载弹幕时，走的是底层的 Match 接口。

* 在文件里使用 Ctrl + F 搜索 function normalizeMatchTitle（大概在代码的第 1437 行左右）。

* 找到 let normalized 赋值的地方。

* 在其下方插入正则替换，修改后如下：

```
function normalizeMatchTitle(title) {
  let normalized = String(title || '').trim();

  // 【在此处插入这1行】过滤自定义后缀，提纯剧名给 API
  normalized = normalized.replace(/\(DV\)|\(HQ\)|\(HDR\)|\(HFR\)|\(SDR\)|\(4K\)|\(2K\)/gi, '').trim();

  if (globals.animeTitleSimplified) normalized = simplized(normalized);
  if (globals.titleNoiseFilter) normalized = normalized.replace(globals.titleNoiseFilter, '').trim();
  return normalized;
}
```