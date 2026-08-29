---

title: "简单封装网页app"

date: 2025-12-03T21:02:53+08:00

lastmod: 2025-12-03T21:02:53+08:00

draft: false

categories: ["技术"]

tags: ["技术"]

author: "xxsky"

description: ""

\# cover: /images/default.jpg

---



只要做一个“空壳” APP，它唯一的目的就是打开您在网址（server.url），然后适应手机的手势。


<!--more-->

## 步骤一
### 1.打开 cmd 命令行，进入这个新的、空的文件夹：
```bash
##新建文件夹
mkdir d:\Apps\dav
##进入文件夹目录
d:
cd D:\Apps\dav
```
### 2. 安装“打包工具”：
```bash
npm install @capacitor/core @capacitor/android @capacitor/cli
```
### 3. 初始化 Capacitor：
```bash
npx cap init
```
* App Name? -> 我的导航 (您自己起名)

* App ID? -> com.xxsky.nav (您自己起ID)

* web asset directory? -> public

### 4. 创建“欺骗”用的文件夹和文件：

* （npx cap init 不会创建 public 文件夹，我们必须手动创建）

* 在 cmd 中运行：
```bash
mkdir public
```
* （public 文件夹里必须有一个 index.html）

* 在 cmd 中运行：
```bash
echo "<html></html>" > public/index.html
```
* 修改配置文件：

  * 用记事本打开 D:\Tools\Nav\capacitor.config.json。

  * 确保它看起来是这样的（webDir: 'public' 并且有 server.url）：
```bash
{
  "appId": "com.myname.nav",
  "appName": "我的导航",
  "webDir": "public",

  "server": {
    "url": "https://[您的导航站网址]",
    "cleartext": true
  }
}
```

### 5. 添加 Android 平台：
```bash
npx cap add android
```
### 6. 同步配置：
```bash
npx cap sync android
```
## 步骤二
### 1. 注入“链接”修复代码
* 打开 Android Studio

* 打开您的新项目：D:\Tools\MyNavApp\android。

* 等待 Gradle 同步（这会很慢，因为它是一个新项目，需要下载“零件”）。

* 【关键】 找到您的 MainActivity.java：

* app -> java -> com.myname.nav (您在步骤1里起的新ID) -> MainActivity.java

* 双击打开它。

* 删除里面的所有内容。

* 粘贴这个新的、专门为“导航站”准备的“魔法代码”：
```bash
// 【【【【【 这是您在 init 时的 ID 】】】】】
package com.xxsky.nav; 

import android.os.Bundle;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import com.getcapacitor.BridgeActivity;
// (注意：我们不再需要 "OnBackPressedCallback" 了)

public class MainActivity extends BridgeActivity {

    // 我们需要把 webView 变量放在这里，以便“返回键”能“看到”它
    private WebView webView; 

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState); 

        // 1. 【链接修复】(和之前一样)
        //    我们把 webView 存到“全局”变量里
        this.webView = this.bridge.getWebView();

        WebSettings webSettings = this.webView.getSettings();
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        this.webView.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url); 
                return true;
            }
        });
    }

    // --- 2. 【【【这就是“返回键”的“终极修复”（安全版）】】】 ---

    // 我们“覆盖”那个“旧的、可靠的” onBackPressed 函数
    @Override
    public void onBackPressed() {
        // a. 当用户按下“返回”...

        // b. ...我们检查“浏览器”是否可以后退
        if (this.webView.canGoBack()) {
            // c. ...如果可以，我们就在“浏览器”里后退
            this.webView.goBack();
        } else {
            // d. ...如果不能(在主页)，我们就调用“系统默认”的返回 (也就是“退出APP”)
            super.onBackPressed();
        }
    }
}
```
或者有沉浸式功能的
```bash
// 【【【 关键：这是您的 ID ！】】】
package com.navgo.hxjx;

import android.graphics.Color; // <-- 【新】导入“颜色”
import android.os.Bundle;
import android.view.View; // <-- 【新】导入“View”
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import com.getcapacitor.BridgeActivity;
import com.getcapacitor.WebViewListener; // 导入“安全”工具

public class MainActivity extends BridgeActivity {

    // 我们需要把 webView 变量放在这里，以便“返回键”能“看到”它
    private WebView webView;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // --- 【【【这就是“沉浸式”的“终极修复”】】】 ---
        // 这会“强行”让窗口画在状态栏下面 (导致“重叠”)
        // 并且把状态栏设为“透明”
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
                        View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        );
        getWindow().setStatusBarColor(Color.TRANSPARENT);
        //

        // 1. 【链接修复】(和之前一样)
        //    我们把 webView 存到“全局”变量里
        this.webView = this.bridge.getWebView();

        WebSettings webSettings = this.webView.getSettings();
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        this.webView.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }
        });
    }

    // --- 2. 【【【这就是“返回键”的“终极修复”（安全版）】】】 ---

    // 我们“覆盖”那个“旧的、可靠的” onBackPressed 函数
    @Override
    public void onBackPressed() {
        // a. 当用户按下“返回”...

        // b. ...我们检查“浏览器”是否可以后退
        if (this.webView.canGoBack()) {
            // c. ...如果可以，我们就在“浏览器”里后退
            this.webView.goBack();
        } else {
            // d. ...如果不能(在主页)，我们就调用“系统默认”的返回 (也就是“退出APP”)
            super.onBackPressed();
        }
    }
}
```

