# Android开发--启动页背景图全屏、沉浸式

在制作启动页时，发现多款不同类型的手机设备的启动页展示效果不一样，于是总结了以下一些经验。<br>
测试手机类型：华为P9 、 荣耀10（刘海屏）、荣耀20（挖孔屏）<br>

```
Manifest.xml

//引入自定的theme
<activity android:name=".SplashActivity"
    android:theme="@style/AppTheme"
    android:configChanges="locale">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />

        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
</activity>
```

```
res => values => styles.xml

//定义的theme继承android的Theme.AppCompat.Light.NoActionBar
//设置windowBackground属性为一张全屏的背景图
<style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:windowBackground">@drawable/background</item>
    <item name="android:windowFullscreen">true</item>
</style>
```

```
res => values-v21 => styles.xml

//在API21下，增加windowDrawsSystemBarBackgrounds属性为false
<style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:windowBackground">@drawable/background</item>
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowDrawsSystemBarBackgrounds">false</item>
</style>
```

```
res => values-v28 => styles.xml

//在API28下
//增加windowDrawsSystemBarBackgrounds属性为false
//增加windowLayoutInDisplayCutoutMode属性为shortEdges
<style name="AppTheme" parent="Theme.AppCompat.Light.NoActionBar">
    <item name="android:windowBackground">@drawable/background</item>
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowDrawsSystemBarBackgrounds">false</item>
    <item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
</style>
```
