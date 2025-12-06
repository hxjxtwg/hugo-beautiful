---

title: "封装LunaTV网页端成app"

date: 2025-12-03T21:13:05+08:00

lastmod: 2025-12-03T21:13:05+08:00

draft: false

categories: ["技术"]

tags: ["技术"]

author: "xxsky"

description: ""

\# cover: /images/default.jpg

---



将 LunaTV（xuhxjx/LunaTV）这个纯网页项目，打包成一个安卓 APP。这个 APP 必须能正确处理全屏播放（自动旋转、隐藏状态栏、适配刘海屏），并且在启动和退出全屏时能正确显示/隐藏状态栏。


<!--more-->

## 阶段一：【准备】安装所有必备工具
这是“搭建工厂”的阶段，在您的 Windows 电脑上准备好所有开发工具。

### 1. Node.js： 确保已安装（您使用的是 v20.19.5）。

### 2. Git： 确保已安装（用于下载项目）。

### 3. Android Studio： 确保已安装，并已完成所有初始 SDK 组件的下载。

### 4. pnpm / npm： 确保 Node.js 自带的 npm 可用。

## 阶段二：【改造】项目配置与依赖（命令行）
这是最复杂、也是最关键的阶段。我们把“纯网页” (LunaTV) 改造为“可以打包的 APP”。

### 1. 获取项目代码：

* 我们最初尝试 git clone，但遇到了网络问题。

* 最终方案： 直接从 https://github.com/xuhxjx/LunaTV 下载 ZIP 压缩包。

* 操作： 解压 LunaTV-main.zip，将其重命名为 LunaTV，并移动到您的工作目录（例如 D:\Tools\LunaTV）。

### 2. 【关键修复-1】解决“依赖冲突”：

* LunaTV 项目的 package.json 依赖是过时且损坏的。

* pnpm install 和 npm install 都会失败（ERESOLVE 错误）。

* 最终方案： 我们必须使用 npm 并附带一个“强制”参数来解决。

* 打开cmd命令窗口操作：
```bash
# 进入项目目录
D:
cd D:\Tools\LunaTV
```
```bash
# 彻底清除旧的/损坏的包（如果存在）
rd /s /q node_modules
del pnpm-lock.yaml
del package-lock.json
```
```bash
# 【关键】使用 --legacy-peer-deps 强制安装
npm install --legacy-peer-deps
```
* (这个过程会很慢（10-20分钟），并且会显示很多 WARN 警告和 vulnerabilities 漏洞，这些都是正常的，可以安全忽略。)

### 3. 安装 Capacitor（添加“APP底盘”）：

* 我们同样需要使用 legacy-peer-deps “作弊码”来安装 Capacitor。

* 操作：
```bash
# 安装 Capacitor 核心
npm install @capacitor/core --legacy-peer-deps
```
```bash
# 安装 Android 平台
npm install @capacitor/android --legacy-peer-deps
```
```bash
# 安装 Capacitor 命令行工具
npm install -D @capacitor/cli --legacy-peer-deps
```

### 4. 初始化 Capacitor（创建配置文件）：

* 操作：
```bash
npx cap init
```
* 它会提问：

  * App Name? -> XxskyTV (您输入的名字)

  * App ID? -> com.xxsky.lunatv (您输入的ID)

  * web asset directory? -> (我们最初填了 out，这是错误的,应该保持默认public)

### 5. 【关键修复-2】配置 Capacitor 并“欺骗”它：

* npx cap add 命令需要一个包含 index.html 的 webDir（网页目录）才能运行，但 LunaTV 并没有现成的。

* 最终方案： 我们将 webDir 指向 public 文件夹，并手动在里面创建一个“假”的 index.html。

* 操作： 
 * 用记事本打开 D:\Tools\LunaTV\capacitor.config.ts 文件。 
 * 将其内容替换为（确保填入您的 appId, appName 和正确的 url）：