## 步骤三
更改 APP 图标

### 1：准备您的图标文件
您需要一张正方形的图片。

* 格式： 最好是 .png 格式（支持透明背景）。

* 尺寸： 尺寸越大越好，Android Studio 会自动帮您缩放。理想尺寸是 1024x1024 像素（512x512 也可以）。

* 注意： 现代安卓图标有“安全区域”（通常是中间 66% 的圆形或圆角矩形）。请确保您的“核心”Logo 在这个安全区域内，否则它可能会被裁切。

* 准备好这张 .png 文件，并把它放在您电脑上一个容易找到的位置（比如“桌面”）。

### 2：使用 Android Studio 的“Asset Studio”工具

* 请回到您的 Android Studio（确保 D:\Tools\LunaTV\android 项目是打开的）。

* 在左上角的“项目”文件浏览器中（您之前找 MainActivity.java 的地方）。

* 右键点击最顶层的那个 app 文件夹（它有一个蓝色的“a”图标）。

* 在弹出的菜单中，移动到 "New" (新建) -> "Image Asset" (图像资产)。

### 3. 配置您的新图标

* 点击后，会弹出一个叫 "Asset Studio" 的新窗口。

* Icon Type (图标类型)： 保持默认的 Launcher Icons (Adaptive & Legacy)。

* Name (名称)： 保持默认的 ic_launcher。

* Source Asset (源资产)：

 * 在 Path (路径) 这一行，点击它最右边的“文件夹”图标。

 * 在弹出的文件选择器中，找到并选择您在步骤 1 中准备好的那张 .png 图标文件。

 * 点击 OK。

* 调整 (Scaling)：

 * 您会立刻在右侧的“预览”窗口看到您的图标。

 * 您可能会看到一个“安全区”的圆圈。

 * 在中间的 Resize (调整大小) 滑块上，您可以左右拖动，把您的图标缩放到您觉得合适的大小（通常是让它刚好填满那个“安全区”）。

* （可选）Background Layer (背景层)：

 * 如果您的 .png 是透明的，您可以在这里给它加一个纯色背景。

 * 在 Name: ic_launcher 的下方，您会看到三个标签页 (Tab)：

    * Foreground Layer (前景层) <-- 您现在在这里！（它是蓝色的）

    * Background Layer (背景层) <-- 您要点击这里！

    * Options (选项)
 * 点击一下第二个标签页，也就是 Background Layer (背景层)
    * Source Asset -> Asset Type -> Color (颜色)，
    * 然后选择您想要的背景色。

