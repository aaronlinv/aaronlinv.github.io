---
date: '2022-05-10T09:08:33+08:00'
title: '安卓导航抽屉 Navigation Drawer 实现沉浸通知栏'
categories: ["Android"]
---

在使用 Navigation Drawer Activity 模版的时候，遇到了通知栏无法完全沉浸的问题，尝试搜索一些现有的解决方法，但是或多或少都会存在一些问题，通过反复尝试找到找到了一种比较靠谱的思路

## 环境
测试模拟器：Pixel 3A

compileSdk：32

minSdk：28

targetSdk：32

### 创建工程
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210514734-1960395499.png)
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210525579-2009840573.png)

### 默认效果展示：
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210614755-1486958964.png)
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210619669-753508730.png)

## 修改步骤
1. 设置状态栏变为透明：修改主题配置 
```
 <item name="android:statusBarColor" tools:targetApi="l">?attr/colorPrimaryVariant</item>
```

修改为：

```
<item name="android:statusBarColor">@android:color/transparent</item>
```
修改后完整文件：
```diff
<resources>
    <!-- Base application theme. -->
    <style name="Theme.MyApplication" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
        <!-- Primary brand color. -->
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@color/white</item>
        <!-- Secondary brand color. -->
        <item name="colorSecondary">@color/teal_200</item>
        <item name="colorSecondaryVariant">@color/teal_700</item>
        <item name="colorOnSecondary">@color/black</item>
        <!-- Status bar color. -->
+       <item name="android:statusBarColor">@android:color/transparent</item>
        <!-- Customize your theme here. -->
    </style>

    <style name="Theme.MyApplication.NoActionBar">
        <item name="windowActionBar">false</item>
        <item name="windowNoTitle">true</item>
    </style>

    <style name="Theme.MyApplication.AppBarOverlay" parent="ThemeOverlay.AppCompat.Dark.ActionBar" />

    <style name="Theme.MyApplication.PopupOverlay" parent="ThemeOverlay.AppCompat.Light" />
</resources>
```

修改后的效果：
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210630970-1651493127.png)
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210636209-924413269.png)

打开抽屉发现已经透明，但是会覆盖一层浅色阴影

2. 在布局中增加 `fitsSystemWindows` 属性

在 `app_bar_main.xml` 的 `androidx.coordinatorlayout.widget.CoordinatorLayout` 和 `com.google.android.material.appbar.AppBarLayout` 中增加：`android:fitsSystemWindows="true"`，这样它会自动的给View增加一个值等于状态栏高度的 PaddingTop，让它的背景颜色占据状态栏

完整代码：
```diff
<?xml version="1.0" encoding="utf-8"?>
<androidx.coordinatorlayout.widget.CoordinatorLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
+   android:fitsSystemWindows="true"
    tools:context=".MainActivity">

    <com.google.android.material.appbar.AppBarLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
+       android:fitsSystemWindows="true"
        android:theme="@style/Theme.MyApplication.AppBarOverlay">

        <androidx.appcompat.widget.Toolbar
            android:id="@+id/toolbar"
            android:layout_width="match_parent"
            android:layout_height="?attr/actionBarSize"
            android:background="?attr/colorPrimary"
            app:popupTheme="@style/Theme.MyApplication.PopupOverlay" />

    </com.google.android.material.appbar.AppBarLayout>

    <include layout="@layout/content_main" />

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fab"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="bottom|end"
        android:layout_marginEnd="@dimen/fab_margin"
        android:layout_marginBottom="16dp"
        app:srcCompat="@android:drawable/ic_dialog_email" />

</androidx.coordinatorlayout.widget.CoordinatorLayout>
```

效果：

![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210649500-1994494725.png)
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210656790-468543009.png)


3. 修改 `activity_main.xml`，给 `com.google.android.material.navigation.NavigationView` 增加上 `app:insetForeground="@android:color/transparent"`，去除抽屉状态栏浅色阴影，

效果：

![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210703658-1011643603.png)

4. 兼容深色模式

切换到深色模式的效果：
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210710976-27408475.png)
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210722580-1676026619.png)

修改：`night/themes.xml`

![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210730892-1605884684.png)

将 `android:statusBarColor` 设置为：`@android:color/transparent`
```xml
<item name="android:statusBarColor">@android:color/transparent</item>
```
打开抽屉发现，这个时候状态栏已经透明了，但是状态栏背景还是会有一个黑色的背景色

![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210750558-1254180792.png)
![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210757559-951559363.png)


修改 `app_bar_main.xml`，在 `<com.google.android.material.appbar.AppBarLayout` 中增加背景色，就可以了

```xml
android:background="?attr/colorPrimary"
```

最终效果：

![](../安卓导航抽屉NavigationDrawer实现沉浸通知栏/1929786-20220509210824389-1166499930.png)


## fitsSystemWindows 属性 
`android:fitsSystemWindows="true"` 的[官方文档描述](https://developer.android.com/reference/android/view/View#attr_android:fitsSystemWindows)：

> Boolean internal attribute to adjust view layout based on system windows such as the status bar. 
> If true, adjusts the padding of this view to leave space for the system windows. 
> Will only take effect if this view is in a non-embedded activity.

对某个 View 设置 `fitsSystemWindows` 为 true，本质就是给这个 View 设置了 padding，所以在 app_bar_main.xml 的 AppBarLayout 设置 `fitsSystemWindows`，这样可以使得 AppBarLayout 的背景可以通过 padding 延展到状态栏，通过对 AppBarLayout 设置背景，就可以到达沉浸状态栏的效果

## 其他常用配置
### 设置状态栏为浅色模式（文字为黑字）
代码：
```kotlin
val controller = ViewCompat.getWindowInsetsController(binding.root)
controller?.isAppearanceLightStatusBars = true
```

或者使用主题 xml 来定义：
```
<item name="android:windowLightStatusBar">true</item>
```

## 参考资料
[深色主题背景](https://developer.android.com/guide/topics/ui/look-and-feel/darktheme)
[What exactly does fitsSystemWindows do?](https://stackoverflow.com/questions/31761046/what-exactly-does-fitssystemwindows-do)
[No Action Bar & Transparent Status Bar](https://www.youtube.com/watch?v=a1CQPKuGM_A)
[Android沉浸式状态栏(透明状态栏)最佳实现](https://www.cnblogs.com/Free-Thinker/p/8624791.html)
[透明状态栏、全屏应用、沉浸模式](https://www.jianshu.com/p/4927afb3b6e8)
[NavigationView阴影](https://blog.csdn.net/fwt336/article/details/80804516)
[定制你的Toolbar](https://loody.github.io/2016/02/21/2016-02-21-custom-toolbar/)
[Android 的style和theme](https://www.jianshu.com/p/c7bab1cbc9b4)