```bash
import { CapacitorConfig } from '@capacitor/cli'; // <-- 关键：添加这一行

const config: CapacitorConfig = {
    appId: 'com.xxsky.lunatv', // 您的 App ID
    appName: 'XxskyTV',        // 您的 App Name

    // 【关键】必须指向 'public'
    webDir: 'public',

    // 【关键】指向您的服务器网址
    server: {
      url: 'https://mv.hxjx.dpdns.org'
    }
};

export default config;
```
 * 保存并关闭文件。 
 * 回到 cmd 命令行，运行“欺骗”命令： 
```bash 
# 在 public 文件夹里创建一个空的 index.html 
echo "<html></html>" > public/index.html
```

### 6. 构建与添加平台：

* 我们需要运行 npm run build 来生成 manifest，并创建 android 文件夹。

* 操作：
```bash
# 运行构建（它会失败，但能生成 manifest）
# 我们修复了 "Cannot find module './impl'" 的BUG
# 最终 `npm run build` 成功运行
npm run build 
```
```bash
# 添加 Android 平台
npx cap add android
```
```bash
# 同步所有配置
npx cap sync android
```
* (在这个过程中，我们还安装了 @capacitor/status-bar 和 @capacitor/screen-orientation 插件，但最后发现它们是“敌人”，所以我们最终把它们卸载了。最终的完美方案里不需要它们。)

## 阶段三：【原生修复】解决全屏 BUG（Android Studio）
这是我们能“完美实现”的最关键一步。我们发现所有问题（状态栏、刘海屏、旋转）都必须通过修改原生 Java 代码来接管，而不是靠插件。

### 1. 打开项目：

* 启动 Android Studio。

* 点击 "Open" (打开)。

* 选择 D:\Tools\LunaTV\android 文件夹。

### 2. 等待 Gradle 同步：

* 第一次打开会很慢，但最终会成功。

### 3.【终极修复】替换 MainActivity.java：

* 这是我们所有修复工作的结晶。这段代码教会了 APP 如何像“浏览器”一样处理全屏。

* 操作： 
 * 在 Android Studio 的左侧文件树中，导航到： app -> java -> com.xxsky.lunatv -> MainActivity.java 
 * 双击打开它。 
 * 删除里面的所有内容。 
 * 粘贴以下**(这个版本 = 刘海修复 + 全屏播放 + 自动旋转 + “智能”隐藏UI + 【新功能】“完整”的“返回键”拦截)**的代码：