### 4. 生成图标
* 当您对预览满意后，点击右下角的 "Next" (下一步) 按钮。

* 它会弹出一个新窗口，显示它将要“覆盖”（Overwrite）一大堆旧的 ic_launcher.png 文件。

* 这是正常的，这正是我们想要的。

* 点击右下角的 "Finish" (完成) 按钮。

Android Studio 现在会自动生成所有不同尺寸（mdpi, hdpi, xhdpi 等）的图标，并替换掉项目里默认的那个 Capacitor 图标。

## 步骤四：
启动logo图标

### 1. 准备您的新图片
* 请您准备好您自己的“启动页面”图片（例如，您桌面上有一个 logo.png）。

* 重要：您不需要准备11个版本。您只需要一张高分辨率的 .png 文件。

* 尺寸如：215*215，底色透明。

### 2. 添加logo.png
* 请您再次找到并打开那个文件： app -> res -> drawable -> splash_layout.xml

* 回到 Android Studio。

* 在左侧文件树中，找到 res/drawable 文件夹。

* 右键点击 drawable 文件夹->open in->Explorer" (在资源管理器中显示)。

* Windows 文件夹会打开 drawable 文件夹。

* 把logo.png 复制到这个文件夹里。 (我们不再需要 drawable-hdpi那些文件夹了)

### 3. 创建“智能”布局文件 (splash_layout.xml)

* 回到 Android Studio。

* 右键点击 res/drawable 文件夹。

* 选择 "New" (新建) -> "Drawable Resource File" (Drawable 资源文件)。

* File name (文件名): 请输入 splash_layout

* Root element (根元素): 请输入 layer-list

* 点击 OK。

* 一个新的 splash_layout.xml 文件会生成。

* 在资源管理器中记事本打开，然后替换粘贴以下全部代码：
```xml
<?xml version="1.0" encoding="utf-8"?>
<layer-list xmlns:android="http://schemas.android.com/apk/res/android">

    <item>
        <color android:color="?android:attr/colorBackground" /> 
    </item>

    <item
        android:width="200dp" 
        android:height="200dp"
        android:drawable="@drawable/logo"  
        android:gravity="center" />

</layer-list>
```
*（重要！） 在粘贴后，请务必修改 android:drawable="@drawable/logo" 这一行，把 logo 改成您自己的文件名（不带 .png）。

* 保存并关闭 splash_layout.xml。

### 4. 告诉 APP 使用“新”的启动页

* 打开 res/values/styles.xml 文件。

* 找到 AppTheme.NoActionBarLaunch 这一段，整个`<style>`替换修改成
```xml
    <style name="AppTheme.NoActionBarLaunch" parent="AppTheme.NoActionBar">
        <item name="android:windowBackground">@drawable/splash_layout</item>
        <item name="windowActionBar">false</item>
        <item name="windowNoTitle">true</item>
    </style>
```
或者
```xml
   <style name="AppTheme.NoActionBarLaunch" parent="Theme.AppCompat.Light.NoActionBar">
       <item name="android:windowBackground">@drawable/splash_layout</item>
       <item name="windowActionBar">false</item>
       <item name="windowNoTitle">true</item>
   </style>
```
* 保存并关闭 styles.xml。

* （可选，但推荐） 为了省空间，您可以删除掉那 11 张旧的 splash.png 文件了（在 drawable 文件夹下的 splash (11) 组上点右键 -> "Delete"）。

## 步骤五：
打包生成最终的 APK

### 1. 在 Android Studio 中，点击顶部菜单栏的 "Build" (构建)。

### 2. 选择 "Generate App Bundles or APKs" (生成...)。

### 3. 在子菜单中，选择 "Generate APKs" (生成 APKs)。

### 4. 等待构建完成（BUILD SUCCESSFUL）。

### 5. 在右下角弹出的通知中，点击 "Locate" (定位)。

### 6. 最终成品： 您会得到 app-debug.apk 文件，它现在是“布局完美”且“功能完美”的。

## 步骤六：
强制排除掉冲突的旧包

