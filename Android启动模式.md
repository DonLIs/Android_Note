# Android启动模式

## Task & Affinity

Task
> Task是一个任务栈，用来装载Activity。先创建的Activity会压入栈底，后创建的Activity会压入栈顶，遵循先入后出的原则。
> 直到最后一个Activity销毁时，Task清空信息并销毁。

Affinity
> Affinity是Activity的一个属性，默认情况下它是App的包名，Affinity名字相同的Activity在创建时会压入同一个Task中。
> Task也有Affinity，以第一个Activity的Affinity为名字。

> 启动Activity可以分为静态模式和动态模式，静态模式：通过在Manifest清单文件上设置<activity>标签的`android:launchMode`属性指定哪一种启动模式；
> 动态模式：通过代码Intent的setFlags方法设置Flag，然后使用Intent启动Activity。


## 四种启动模式(静态)

### standard
> Activity的默认启动模式，在不设置launchMode的情况下为standard。
> A打开B，B再打开C，[A] -> [AB] -> [ABC]
> 再回退3次，[ABC] -> [AB] -> [A] -> [空]

### singleTop
> singleTop模式表示为栈顶复用，如果栈顶的Activity是目标Activity，那么不会重新创建Activity并复用该Activity，回调onNewIntent方法，否则将创建一个新的Activity压入栈。
> 将B设置为singleTop
> A打开B，B再打开C，C再打开B，[A] -> [AB] -> [ABC] -> [ABCB]
> A打开B，B再打开B，[A] -> [AB] -> [AB]

### singleTask
> singleTask模式表示为栈内复用，如果栈内存在该目标Activity，那么会移除该目标Activity之上的其他Activity，回调onNewIntent方法，否则将创建一个新的Activity压入栈。
> 将A设置为singleTask
> A打开B，B再打开C，C再打开A，[A] -> [AB] -> [ABC] -> [A]
> B打开C，C再打开A，[B] -> [BC] -> [BCA]

### singleInstance
> 该模式会创建一个新的Task并拥有新的Affinity，Task中有且只能存放该目标Activity，如果目标Activity已存在，则把该Task由后台切换到前台。
> 只设置C为singleInstance，其他都是默认standard
> A打开B，B再打开C，[A] -> [AB] -> [C][AB]
> 再回退3次，[C][AB] -> [AB] -> [A] -> [空]
> A打开B，B再打开C，C再打开D，[A] -> [AB] -> [C][AB] -> [ABD][C]
> 再回退4次，[ABD][C] -> [AB][C] -> [A][C] -> [C] -> [空]

## Intent-Flag(动态)

### FLAG_ACTIVITY_NEW_TASK
> 如果目标Activity的Affinity没有额外在Manifest文件中声明，也就是使用默认的，那么新创建的Activity会压入默认的Task中。
> 如果目标Activity设置了与默认包名不一样的Affinity，那么会先创建该Affinity名字的Task，再把新创建的Activity压入该Task中。
> 在非Activity的上下文启动Activity的话，需要添加FLAG_ACTIVITY_NEW_TASK。

### FLAG_ACTIVITY_CLEAR_TASK
> 清空目标Activity所在Task的全部Activity，目标Activity成为该Task中的根Activity，与FLAG_ACTIVITY_NEW_TASK配合使用。

### FLAG_ACTIVITY_SINGLE_TOP
> 该标记与singleTop一样。

### FLAG_ACTIVITY_CLEAR_TOP
> 该标记与singleTask很相似，如果与FLAG_ACTIVITY_SINGLE_TOP一起使用的话，会清除目标Activity之上的全部Activity，并回调onNewIntent；
> 如果没添加FLAG_ACTIVITY_SINGLE_TOP，那么会清除目标Activity之上的全部Activity，目标Activity会先销毁并重新创建，显示在栈顶。

### FLAG_ACTIVITY_NO_HISTORY
> 标志了FLAG_ACTIVITY_NO_HISTORY，不会把目标Activity压入栈，