```java
package com.xxsky.lunatv; // 这是您的包名

import android.content.pm.ActivityInfo;
import android.os.Build;
import android.os.Bundle;
import android.view.View;
import android.view.ViewGroup;
import android.view.Window;
import android.view.WindowManager;
import android.webkit.WebChromeClient;
import android.webkit.WebView;
import android.widget.FrameLayout;
import com.getcapacitor.BridgeActivity;
import android.view.WindowInsets;
import android.view.WindowInsetsController;

public class MainActivity extends BridgeActivity {

    private View customView;
    private WebChromeClient.CustomViewCallback customViewCallback;
    private FrameLayout fullscreenContainer;
    private WebView webView; // 我们需要这个变量

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        // --- 1. 【修复刘海屏】(不变) ---
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            Window w = getWindow();
            WindowManager.LayoutParams lp = w.getAttributes();
            lp.layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
            w.setAttributes(lp);
        }

        // --- 2. 【设置全屏播放器】 (不变) ---

        this.webView = this.bridge.getWebView();

        this.webView.setWebChromeClient(new WebChromeClient() {

            @Override
            public void onShowCustomView(View view, CustomViewCallback callback) {
                if (customView != null) { onHideCustomView(); return; }
                customView = view;
                customViewCallback = callback;
                fullscreenContainer = new FrameLayout(MainActivity.this);
                fullscreenContainer.setLayoutParams(new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
                fullscreenContainer.setBackgroundColor(getResources().getColor(android.R.color.black));
                fullscreenContainer.addView(customView, new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
                ViewGroup decorView = (ViewGroup) getWindow().getDecorView();
                decorView.addView(fullscreenContainer, new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));

                hideSystemUI(); 
                setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
            }

            @Override
            public void onHideCustomView() {
                if (customView == null) { return; }
                ViewGroup decorView = (ViewGroup) getWindow().getDecorView();
                decorView.removeView(fullscreenContainer);
                fullscreenContainer = null;
                customView = null;
                if (customViewCallback != null) {
                    customViewCallback.onCustomViewHidden();
                }

                showSystemUI(); 
                setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
            }
        });
    }

    // --- 3. 【【【“返回键”的“终极修复”（“复制”版）】】】 ---
    @Override
    public void onBackPressed() {
        // a. 当用户按下“返回”...

        // b. ...【优先】检查“全屏播放器”(`customView`) 是不是“正在显示”
        if (this.customView != null) {
            // c. ...如果是，我们就“退出全屏”

            // ↓↓↓ 我们“复制” onHideCustomView() 里的代码 ↓↓↓
            ViewGroup decorView = (ViewGroup) getWindow().getDecorView();
            decorView.removeView(fullscreenContainer);
            fullscreenContainer = null;
            customView = null;
            if (customViewCallback != null) {
                customViewCallback.onCustomViewHidden();
            }
            showSystemUI(); 
            setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
            // ↑↑↑ “复制”结束 ↑↑↑

        } 
        // d. ...【其次】检查“网页”(`webView`) 是不是“可以后退”
        else if (this.webView != null && this.webView.canGoBack()) {
            // e. ...如果是，我们就在“网页”里后退
            this.webView.goBack();
        }
        // f. ...【最后】如果“既没有全屏”也“不能后退” (在主页)
        else {
            // g. ...我们就“退出APP”
            super.onBackPressed();
        }
    }


    // --- 4. 【“智能”隐藏UI函数”（不变）】 ---
    private void hideSystemUI() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            WindowInsetsController controller = getWindow().getInsetsController();
            if (controller != null) {
                controller.hide(WindowInsets.Type.statusBars() | WindowInsets.Type.navigationBars());
                controller.setSystemBarsBehavior(WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
            }
        } else {
            View decorView = getWindow().getDecorView();
            decorView.setSystemUiVisibility(
                    View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
                    | View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                    | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                    | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                    | View.SYSTEM_UI_FLAG_FULLSCREEN);
        }
    }

    // --- 5. 【“智能”显示UI函数”（不变）】 ---
    private void showSystemUI() {
         if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            WindowInsetsController controller = getWindow().getInsetsController();
            if (controller != null) {
                controller.show(WindowInsets.Type.statusBars() | WindowInsets.Type.navigationBars());
            }
        } else {
            View decorView = getWindow().getDecorView();
            decorView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        }
    }
}
```
 * 保存这个文件 (Ctrl + S)。