1.打开你的电脑文件夹，直接进入 C:\Users\Administrator\.gradle 目录。

2.在该文件夹下新建一个文本文档，将其重命名为 init.gradle（必须确保删除了 .txt 后缀）。

3.用记事本打开这个文件，直接粘贴以下代码并保存：
```
allprojects {
    configurations.all {
        exclude group: 'org.jetbrains.kotlin', module: 'kotlin-stdlib-jdk7'
        exclude group: 'org.jetbrains.kotlin', module: 'kotlin-stdlib-jdk8'
    }
}
```
电脑里安装了 NVM for Windows（Node版本管理器）。

NVM 会强制接管系统的 Node.js 运行环境。只要它存在，你刚才通过官方安装包（.msi）手动安装的 24.20.0 就会被直接“屏蔽”掉，导致命令行依然固执地使用 NVM 锁定的旧版本（20.19.5）。

既然有 NVM 接管，最快的解决方式是直接用它的命令来切换版本，完全不需要去控制面板卸载东西。请直接在刚才那个黑色命令行窗口中按顺序执行以下命令：

* 下载新版本
输入 nvm install 24.20.0 并回车，等待下载完成。

* 切换到新版本
输入 nvm use 24.20.0 并回车，系统会提示版本已切换。

* 验证并继续
输入 node -v，此时屏幕应该会正确显示 v24.20.0。
确认无误后，直接输入你的打包同步命令 npx cap sync android 即可顺利执行。

## 步骤七：
net::ERR_CLEARTEXT_NOT_PERMITTED 是 Android 开发中最常见的安全限制问题

第一步：找到 AndroidManifest.xml 文件
在 Android Studio 左侧的目录树中，依次展开 app -> manifests，双击打开 AndroidManifest.xml。

第二步：添加允许明文流量的代码
在文件里找到 <application 开头的那一段，在它里面的空白处（不要破坏原有的代码）加上这一行核心指令：
```
android:usesCleartextTraffic="true"
```
加完之后，你的 <application 标签看起来应该是类似这样的：
```
<application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme"
        android:usesCleartextTraffic="true"> <!-- 【就是加上这一行】 -->
        
        <!-- 下面是其他的 activity 等配置... -->
    </application>
```
第三步：重新打包运行
加完这行代码后直接保存，点击顶部的绿色三角形（Run）重新编译安装到手机。

再次打开你的导航 App，点击局域网的 Emby、Oplist 等内网 HTTP 链接，就可以完美顺畅地跳转打开了，不会再有任何拦截。

## 附录：
1.longpt网站封装带下载种子功能的MainActivity.java

