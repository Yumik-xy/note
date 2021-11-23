# Android简介

## Activity声明

创建任何一个Activity后，必须在AndroidManifest.xml文件中进行声明，否则该Activity文件不会视为一个真正的Activity

其中必须要指定的名字为android:name=".ExampleActivity"，这个名字称为类名，其对应的Activity文件名一致

```xml
<manifest ... >
  <application ... >
      <activity android:name=".ExampleActivity" />
      ...
  </application ... >
  ...
</manifest >
```
## 权限声明

对于某些危险事件必须要向用户进行权限申请，例如网络请求和数据存储，这些权限需要在清单中进行阐述，当使用到他们时，系统就会自动向用户进行申请

所有列在AndroidManifest.xml的声明权限都会在安装时罗列出来

```xml
<manifest>
	<activity android:permission="com.google.socialapp.permission.SHARE_POST"/>
    ...
</manifest >
```

# Activity的生命周期

## 生命周期的概念

Activity生命周期一共有六个核心回调，分别是onCreate()、onStart()、onResume()、onPause()、onStop()、onDestroy()，当用户的操作使得生命周期产生变化时，系统便会调用这些回调，因此我们可以通过回调明确感知当前的生命周期，并执行对应的操作

<img src="https://developer.android.google.cn/guide/components/images/activity_lifecycle.png" alt="img" style="zoom:67%;" />

## 生命周期回调

### onCreate(savedInstanceState: Bundle?)

会在**首次**创建Activity的时候触发，一般只会在整个生命周期中触发一次，因此那些只需要执行一次的「视图绑定」和「数据绑定」应该在这里执行

注意到onCreate回调包含savedInstanceState参数，该参数会保存先前Activity销毁前写入的Bundle对象，如果该Activity对象为第一次创建，则该值为null

### onstart()

在onCreate状态结束后，会进入onStart状态，并执行该回调

onstart()的调用意味着Activity即将对用户可见（但是仍然看不见），因此需要对前台互动进行准备，开始进行界面的维护

但是这一状态持续的时间非常短，几乎不会做停留，大部分时候也不会在这一步执行操作

### onResume()

在onStart状态结束后，会进入onResume状态，并执行该回调

onstart()的调用意味着Activity**已经对用户可见**，应用会**一致保持在该状态**，直到某些事件发生使得生命周期发生改变

如果某些操作使得应用失去了焦点，发生了中断事件，则Activity会进入onPause状态。如果系统没有回收内存而用户返回了应用，则Activity会重新返回onResume状态，并再一次执行该回调。

🌰该状态可能用于如下应用：视频播放器

用户进入Activity后进入onResume状态，通过回调执行网络请求和视频播放任务，当用户切出应用后，进入onPause状态则暂停网络请求并暂停视频播放，直到用户重新返回应用，继续执行onResume回调播放视频

### onPause()

此方法是用户将要离开当前Activity的第一个标志回调，但这并不意味着Activity会被销毁，仅仅是不在前台了

系统进入该状态的原因可能有：

1. 某个事件导致应用发生中断，失去焦点

2. 多窗口模式下，无论如何只能有一个应用获取焦点，即使是该Activity仍然可见，但仍处于onPause状态

3. 有半透明的Activity被执行，如「对话框」或「下拉列表」等

4. 用户进行「锁屏操作」

   > 如果在AndroidManifest.xml未指定屏幕方向，则锁屏时生命周期为：
   >
   > onPause -> onStop -> onDestroy -> onCreate -> onStart -> onResume -> onPause
   >
   > 解锁时生命周期为：
   >
   > onResume -> onPause -> onStop -> onDestroy -> onCreate -> onStart -> onResume
   >
   > 其锁屏和解锁时都进行了一次销毁和重建的操作，这样会导致资源的浪费
   >
   > 如果指定了屏幕方向screenOrientation="landscape"，则锁屏时生命周期为：
   >
   > onPause -> onStop
   >
   > 解锁时生命周期为：
   >
   > onRestart -> onStart -> onResume

所以，为了更好的支持「多窗口模式」很多资源可以在onStop的状态再释放，但如果为了保证应用的低功耗，很多资源就应当在onPause的时候进行释放

此外，onPause的持续时间很短，不应执行大量的数据保存操作或网络请求，应当在onStop中进行

### onStop()

进入此状态后，Activity不再对用户可见

由于进入该状态后并不意味着引用会被直接销毁，Activity还是可以进行部分后台操作，因此一般可以进行以下操作：

1. 开始释放用户的资源，或者调制界面动画的渲染速度，以节约能耗
2. 存储界面数据或临时草稿，进行数据库读写
3. 进行网络通信
4. ……

如果Activity返回，则会调用onRestart()回调（该回调并没有对应的生命周期），如果被销毁则进入onDestroy状态

### onDestroy()

进入该状态一般有两种情况，Activity即将结束或设备旋转/进入多窗口模式

进入该状态后，系统会释放所有在之前申请而未被释放的所有资源

# Activity启动模式

定义启动模式有两种方式：

## 使用清单文件

可以直接在AndroidManifest.xml文件中声明

### 默认模式standard

在这个模式下，每一次创建Activity都会创建一个新的实例，该实例会执行完整的生命周期