另一个不完美方案(这个版本 = 刘海修复 + 全屏播放 + 自动旋转 + “智能”隐藏UI + “完整”的“返回键”拦截+沉浸式‘网页标题栏与手机状态栏重叠’)的代码：
```java
package com.xxsky.lunatv; // 这是您的包名

import android.content.pm.ActivityInfo;
import android.os.Build;
import android.os.Bundle;
import android.view.View;
import android.view.ViewGroup;
import android.view.Window;
import android.view.WindowManager;
import android.webkit.WebChromeClient;
import android.webkit.WebView;
import android.widget.FrameLayout;
import com.getcapacitor.BridgeActivity;
import android.view.Window;
import android.view.WindowManager;
import android.os.Build;
import android.view.WindowInsets;
import android.view.WindowInsetsController;

public class MainActivity extends BridgeActivity {

    private View customView;
    private WebChromeClient.CustomViewCallback customViewCallback;
    private FrameLayout fullscreenContainer;
    private WebView webView; // 我们需要这个变量

    @Override
    public void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            Window w = getWindow();
            w.setDecorFitsSystemWindows(false);
            w.setFlags(WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS, WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS);
        } else {
            Window w = getWindow();
            w.setFlags(WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS, WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS);
        }

        // --- 1. 【修复刘海屏】(不变) ---
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.P) {
            Window w = getWindow();
            WindowManager.LayoutParams lp = w.getAttributes();
            lp.layoutInDisplayCutoutMode = WindowManager.LayoutParams.LAYOUT_IN_DISPLAY_CUTOUT_MODE_SHORT_EDGES;
            w.setAttributes(lp);
        }

        // --- 2. 【设置全屏播放器】 (不变) ---

        this.webView = this.bridge.getWebView();

        this.webView.setWebChromeClient(new WebChromeClient() {

            @Override
            public void onShowCustomView(View view, CustomViewCallback callback) {
                if (customView != null) { onHideCustomView(); return; }
                customView = view;
                customViewCallback = callback;
                fullscreenContainer = new FrameLayout(MainActivity.this);
                fullscreenContainer.setLayoutParams(new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
                fullscreenContainer.setBackgroundColor(getResources().getColor(android.R.color.black));
                fullscreenContainer.addView(customView, new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));
                ViewGroup decorView = (ViewGroup) getWindow().getDecorView();
                decorView.addView(fullscreenContainer, new FrameLayout.LayoutParams(ViewGroup.LayoutParams.MATCH_PARENT, ViewGroup.LayoutParams.MATCH_PARENT));

                hideSystemUI();
                setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE);
            }

            @Override
            public void onHideCustomView() {
                if (customView == null) { return; }
                ViewGroup decorView = (ViewGroup) getWindow().getDecorView();
                decorView.removeView(fullscreenContainer);
                fullscreenContainer = null;
                customView = null;
                if (customViewCallback != null) {
                    customViewCallback.onCustomViewHidden();
                }

                showSystemUI();
                setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
            }
        });
    }

    // --- 3. 【【【“返回键”的“终极修复”（“复制”版）】】】 ---
    @Override
    public void onBackPressed() {
        // a. 当用户按下“返回”...

        // b. ...【优先】检查“全屏播放器”(`customView`) 是不是“正在显示”
        if (this.customView != null) {
            // c. ...如果是，我们就“退出全屏”

            // ↓↓↓ 我们“复制” onHideCustomView() 里的代码 ↓↓↓
            ViewGroup decorView = (ViewGroup) getWindow().getDecorView();
            decorView.removeView(fullscreenContainer);
            fullscreenContainer = null;
            customView = null;
            if (customViewCallback != null) {
                customViewCallback.onCustomViewHidden();
            }
            showSystemUI();
            setRequestedOrientation(ActivityInfo.SCREEN_ORIENTATION_PORTRAIT);
            // ↑↑↑ “复制”结束 ↑↑↑

        }
        // d. ...【其次】检查“网页”(`webView`) 是不是“可以后退”
        else if (this.webView != null && this.webView.canGoBack()) {
            // e. ...如果是，我们就在“网页”里后退
            this.webView.goBack();
        }
        // f. ...【最后】如果“既没有全屏”也“不能后退” (在主页)
        else {
            // g. ...我们就“退出APP”
            super.onBackPressed();
        }
    }


    // --- 4. 【“智能”隐藏UI函数”（不变）】 ---
    private void hideSystemUI() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            WindowInsetsController controller = getWindow().getInsetsController();
            if (controller != null) {
                controller.hide(WindowInsets.Type.statusBars() | WindowInsets.Type.navigationBars());
                controller.setSystemBarsBehavior(WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE);
            }
        } else {
            View decorView = getWindow().getDecorView();
            decorView.setSystemUiVisibility(
                    View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
                            | View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                            | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                            | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                            | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                            | View.SYSTEM_UI_FLAG_FULLSCREEN);
        }
    }

    // --- 5. 【“智能”显示UI函数”（不变）】 ---
    private void showSystemUI() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
            WindowInsetsController controller = getWindow().getInsetsController();
            if (controller != null) {
                controller.show(WindowInsets.Type.statusBars() | WindowInsets.Type.navigationBars());
            }
        } else {
            View decorView = getWindow().getDecorView();
            decorView.setSystemUiVisibility(View.SYSTEM_UI_FLAG_LAYOUT_STABLE);
        }
    }
}
```