```
// 【【【 关键：这是您的 ID ！】】】
package com.xxsky.longpt;

import android.graphics.Color;
import android.os.Bundle;
import android.view.View;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import com.getcapacitor.BridgeActivity;
import com.getcapacitor.WebViewListener;

import android.app.DownloadManager;
import android.net.Uri;
import android.os.Environment;
import android.webkit.CookieManager;
import android.webkit.DownloadListener;
import android.webkit.URLUtil;
import android.widget.Toast;

// 【新增的解码与正则表达式工具】
import java.net.URLDecoder;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class MainActivity extends BridgeActivity {

    private WebView webView;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // --- 【保留】沉浸式状态栏修复 ---
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
                        View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        );
        getWindow().setStatusBarColor(Color.TRANSPARENT);

        // --- 【保留】链接覆盖修复 ---
        this.webView = this.bridge.getWebView();
        WebSettings webSettings = this.webView.getSettings();
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        this.webView.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }
        });

        // --- 【修复】接管下载功能并使用正则处理 PT 站文件名 ---
        this.webView.setDownloadListener(new DownloadListener() {
            @Override
            public void onDownloadStart(String url, String userAgent, String contentDisposition, String mimeType, long contentLength) {
                DownloadManager.Request request = new DownloadManager.Request(Uri.parse(url));

                // 传递网页 Cookie 绕过 PT 站下载权限拦截
                String cookies = CookieManager.getInstance().getCookie(url);
                request.addRequestHeader("cookie", cookies);
                request.addRequestHeader("User-Agent", userAgent);

                // 1. 获取基础文件名（安卓默认 API，通常会取到 download.php）
                String fileName = URLUtil.guessFileName(url, contentDisposition, mimeType);

                // 2. 利用正则表达式从 HTTP 头部强行提取被 URL 编码的真实文件名
                if (contentDisposition != null) {
                    try {
                        Matcher m1 = Pattern.compile("filename\\*\\s*=\\s*UTF-8''([^;]+)", Pattern.CASE_INSENSITIVE).matcher(contentDisposition);
                        Matcher m2 = Pattern.compile("filename\\s*=\\s*\"?([^\";]+)\"?", Pattern.CASE_INSENSITIVE).matcher(contentDisposition);
                        if (m1.find()) {
                            fileName = URLDecoder.decode(m1.group(1), "UTF-8");
                        } else if (m2.find()) {
                            fileName = URLDecoder.decode(m2.group(1), "UTF-8");
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

                // 3. 正则清理标签：剔除开头无用的站点前缀，保持名称干净并保留 H265、2160p 等关键参数
                fileName = fileName.replaceAll("^\\[.*?\\][\\.\\s_-]*", "");

                // 4. 极端情况兜底防错，确保后缀正确
                if (fileName.startsWith("download.php")) {
                    fileName = "PT_Torrent_" + System.currentTimeMillis() + ".torrent";
                } else if (!fileName.endsWith(".torrent") && mimeType != null && mimeType.contains("torrent")) {
                    fileName = fileName + ".torrent";
                }

                // 呼出系统下载器并存入 Download 文件夹
                request.setDescription("正在下载...");
                request.setTitle(fileName);
                request.allowScanningByMediaScanner();
                request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED);
                request.setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName);

                DownloadManager dm = (DownloadManager) getSystemService(DOWNLOAD_SERVICE);
                dm.enqueue(request);

                Toast.makeText(getApplicationContext(), "开始下载: " + fileName, Toast.LENGTH_SHORT).show();
            }
        });
    }

    // --- 【保留】返回键逻辑修复 ---
    @Override
    public void onBackPressed() {
        if (this.webView.canGoBack()) {
            this.webView.goBack();
        } else {
            super.onBackPressed();
        }
    }
}
```