因此该模式下允许同一个Activity创建多次，当按下Back返回时，会按创建顺序依次退出

🌰例子：

|      操作      |  task栈   |
| :------------: | :-------: |
| 创建Activity A |    [A]    |
| 创建Activity A |  [A, A]   |
| 创建Activity B | [A, A, B] |

### 单一实例模式singleTask

该模式允许一个task中运行多个实例，但是同一个Activity只能被创建一次

若该Activity不存在，将会正常创建Activity

若该Activity存在，则会先执行该Activity的onDestroy()回调，重新创建该Activity，期间会执行onNewIntent()方法来重新恢复数据

🌰例子：

|      操作      | task栈 |
| :------------: | :----: |
| 创建Activity A |  [A]   |
| 创建Activity A |  [A]   |
| 创建Activity B | [A, B] |
| 创建Activity A | [B, A] |

### 单一栈顶模式singleTop

该模式允许一个task中运行多个实例，允许同一个Activity创建多次，但是不能「叠加」

- 若该Activity不为栈顶（不存在或存在但不为栈顶实例）：将会正常创建Activity

- 若该Activity为栈顶：则会调用该Activity的onNewIntent()方法，并将对应的intent参数传递给该实例，而不会创建一个新的实例

🌰例子：Activity A和B都是singleTop模式

|      操作      |  task栈   |
| :------------: | :-------: |
| 创建Activity A |    [A]    |
| 创建Activity A |    [A]    |
| 创建Activity B |  [A, B]   |
| 创建Activity A | [A, B, A] |

### 单一实例模式singleInstance

该模式要求一个Task中只能存在一个Activity的Instance实例

该模式一般用于拉起登录界面。系统会默认响应主Task，所以即使创建了一个新的Task，但是从桌面点击应用图标进入时仍然会调起主Task

🌰例子：(B声明为单一实例模式)

|      操作      |   task栈A    | task栈B |
| :------------: | :----------: | :-----: |
| 创建Activity A |     [A]      |   []    |
| 创建Activity B |     [A]      |   [B]   |
| 创建Activity C |    [A, C]    |   [B]   |
|    按下Back    |     [A]      |   [B]   |
|    按下Back    | [] (栈A销毁) |   [B]   |
|    按下Back    |      []      |   []    |

## 使用Intent标记

在调用「startActivity()」时通过intent传递一个启动模式标记

### FLAG_ACTIVITY_NEW_TASK

该模式与「singleTask」模式一致，如果启动的Activity已经存在，则会把该实例转到前台并恢复最后的状态

### FLAG_ACTIVITY_SINGLE_TOP

该模式与「singleTop」模式一致，如果启动的Activity已经时栈顶实例，则会调用onNewIntent()方法，而不会创建新的实例

### FLAG_ACTIVITY_CLEAR_TOP

如果启动的Activity已经存在，这回销毁该实例上方的所有其他实例，使得该实例恢复到栈顶位置

该标记通常和FLAG_ACTIVITY_SINGLE_TOP标记共同使用，保证只启动一个实例的同时，对栈顶元素传递intent信息

# 生命周期实例

如果不做说明，则默认探究默认模式下的Activity生命周期变化

## 同一进程中的Activity启动的生命周期

同一进程中Activity A启动Activity B时的生命周期如下：

1. Activity A中onPause()执行
2. Activity B中onCreate()、onStart()、onPause()顺序执行，B成为前台
3. Activity A中onStop()执行

## 横竖屏切换时的生命周期

无论是竖屏切换到横屏还是反之，都会执行完整的生命周期

结合使用ViewModels和onSaveInstanceState()方法或其他持久化方法可以保持页面数据

onPause() -> onStop() -> onDestroy() -> onCreate() -> onStart() -> onResume()

## 多窗口切换时的生命周期

当应用为Android7.0以上支持多窗口时，对分屏窗口的不同进程A和B切换时若从窗口B切换到窗口A，生命周期如下：

1. B中Activity的onPause()执行
2. A中的Activity的onResume()执行

注意，并不会进入到onStop状态

## Activity或对话框显示在前台的生命周期

若有新的Activity或对话框显示在前台，但是并没有完全覆盖上一个Activity时，被覆盖的实例会**只进入onPause状态**，但其重新回归前台获取焦点时，也只需要调用onResume()

若新的Activity完全覆盖住了上一个Activity，则被覆盖的实例会依次进入onPause和onStop状态。再次回归前台则需要调用onRestart()、onStart()、onResume()

## 手机锁屏时的生命周期

默认竖屏锁定后，不考虑上述说明的情况

1. 锁定手机：onPause() -> onStop()
2. 解锁手机：onRestart() -> onStart() -> onResume()

## 切换到桌面的生命周期

1. 切换到桌面：onPause() -> onStop()
2. 切换回应用：onRestart() -> onStart() -> onResume()

*可以把切换桌面等价于一个Activity显示在前台，完全覆盖了上一个Activity的行为*

## 用户按下”返回“按钮的生命周期

考虑到用户按下「返回」按钮意味着不在想回到当前Activity中，因此不会触发onSaveInstanceState()回调，而是直接顺序进入到onDestroy状态，但也可以在onBackPressed()方法中对返回按钮实现某些的自定义行为