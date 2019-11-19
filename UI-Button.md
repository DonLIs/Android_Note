# Android开发--UI-改变Button的颜色并保持点击效果

在平时开发中，想改变Button的颜色，都会去添加Button属性android:background,然而Button的颜色改变了，但点击效果却没了<br>
<br>
先来看一下Button原本使用的样式<br>

在android的values.xml文件下找到Widget.AppCompat.Button<br>
这里有好几种Button的样式<br>
```
<style name="Widget.AppCompat.Button" parent="Base.Widget.AppCompat.Button"/>
<style name="Widget.AppCompat.Button.Borderless" parent="Base.Widget.AppCompat.Button.Borderless"/>
<style name="Widget.AppCompat.Button.Borderless.Colored" parent="Base.Widget.AppCompat.Button.Borderless.Colored"/>
<style name="Widget.AppCompat.Button.ButtonBar.AlertDialog" parent="Base.Widget.AppCompat.Button.ButtonBar.AlertDialog"/>
<style name="Widget.AppCompat.Button.Colored" parent="Base.Widget.AppCompat.Button.Colored"/>
<style name="Widget.AppCompat.Button.Small" parent="Base.Widget.AppCompat.Button.Small"/>
```
可以查看Base.Widget.AppCompat.Button<br>

```
res -> values -> valses.xml

<style name="Base.Widget.AppCompat.Button" parent="android:Widget">
        <item name="android:background">@drawable/abc_btn_default_mtrl_shape</item>
        <item name="android:textAppearance">?android:attr/textAppearanceButton</item>
        <item name="android:minHeight">48dip</item>
        <item name="android:minWidth">88dip</item>
        <item name="android:focusable">true</item>
        <item name="android:clickable">true</item>
        <item name="android:gravity">center_vertical|center_horizontal</item>
    </style>
```

```
res-> values-v21 -> valses-v21.xml

<style name="Base.Widget.AppCompat.Button" parent="android:Widget.Material.Button"/>

<style name="Widget.Material.Button">
        <item name="background">@drawable/btn_default_material</item>
        <item name="textAppearance">?attr/textAppearanceButton</item>
        <item name="minHeight">48dip</item>
        <item name="minWidth">88dip</item>
        <item name="stateListAnimator">@anim/button_state_list_anim_material</item>
        <item name="focusable">true</item>
        <item name="clickable">true</item>
        <item name="gravity">center_vertical|center_horizontal</item>
    </style>
```
<br>
从源码可以看出Button的样式<br>
<br>
在我们的styles.xml中自定义style继承Button的样式Widget.AppCompat.Button <br>

```
<style name="MyStyleButton">
    <item name="buttonStyle">@style/Widget.AppCompat.Button</item>
    <item name="colorButtonNormal">@color/holo_light</item>
</style>

<Button
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="text"
    android:textColor="#F2F9EB"
    android:theme="@style/CusButton" />

```