2.增加自动复制功能
```
// 【【【 关键：这是您的 ID ！】】】
package com.xxsky.longpt;

import android.graphics.Color;
import android.os.Bundle;
import android.view.View;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import com.getcapacitor.BridgeActivity;

import android.app.DownloadManager;
import android.net.Uri;
import android.os.Environment;
import android.webkit.CookieManager;
import android.webkit.DownloadListener;
import android.webkit.URLUtil;
import android.widget.Toast;

import java.net.URLDecoder;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

// --- 【新增的剪贴板必备工具】 ---
import android.content.ClipboardManager;
import android.content.ClipData;
import android.content.Context;

public class MainActivity extends BridgeActivity {

    private WebView webView;

    // --- 【新增核心】这是一个连接网页和安卓原生剪贴板的“桥梁” ---
    class JSBridge {
        @android.webkit.JavascriptInterface
        public void copyText(String text) {
            ClipboardManager clipboard = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
            ClipData clip = ClipData.newPlainText("Copied", text);
            clipboard.setPrimaryClip(clip);
            // 呼出安卓原生底层提示，无视网页限制
            runOnUiThread(() -> Toast.makeText(getApplicationContext(), "✅ 已自动复制: " + text, Toast.LENGTH_SHORT).show());
        }
    }

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // --- 【保留】沉浸式状态栏修复 ---
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
                        View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        );
        getWindow().setStatusBarColor(Color.TRANSPARENT);

        this.webView = this.bridge.getWebView();
        WebSettings webSettings = this.webView.getSettings();
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        // --- 【保留】电脑端网页自适应与缩放优化 ---
        webSettings.setUseWideViewPort(true); 
        webSettings.setLoadWithOverviewMode(true); 
        webSettings.setSupportZoom(true);
        webSettings.setBuiltInZoomControls(true);
        webSettings.setDisplayZoomControls(false); 

        // --- 【保留】防频繁掉线与多媒体加载优化 ---
        webSettings.setDomStorageEnabled(true);
        webSettings.setDatabaseEnabled(true);
        webSettings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);

        // --- 【关键配置】将剪贴板桥梁注册到内核中 ---
        this.webView.addJavascriptInterface(new JSBridge(), "AndroidClipboard");

        // --- 【原生增强】如果长按的是标准的超链接，直接底层拦截并复制 ---
        this.webView.setOnLongClickListener(new View.OnLongClickListener() {
            @Override
            public boolean onLongClick(View v) {
                WebView.HitTestResult result = ((WebView) v).getHitTestResult();
                if (result != null && result.getType() == WebView.HitTestResult.SRC_ANCHOR_TYPE) {
                    String url = result.getExtra();
                    if (url != null) {
                        ClipboardManager clipboard = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
                        clipboard.setPrimaryClip(ClipData.newPlainText("URL", url));
                        Toast.makeText(getApplicationContext(), "✅ 链接已复制", Toast.LENGTH_SHORT).show();
                        return true; 
                    }
                }
                return false;
            }
        });

        this.webView.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }

            @Override
            public void onPageFinished(WebView view, String url) {
                super.onPageFinished(view, url);
                
                // --- 【黑科技】强制注入“长按一键自动复制”脚本 ---
                view.evaluateJavascript(
                    "javascript:(function() { " +
                    "  var style = document.createElement('style');" +
                    "  style.innerHTML = '* { -webkit-user-select: text !important; user-select: text !important; }';" +
                    "  document.head.appendChild(style);" +
                    "  var longPressTimer;" +
                    "  document.addEventListener('touchstart', function(e) { " +
                    "    longPressTimer = setTimeout(function() { " +
                    "      var text = window.getSelection().toString();" +
                    "      if(!text) { text = (e.target.innerText || e.target.value || '').trim(); }" +
                    "      if(text && text.length > 0) { AndroidClipboard.copyText(text); } " +
                    "    }, 600); " + // 手指按住 0.6 秒即触发自动复制
                    "  });" +
                    "  document.addEventListener('touchend', function() { clearTimeout(longPressTimer); });" +
                    "  document.addEventListener('touchmove', function() { clearTimeout(longPressTimer); });" +
                    "})()", null);
            }
        });

        // --- 【保留】接管下载功能并使用正则处理 PT 站文件名 ---
        this.webView.setDownloadListener(new DownloadListener() {
            @Override
            public void onDownloadStart(String url, String userAgent, String contentDisposition, String mimeType, long contentLength) {
                DownloadManager.Request request = new DownloadManager.Request(Uri.parse(url));

                String cookies = CookieManager.getInstance().getCookie(url);
                request.addRequestHeader("cookie", cookies);
                request.addRequestHeader("User-Agent", userAgent);

                String fileName = URLUtil.guessFileName(url, contentDisposition, mimeType);

                if (contentDisposition != null) {
                    try {
                        Matcher m1 = Pattern.compile("filename\\*\\s*=\\s*UTF-8''([^;]+)", Pattern.CASE_INSENSITIVE).matcher(contentDisposition);
                        Matcher m2 = Pattern.compile("filename\\s*=\\s*\"?([^\";]+)\"?", Pattern.CASE_INSENSITIVE).matcher(contentDisposition);
                        if (m1.find()) {
                            fileName = URLDecoder.decode(m1.group(1), "UTF-8");
                        } else if (m2.find()) {
                            fileName = URLDecoder.decode(m2.group(1), "UTF-8");
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

                fileName = fileName.replaceAll("^\\[.*?\\][\\.\\s_-]*", "");

                if (fileName.startsWith("download.php")) {
                    fileName = "PT_Torrent_" + System.currentTimeMillis() + ".torrent";
                } else if (!fileName.endsWith(".torrent") && mimeType != null && mimeType.contains("torrent")) {
                    fileName = fileName + ".torrent";
                }

                request.setDescription("正在下载...");
                request.setTitle(fileName);
                request.allowScanningByMediaScanner();
                request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED);
                request.setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName);

                DownloadManager dm = (DownloadManager) getSystemService(DOWNLOAD_SERVICE);
                dm.enqueue(request);

                Toast.makeText(getApplicationContext(), "开始下载: " + fileName, Toast.LENGTH_SHORT).show();
            }
        });
    }

    // --- 【保留】返回键逻辑修复 ---
    @Override
    public void onBackPressed() {
        if (this.webView.canGoBack()) {
            this.webView.goBack();
        } else {
            super.onBackPressed();
        }
    }
}
```

