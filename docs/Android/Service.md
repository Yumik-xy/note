# 服务的类型

## 前台服务

前台服务通常用于执行一些需要与用户进行交互的操作，因此前台服务拥有较高的权级，其可以长时间保持运行，直到用户主动停止

例如网易云在音乐播放时，会在任务窗口显示一个类似于通知的播放条，这是**前台服务**的象征

## 后台服务

后台服务执行用户不会直接注意到的操作，该操作通常是短暂的

例如保存数据至数据库时，会在后台进行数据库的读写操作而无需用户进行干涉，并会在操作完成后结束这个服务

## 绑定服务

绑定服务是调用`bindService()`执行的绑定状态，该服务使得前台/后台服务可以与组件进行数据的交互，甚至可以使用 IPC 执行跨进程之间的数据通信

该服务自绑定时运行，在所有绑定解除后立即销毁，但对于长期运行的服务，绑定和解除绑定并不会影响服务的正常运行

!> 🧨**注意：**服务仍然使用的是`主线程`，我们必须在 Service 内部主动创建子线程，否则可能会阻塞主线程



### 创建一个 Service

如果要创建服务，必须创建一个继承于`Service`的子类：

```kotlin
class MyService : Service() {
  override fun onBind(intent: Intent): IBinder {}
}
```

#### onBind() 方法

该方法时继承 Service 类后唯一需要实现的抽象方法，当另一个组件想要与服务绑定时，系统会通过调用 `bindService()` 来调用此方法

如果不想进行绑定，官网中给出如下解释：

> 在此方法的实现中，您必须通过返回 `IBinder` 提供一个接口，以供客户端用来与服务进行通信请务必实现此方法
>
> 但是，如果您并不希望允许绑定，则应返回 null

查看源码可以发现：

```java
// Service.class
@Nullable
public abstract IBinder onBind(Intent var1);
```

该函数是允许进行空返回的，故使用 Kotlin 编程时可以将 `IBinder` 自行设置为 `IBinder?` 

#### onStartCommand() 方法

单一个组件使用 `startService()` 加载 Service 时，便会执行该方法

此时该服务可以在后台无限期运行，只有进程被关闭或主动调用 `stopSelf()` 或 `stopService()` 方法才能停止服务

如果只需要使用绑定的方式运行，则不会加载该方法，因此也不需要实现

#### onCreate() 方法

只有首次创建服务时，系统会在调用 `onStartCommand()` 或 `onBind()` 之前调用此方法

#### onDestroy() 方法

当服务所有绑定都解除或是即将被销毁时，便会执行该方法



### 声明一个 Service

每一个 Service 组件都必须在 `AndroidManifest.xml` 中进行声明

```xml
<manifest ... >
  ...
  <application ... >
      <service
					android:name=".MyService"
					android:enabled="true"
					android:exported="false"
          android:description="description"/>
      ...
  </application>
</manifest>
```

>1. 可以将 `android:exported ` 设置为 `false` 表示该服务仅允许本程序使用，如果设置为 `true` 则其他应用程序也可以调用该服务（如：网易云的播放服务可以在不打开网易云的情况下被运行）
>2. `android:description`z 指该服务的作用描述，当服务在后台运行时用户查看所有服务信息时，会显示该描述内容



### 启动一个 Service

#### 启动服务

Activity 可以结合使用显式 Intent 与 `startService()` 启动服务

```kotlin
Intent(this, MyService::class.java).also { intent ->
    startService(intent)
}
```

#### 停止服务

Activity 可以结合使用显式 Intent 获取服务对应的 ID 来结束服务

```kotlin
Intent(this, MyService::class.java).also { intent ->
    stopService(intent)
}
```

或者在服务内部使用 `stopSelf()` 自行停止



### 绑定一个 Service

绑定服务的目的即实现 Activity 和 Service 的通信，因此必须要实现 `onBind()` 

例如，以下服务可让客户端通过 `Binder` 实现访问服务中的方法：

```kotlin
class LocalService : Service() {
    // Binder given to clients
    private val binder = LocalBinder()

    // Random number generator
    private val mGenerator = Random()

    /** method for clients  */
    val randomNumber: Int
        get() = mGenerator.nextInt(100)

    /**
     * Class used for the client Binder.  Because we know this service always
     * runs in the same process as its clients, we don't need to deal with IPC.
     */
    inner class LocalBinder : Binder() {
        // Return this instance of LocalService so clients can call public methods
        fun getService(): LocalService = this@LocalService
    }

    override fun onBind(intent: Intent): IBinder {
        return binder
    }
}
```

`LocalBinder` 为客户端提供 `getService()` 方法，用于检索 `LocalService` 的当前实例

这样，客户端便可调用服务中的公共方法。例如，客户端可调用服务中的 `getRandomNumber()`

点击按钮时，以下 Activity 会绑定到 `LocalService` 并调用 `getRandomNumber()`

```kotlin
class BindingActivity : Activity() {
    private lateinit var mService: LocalService
    private var mBound: Boolean = false

    /** Defines callbacks for service binding, passed to bindService()  */
    private val connection = object : ServiceConnection {

        override fun onServiceConnected(className: ComponentName, service: IBinder) {
            // We've bound to LocalService, cast the IBinder and get LocalService instance
            val binder = service as LocalService.LocalBinder
            mService = binder.getService()
            mBound = true
        }

        override fun onServiceDisconnected(arg0: ComponentName) {
            mBound = false
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.main)
    }

    override fun onStart() {
        super.onStart()
        // Bind to LocalService
        Intent(this, LocalService::class.java).also { intent ->
            bindService(intent, connection, Context.BIND_AUTO_CREATE)
        }
    }

    override fun onStop() {
        super.onStop()
        unbindService(connection)
        mBound = false
    }

    /** Called when a button is clicked (the button in the layout file attaches to
     * this method with the android:onClick attribute)  */
    fun onButtonClick(v: View) {
        if (mBound) {
            // Call a method from the LocalService.
            // However, if this call were something that might hang, then this request should
            // occur in a separate thread to avoid slowing down the activity performance.
            val num: Int = mService.randomNumber
            Toast.makeText(this, "number: $num", Toast.LENGTH_SHORT).show()
        }
    }
}
```



