# Android开发--关于getActivity()为空指针和onAttach版本的坑

一、getActivity() == null<br>

在网上很多资料都会说到在Activity被系统回收时，Fragment不会随着Activity而回收重新初始化，它会保存在FragmentManager里，在onCreate()方法里重新获取，
所以在Fragment里调用getActivity()有可能出现空指针。<br>

建议是在onAttach(Context context)方法里获取activity的上下文。<br>


二、onAttach版本问题<br>

方法onAttach(Context context)是在API23之后加入的，在此之前只有onAttach(Activity activity)，不过在API23后标注为废弃，但是依然可以使用；<br>

调试的结果为：<br>
<br>
先执行onAttach(Context context)<br>
再执行onAttach(Activity activity)<br>
两个方法都被执行。<br>