3.复制功能弹窗
```
// 【【【 关键：这是您的 ID ！】】】
package com.xxsky.longpt;

import android.graphics.Color;
import android.os.Bundle;
import android.view.View;
import android.webkit.WebSettings;
import android.webkit.WebView;
import android.webkit.WebViewClient;
import com.getcapacitor.BridgeActivity;

import android.app.DownloadManager;
import android.net.Uri;
import android.os.Environment;
import android.webkit.CookieManager;
import android.webkit.DownloadListener;
import android.webkit.URLUtil;
import android.widget.Toast;

import java.net.URLDecoder;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

// --- 【新增：原生菜单与剪贴板需要的组件】 ---
import android.view.ContextMenu;
import android.view.MenuItem;
import android.content.ClipboardManager;
import android.content.Context;
import android.content.ClipData;

public class MainActivity extends BridgeActivity {

    private WebView webView;

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // --- 【保留】沉浸式状态栏修复 ---
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_LAYOUT_STABLE |
                        View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
        );
        getWindow().setStatusBarColor(Color.TRANSPARENT);

        this.webView = this.bridge.getWebView();
        WebSettings webSettings = this.webView.getSettings();
        webSettings.setJavaScriptCanOpenWindowsAutomatically(true);

        // --- 【保留】电脑端网页自适应与缩放优化 ---
        webSettings.setUseWideViewPort(true); 
        webSettings.setLoadWithOverviewMode(true); 
        webSettings.setSupportZoom(true);
        webSettings.setBuiltInZoomControls(true);
        webSettings.setDisplayZoomControls(false); 

        // --- 【保留】防频繁掉线与多媒体加载优化 ---
        webSettings.setDomStorageEnabled(true);
        webSettings.setDatabaseEnabled(true);
        webSettings.setMixedContentMode(WebSettings.MIXED_CONTENT_ALWAYS_ALLOW);

        // --- 【终极修复】向安卓系统注册原生长按菜单 ---
        this.webView.setLongClickable(true);
        registerForContextMenu(this.webView); // 核心：让 WebView 拥有呼出原生菜单的权限

        // --- 【保留】破解网页 JS 的防复制机制 ---
        this.webView.setWebViewClient(new WebViewClient() {
            @Override
            public boolean shouldOverrideUrlLoading(WebView view, String url) {
                view.loadUrl(url);
                return true;
            }

            @Override
            public void onPageFinished(WebView view, String url) {
                super.onPageFinished(view, url);
                view.evaluateJavascript(
                    "javascript:(function() { " +
                    "var style = document.createElement('style');" +
                    "style.innerHTML = '* { -webkit-user-select: text !important; user-select: text !important; -webkit-touch-callout: default !important; }';" +
                    "document.head.appendChild(style);" +
                    "})()", null);
            }
        });

        // --- 【保留】接管下载功能并使用正则处理 PT 站文件名 ---
        this.webView.setDownloadListener(new DownloadListener() {
            @Override
            public void onDownloadStart(String url, String userAgent, String contentDisposition, String mimeType, long contentLength) {
                DownloadManager.Request request = new DownloadManager.Request(Uri.parse(url));

                String cookies = CookieManager.getInstance().getCookie(url);
                request.addRequestHeader("cookie", cookies);
                request.addRequestHeader("User-Agent", userAgent);

                String fileName = URLUtil.guessFileName(url, contentDisposition, mimeType);

                if (contentDisposition != null) {
                    try {
                        Matcher m1 = Pattern.compile("filename\\*\\s*=\\s*UTF-8''([^;]+)", Pattern.CASE_INSENSITIVE).matcher(contentDisposition);
                        Matcher m2 = Pattern.compile("filename\\s*=\\s*\"?([^\";]+)\"?", Pattern.CASE_INSENSITIVE).matcher(contentDisposition);
                        if (m1.find()) {
                            fileName = URLDecoder.decode(m1.group(1), "UTF-8");
                        } else if (m2.find()) {
                            fileName = URLDecoder.decode(m2.group(1), "UTF-8");
                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }

                fileName = fileName.replaceAll("^\\[.*?\\][\\.\\s_-]*", "");

                if (fileName.startsWith("download.php")) {
                    fileName = "PT_Torrent_" + System.currentTimeMillis() + ".torrent";
                } else if (!fileName.endsWith(".torrent") && mimeType != null && mimeType.contains("torrent")) {
                    fileName = fileName + ".torrent";
                }

                request.setDescription("正在下载...");
                request.setTitle(fileName);
                request.allowScanningByMediaScanner();
                request.setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED);
                request.setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName);

                DownloadManager dm = (DownloadManager) getSystemService(DOWNLOAD_SERVICE);
                dm.enqueue(request);

                Toast.makeText(getApplicationContext(), "开始下载: " + fileName, Toast.LENGTH_SHORT).show();
            }
        });
    }

    // ==========================================================
    // --- 【新增核心】创建长按原生菜单 ---
    // ==========================================================
    @Override
    public void onCreateContextMenu(ContextMenu menu, View v, ContextMenu.ContextMenuInfo menuInfo) {
        super.onCreateContextMenu(menu, v, menuInfo);
        
        // 探测用户手指当前按在什么东西上
        WebView.HitTestResult result = this.webView.getHitTestResult();
        if (result != null) {
            int type = result.getType();
            
            // 如果按的是链接（比如 RSS 地址）
            if (type == WebView.HitTestResult.SRC_ANCHOR_TYPE || type == WebView.HitTestResult.SRC_IMAGE_ANCHOR_TYPE) {
                menu.setHeaderTitle("操作链接");
                menu.add(0, 1, 0, "复制链接地址"); // 动态添加一个复制按钮
            } 
            // 如果按的是纯图片
            else if (type == WebView.HitTestResult.IMAGE_TYPE) {
                menu.setHeaderTitle("操作图片");
                menu.add(0, 2, 0, "复制图片链接");
            }
        }
    }

    // --- 【新增核心】点击菜单后的执行动作 ---
    @Override
    public boolean onContextItemSelected(MenuItem item) {
        WebView.HitTestResult result = this.webView.getHitTestResult();
        if (result == null) return false;

        String url = result.getExtra(); // 抓取底层真实的网址
        if (url == null) return false;

        // 如果用户点击了“复制”
        if (item.getItemId() == 1 || item.getItemId() == 2) {
            // 调用安卓系统剪贴板
            ClipboardManager clipboard = (ClipboardManager) getSystemService(Context.CLIPBOARD_SERVICE);
            ClipData clip = ClipData.newPlainText("Copied Link", url);
            clipboard.setPrimaryClip(clip);
            
            // 底部弹个小提示
            Toast.makeText(this, "✅ 已复制到剪贴板", Toast.LENGTH_SHORT).show();
            return true;
        }
        return super.onContextItemSelected(item);
    }
    // ==========================================================


    // --- 【保留】返回键逻辑修复 ---
    @Override
    public void onBackPressed() {
        if (this.webView.canGoBack()) {
            this.webView.goBack();
        } else {
            super.onBackPressed();
        }
    }
}
```