## 阶段四：更改 APP 图标

### 1：准备您的图标文件
您需要一张正方形的图片。

* 格式： 最好是 .png 格式（支持透明背景）。

* 尺寸： 尺寸越大越好，Android Studio 会自动帮您缩放。理想尺寸是 1024x1024 像素（512x512 也可以）。

* 注意： 现代安卓图标有“安全区域”（通常是中间 66% 的圆形或圆角矩形）。请确保您的“核心”Logo 在这个安全区域内，否则它可能会被裁切。

* 准备好这张 .png 文件，并把它放在您电脑上一个容易找到的位置（比如“桌面”）。

### 2：使用 Android Studio 的“Asset Studio”工具

* 请回到您的 Android Studio（确保 D:\Tools\LunaTV\android 项目是打开的）。

* 在左上角的**“项目”文件浏览器**中（您之前找 MainActivity.java 的地方）。

* 右键点击最顶层的那个 app 文件夹（它有一个蓝色的“a”图标）。

* 在弹出的菜单中，移动到 "New" (新建) -> "Image Asset" (图像资产)。

### 3. 配置您的新图标

* 点击后，会弹出一个叫 "Asset Studio" 的新窗口。

* Icon Type (图标类型)： 保持默认的 Launcher Icons (Adaptive & Legacy)。

* Name (名称)： 保持默认的 ic_launcher。

* Source Asset (源资产)：

 * 在 Path (路径) 这一行，点击它最右边的**“文件夹”图标**。

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

* 它会弹出一个新窗口，显示它将要**“覆盖”**（Overwrite）一大堆旧的 ic_launcher.png 文件。

* 这是正常的，这正是我们想要的。

* 点击右下角的 "Finish" (完成) 按钮。

Android Studio 现在会自动生成所有不同尺寸（mdpi, hdpi, xhdpi 等）的图标，并替换掉项目里默认的那个 Capacitor 图标。

## 阶段五：启动logo图标

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

###3. 创建“智能”布局文件 (splash_layout.xml)

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

## 阶段六：【打包】生成最终的 APK

1. 在 Android Studio 中，点击顶部菜单栏的 "Build" (构建)。

2. 选择 "Generate App Bundles or APKs" (生成...)。

3. 在子菜单中，选择 "Generate APKs" (生成 APKs)。

4. 等待构建完成（BUILD SUCCESSFUL）。

5. 在右下角弹出的通知中，点击 "Locate" (定位)。

6. 最终成品： 您会得到 app-debug.apk 文件，它现在是“布局完美”且“功能完美”的。

## 附加七：更新操作

### 1. 如修改app名、App ID、server.url

* 路径：D:\Tools\MyNavApp\capacitor.config.ts
* 用记事本打开您项目根目录的capacitor.config.ts文件
```bash
{
  "appId": "com.myname.nav",
  "appName": "我的新导航名字",  <-- 修改这里
  "webDir": "public",
  "server": {
    "url": "https://..."
  }
}
```
* 保存文件
### 2. 同步”到安卓
* 打开 cmd 命令行，进入您的项目目录：
```bash
cd D:\Tools\MyNavApp
```
* 运行同步命令：
```bash
npx cap sync android
```
### 3. 重新打包：

* 打开 Android Studio（D:\Tools\MyNavApp\android）。

* Android Studio -> file -> Sync project... -> 请点击它。

* "Build" -> "Generate ... APKs" -> "Generate APKs"。

