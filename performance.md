## SparseArrayCompat
1. `SparseArray`用于存放键值对，键是`int`，值是`Object`。
2. `SparseArray`用两个长度相等的数组分别存储键和值，同一个键值对所在两个数组中的索引相等。
3. `SparseArray`比`HashMap`访问速度更慢，因为二分查找速度慢于散列定位。
4. `SparseArray`比`HashMap`更节省空间，因为不需要创建额外的`Entry`存放键值对。
5. `SparseArray`中存放键的数组是递增序列。
6. `SparseArray`删除元素时并不会真正删除，而是标记为待删除元素，在合适的时候会将后面的元素往前挪覆盖掉待删除元素。待删除元素在没有被覆盖前有可能被复用。


## 匿名共享内存
跨进程通信时，数据量大于1MB要怎么传递呢？用匿名共享内存（Ashmem）是个不错的选择，它不仅可以减少内存复制的次数，还没有内存大小的限制。

多个进程的用户空间映射到同一块内核空间，以此方式完成进程通信

android使用匿名共享内存的主要是4个步骤：

通过 MemoryFile / SharedMemory开辟内存空间，获得 FileDescriptor
将 FileDescriptor 传递给其他进程 （Binder驱动通过当前进程的fd找到对应的文件，然后为目标进程新建fd，并传递给目标进程，不同进程的fd不是同一个）
往共享内存写入数据
从共享内存读取数据


## protobuf
sint32 使用了一种称为“变长编码（Variable-Length Encoding，简称 VLE）”的技术，可以将较小的负数编码成较短的二进制数，从而减少存储空间。例如，-1 可以被编码为 0b0001，-127 可以被编码为 0b10000001，-128 可以被编码为 0b10000000 0b00000001。在 protobuf 编码中，sint32 会和 zigzag 编码一起使用，进一步压缩负数的存储空间。

而 int32 则采用固定长度的编码方式，无论正数还是负数，在存储时都使用 32 个比特位。这使得 int32 的编码和存储方式相对简单，但对于存储较多的负数时，可能浪费掉一些存储空间。

综上所述，对于需要存储较多负数的情况，建议使用 sint32 类型，可以更好地降低存储空间的消耗。而对于存储正数较多的情况，int32 类型可能更适合，可避免进行变长编码。


# app流畅度监测
## 帧率
- 全帧率监控：Choreographer.FrameCallback 被回调时，doFrame 方法都带上了一个时间戳，计算与上一次回调的差值，就可以将之视之为一帧的时间。当累加超过 1s 后，就可以计算出一个 FPS 值。我们每一次回调后，都需要对 Choreographer 进行 postFrameCallback 调用，而调用 postFrameCallback 就是在下一帧 CALLBACK_ANIMATION 类型的链表上进行添加一个节点。所以，doFrame 回调时机并不是这一帧开始计算，也不是这一帧上屏，而是 CPU 处理动画过程中的一个 callback。
- 过滤空闲帧率监控：上述方案在主线程无活动时也会不停地注册下一个vsync信号。解决方案是监听消息队列（setMessageLogging()）当遇到异步消息时（doFrame()）通过反射向Choreographer注入一个callback（调用postFrameCallback会注册下一个vsync信号）。
- 滑动帧率：用户产生交互的时候的帧率更有价值，通过ViewTreeObserver.OnScrollChangedListener，标记界面是否存在滚动，只有当存在滚动时才postFrameCallback
- 官方方案：在 Android 7.0 之后，官方已经引入了 FrameMetrics API 来提供帧耗时的详细数据。androidx 的 JankStats 库主要就是基于 FrameMetrics API 来实现的帧耗时数据统计。window.addOnFrameMetricsAvailableListener()

## hitchRate
- 超出帧应该渲染的时间/滚动耗时 = hitchRate
- 不同手机的刷新率不同，导致帧率不同，而且有些场景帧率会失真，比如动画过程中有停顿。但我们总是可以让hitchRate 为0

## app 进程启动
通过加载一个空的ContentProvider实现近似的记录进程启动时间

## 第一个界面显示时机
android 10 之前通过在onWindowFocusChanged()打点（无侵入的方式为ViewTreeObserve.onWindowFocusChange()），android 10 之后在onResume()打点，10以后第一帧提前了(无侵入方式为registerActivityLifecycleCallbacks的onActivityResume()的时候抛一runnable到主线程)

## 内存优化
1. 找出内存消耗的大头，通常是内存泄漏，或者大图
2. 内存泄漏可以通过profile，手动触发一些gc，看是否有该被回收而未被回收的对象，或者leakcanary
3. 避免大图做内存缓存，只做磁盘缓存
4. ViewPager场景下，当页数很多时，使用FragmentStatePagerAdapter，因为他只会缓存邻近的两页
5. sp会将文件内容加载到内存，可能占用大内存
6. 合理RecyclerView缓存长度：广场列表每个itemView很长，同一屏幕最多显示3个。默认的每个type缓存5个itemView浪费，调整到3个。
7. 没有必要的缓存：多个tab共用一个RecyclerView，当切换tab时会拉新数据
8. 冗余字段治理：PostBean本身就已经有100多个字段，各个不同的业务场景还会添加额外字段。抽取基本共用字段，不同业务界面通过持有基本字段实现复用。
8. 冗余控件治理：会inflate没有用的控件，通过数据手动地将其展示或者不展示
7. 内存泄漏：
	- 帖子详情页，把所有的控件都写到xml中，然后通过visibility控制有些显示，有些不显示，但所有的控件都会被加载，播放视频的控件在被加载的时候会设置一些MediaPlayer的监听器，但在退出详情页的时候没有调用release，导致Activity泄漏
	- 广场标签弹窗泄漏：它会观察一个LiveData，LiveData绑定的生命周期对象是Activity
7. glide缓存降级：RecyclerView缓存表项，表项持有图片，图片使用Glide加载，这些图片会被ActiveResource持有，直到发生gc。通过在onViewRecycled()主动调用Glide.clear(),释放掉ActiveResource。
7. Svga优化
	- 播放多个动画时会发生内存抖动，因为在播放动画之前会将所有的帧都解析成bitmap存在hashMap结构中。解析的过程没有bitmap的复用，解决方案构建BitmapPool，用LinkedHashMap缓存一组Bitamp（按大小缓存）。实现lru淘汰策略。
	- 当在RecyclerView中加载svga动画时，如果表项不可见时清理动画持有的bitmap，则表项再次进入需要重新解析再能播放。如果不清理因为RecyclerView的缓存策略，则会导致内存飙升。优化策略是清理掉动画持有的Bitmap，把它降级为弱引用，当表项再次出现时，先从弱引用加载，若没有则从磁盘文件加载。


## 内存优化
USS = Unique Set Size = 进程独占的内存
PSS = Proportional Set Size = USS+ 按比例包含共享库
RSS = Resident Set Size = RSS= USS+ 包含共享库
VSS = Virtual Set Size = VSS= RSS+ 未分配实际物理内存
1. 图片加载优化
	1. 无侵入本地图片压缩：在 gradle 编译期间插入一个图片压缩任务，McImage 这个而库 
	2. 网络大图监控：hook Glide 的 SingleRequest，注册 requestListener 监听图片加载	
	3. 监控 ImageView.setImageDrawable()，修改字节码将ImageView替换为自定义ImageView，计算图片大小（可以在IdleHandler中做），如果大于控件大小，在debug弹窗，release上报后台
2. 内存泄漏监控
	1. 堆内存超过阈值，线程数超过阈值，内存增长过快时进行dump内存
	2. 过 Debug.MemoryInfo 的 getMemoryStat 方法（需要 23 版本及以上），我们可以获得等价于 Memory Profiler 默认视图中的多项数据
	2. dump 出内存镜像（fork 出新进程进行），上传到后台进行分析 	

## 分析OOM
- 关键要找出OOM时，什么东西占用内存最多，为什么占用这么多。使用android monitor dump hprof---转换成标准hprof 使用mat，在dump之前需要手动gc一下，排除软引用和弱引用的情况
- shallow heap的大小指的是该对象在没有引用其他对象的情况下本身占用的内存大小。一个普通对象的shallow heap 的大小（不包括数组类型）依赖于它含的方法，元素的大小。而一个数组类型的shallow heap的大小则依赖于数组的长度和数组里面元素的类型。集合类型的shallow heap的大小则指的是集合所包含的所有对象的大小的总和
- retained heap是指对象自己本身的shallow heap的大小加上对象所引用的对象的大小。换句话说retained heap的大小是指该对象被回收时垃圾回收器应该回收的内存的大小。当shallow heap远小于retained heap时 说明对象持有了大量引用
- Dorminator Tree视图以对象的视角，它主要可以用于诊断一个对象所占内存为什么会不断膨胀，一个对象膨胀，就说明它对应到支配树中的子树就越来越庞大，只要分析这个对象对应的子树，确定那些对象是不应该出现在子树中就可以对问题手到病除
- histogram以类的视角视图可以方便的看到对象个数，如果对象个数不合常理的多，可能就是泄露所在
	- List Objects---with incoming refrences和with outcoming refreences，分别会罗列出选定对象在引用树中的父节点和子节点，继续展开子结点也是对应的incomming和outcoming，如果两个对象存在循环引用，你需要仔细分析，排除循环引用的路径，否认则层层展开的内容都是一样的且无限循环。可以通过不断展开来找到gc root
	- Merge Shortest Path To GC Roots：从gc root到指定对象的最短路径（gc root在最上面，指定对象在最下面）
- OQL：将内存中的每一个不同的类当做一个表，类的对象则是这表中的一条记录，每个对象的成员变量则对应着表中的一列	
	- select * from instanceof android.app.Activity
- 8.0之后 oom率明显下降，因为bitmap的内存在native上分配
- oom种类
	+ 线程太多导致pthread_create failed，不同rom对threads-max 不一样
		* 通过 cat proc/{pid}/status 查看当前进程的线程数
		* 1. 使用字节码插桩在编译期间动态地将 new Thread 替换为自定义线程，重写 start 将其放到线程池中执行
		* 2. 优化线程池，允许核心线程超时销毁
		* 3. 内存监控，hook 线程生命周期函数（pthread_create/pthread_exit），当发现一个joinable的线程在没有detach或者join的情况下，执行了pthread_exit，则记录下泄露线程信息，当泄漏时间打到阈值后上报线程信息
	* 创建文件太多，too many open file
	* 堆内存溢出：大图优化，监控内存泄漏

## 内存泄漏
内存泄漏是因为堆内存无法释放
android内存泄漏就是生命周期长的对象持有了生命周期较短对象的引用
1. 静态成员变量（单例）
	- 静态成员变量的生命周期和整个app一样，如果它持有短生命周期的对象则会导致这些对象内存泄露
	- 静态变量在不使用时需要置空
	- 静态变量使用弱引用持有Activity
	- 单例持有App context而不是Activity Contex
	- 静态方法可以被子类隐藏，而不是重写
2. 非静态内部类
	- 匿名内部类持有外部类引用
	- handler是典型的匿名内部类，handler中的消息持有handler引用，如果有未处理完的消息，则会导致handler外层类内存泄露，Looper -> MessageQueue -> Message -> Handler -> Activity，解决办法是静态内部类+Activity弱引用，并且在activity退出时清除所有消息
	- new Thread()是典型的匿名内部类，如果Activity退出后Thread还在执行则会引起Activity内存泄露
3. 集合类
 	- 集合对象会持有孩子的引用，需要及时清除且置空
4. webview内存泄露
	- WebView在加载网页后会长期占用内存而不能被释放，因此我们在Activity销毁后要调用它的destory()方法来销毁它以释放内存

## app有限的内存
1. 使用内存友好的数据结构 SpareseArray，ArrayMap
2. 避免内存泄漏，避免长生命周期对象持有短生命周期对象
3. 使用池结构，复用对象避免内存抖动。
4. 根据手机内存大小，设置内存缓存的大小。
5. 多进程，扩大可使用内存。
6. 通过ComponentCallback2 监听内存吃紧，进行内存缓存的释放。

## anr分析
KeyDispatchTimeout：5秒内未响应输入事件,只有当新的点击事件到来时，发现超时事件才会触发anr
BroadcastTimeout：广播onReceive()函数运行在主线程中，在特定的时间（10s）内，后台广播60s无法完成处理。在onReceive() 中 sleep会timeout
ServiceTimeout：前台服务20秒 ，后台服务200s,在service.onCreate()，onstart,onbind,onrebind,onUnbind()中sleep都会会anr,ActiveServices 埋下炸弹，当ActivityThread中handler收到CREATE_SERVICE消息时,拆炸弹再回调service.oncreate()
ContentProvider：publish 在10秒内未完成,所以在provider.onCreate() 中sleep 不会anr
activity生命周期回调中sleep不会发生anr，但是此时触发触摸事件就会发生anr
- 监听anr ：当一个进程发生ANR时，则会收到SIGQUIT信号。如果，我们能监控到系统发送的SIGQUIT信号，也许就能感知到发生了ANR，除Zygote进程外，每个进程都会创建一个SignalCatcher守护线程，用于捕获SIGQUIT、SIGUSR1信号，并采取相应的行为。
- 过滤anr：并不是所有SIGQUIT 信号都代表anr，可以通过Looper的mMessage对象，该对象的when变量，表示的是当前正在处理的消息入队的时间，我们可以通过when变量减去当前时间，得到的就是等待时间，如果等待时间过长，就说明主线程是处于卡住的状态。这时候收到SIGQUIT信号基本上就可以认为的确发生了一次ANR
- 分为前台anr和后台anr：前台ANR会弹出无响应的Dialog，后台ANR会直接杀死进程
- anr的原因在规定时间内没有完成指定任务，anr监控是通过延迟消息埋炸弹，拆炸弹，没拆成功则引爆
- anr触发原点ActivityManagerService.appNotResponding()，ams会发送消息到主线程消息队列触发弹窗.系统进程会发signal给应用进程触发器dump堆栈。监听signal然后检测主线程消息队列头部消息的when字段（表示该消息应该被消费的时间，如果与当前时间差距很多，表示主线程阻塞）
- 信息采集
	+ 原因：通过AcivityManager获取ANRInfo其中包含shortMsg( shortMsg: ANR Inputdispatching timed out ...)可以知道anr原因
	+ 日志：
	`String cmd = "logcat -f " + file.getAbsolutePath(); Runtime.getRuntime().exec(cmd);`
- data/anr/trace文件：发生anr的调用栈
- 如果非anr线程cpu使用率非常高，则可能是因为没有被分配cpu执行时间导致anr
- 如果anr进程cpu使用率非常高，则可能是因为不合理代码占用cpu资源
- IOwait很高时说明有io操作
- BlockCanary 利用主线程消息队列的日志打印监控卡顿，通过Looper.setMessageLogging(printer)监听每个消息的打印，分发消息前和消息处理完毕后都会打印，blockCanary会计算这两次打印的时间差，如果时间差超出阈值则发消息到另一个线程入捕获现场信息（调用栈，cpu信息），设置 Printer 的方案，随着 Looper 的 for 循环不断执行，会导致字符串频繁拼接，产生大量临时对象，有可能加重 App 的 GC 频率，导致内存抖动
- BlockCanary 
	+ 不能检测到取消息的卡顿，只能检测到处理消息的卡顿，
	+ 不能检测到 IdleHandler.queueIdle() 卡顿，因为只能监听消息处理，idleHandler 是在没有消息处理时，才会触发，通过装饰者模式包装一些IdleHandler接口（或者通过修改字节码，将所有IdleHandler换成自定义的），在原有逻辑外面包一层监控逻辑（通过向ThreadHandler抛一个延迟处理消息用于打印调用栈，带queueIdle逻辑执行完后移除消息）
	+ 无法监听触摸事件的卡顿，因为触摸事件捕获的源头是在system server 进程的 inputDispatcher 线程，它通过和应用进程通信完成触摸事件的分发（socket 通信），卡顿可能是因为应用进程未收到触摸事件。可以通过 PLT hook socket 通信的过程（sendto和recvfrom函数）
	+ 不能检测同步消息泄漏：同步消息添加和移除是成对的，可能会存在同步消息未被移除的情况（一次插入了两个屏障，异步消息被移除导致无人移除屏障）。通过子线程是不是地向主线程 发一个同步和异步消息（都发一次，因为可能遇到正在渲染），判断他们是否被执行，如果异步执行了，同步没执行，则说明有遗漏的同步消息屏障。
- Choreographer：为~设置FrameCallback，在每一帧的绘制时都会回调，以织入检测绘制卡顿的逻辑（也是通过向 HandlerThread 抛一个延迟消息）

## 卡顿检测方法
 - `adb shell dumpsys gfxinfo 包名`展示整体卡顿严重度（卡顿帧数占比）

## Bitmap
- 8.0 之后，像素数据在native层分配内存，导致java堆增长速度慢，但native增长速度快。
- 系统通过NativeAllocationRegistry，解决了在java对象释放的时候自动释放它引用的native对象，当java堆或者native发生溢出时都会触发native的gc

## leakCanary
- 通过ActivityLifecycleCallbacks监听Activity生命周期，在onActivityDestroy时获取Activity实例，并为其构建弱引用并关联引用队列。
- 起异步线程，观察ReferenceQueue是否有Activity的弱引用，如果有则说明回收成功，否则回收失败
- 回收失败后会手动触发一次gc，再监听ReferenceQueue，如果还是回收失败，则dump内存
- LeakCanary 通过contentProvider安装
- 当一个Activity的onDestory方法被执行后，说明该Activity的生命周期已经走完，在下次GC发生时，该Activity对象应将被回收

## 监听gc
构建一个 WeakReference，再构建一个自定义对象塞给他，重写该对象的 finalize方法，该方法执行的时候就是发生gc的时候
```
public class GCCheck {
    private WeakReference<GCOwer> rf = new WeakReference<>(new GCOwer());

    public class GCOwer {

        @Override
        protected void finalize() throws Throwable {
            super.finalize();
            Log.i("GCCheck", "finalize: app gc occur");

            rf = new WeakReference<>(new GCOwer());
        }
    }
}
```


## 线上oom
1. 在oom，或者内存激增，或者内存快触顶的时候dump内存获取hprof文件。
2. dump内存是阻塞操作，要放到子进程做，通过fork主进程，就得到了主进程的内存拷贝。然后在主进程通过fileObserver异步等待hprof文件生成

## oom兜底
1. 通过ActivityLifecycleCallbacks监听Activity生命周期，在onDestroy中将Activity添加到弱引用，并监听引用队列，如果30s之后activity还是没有被回收，则主动释放activity中持有的图片。
2. 全局缓存需要实现统一接口，并且监听ComposeCallback2，当内存吃紧的时候，释放一部分缓存

## OOM类型
1. 堆内存不足
2. 无足够的连续内存空间
3. 文件描述符超过数量限制
4. 线程数量超过限制
5. 虚拟内存不足

## 绘制性能
布局嵌套性能差是因为父控件有可能多次测量孩子，嵌套层次对性能的影响是指数级的（父亲是wrap_content,孩子是match_parent，父亲先以0强制测量下孩子，然后选取孩子中最宽的那个最为自己宽度，然后再去测量下孩子），compose只允许测量一次（固有特性测量，先查询子项，然后在进行实际测量）


## 缩包
1. 只保留一张3倍图（在低分辨率手机上图片会根据像素密度裁剪）
2. 所有的图片webp化（5mb）
3. shape+layout资源 dsl 化（1mb）

# 启动优化
冷启动 & 温启动 & 热启动
- 温启动是指进程还在，但是Activity需要重新执行onCreate()，所以主界面的 onCreate 影响温启动速度
- 热启动是指进程和activity实例都在，只需要执行 Activity.onStart()
启动耗时统计：
```
tangliang@SH-tangliang tools % adb shell am start -S -W com.snda.lantern.wifilocating/com.wifitutu.ui.launcher.LauncherActivity
Stopping: com.snda.lantern.wifilocating
Starting: Intent { act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] cmp=com.snda.lantern.wifilocating/com.wifitutu.ui.launcher.LauncherActivity }
Status: ok
Activity: com.snda.lantern.wifilocating/com.wifitutu.ui.launcher.LauncherActivity
ThisTime: 2721
TotalTime: 2721
WaitTime: 2742
Complete
```


## 冷启动优化
初步显示所用时间 (TTID) 和完全绘制所用时间 (TTFD)。TTID 是显示第一帧所用的时间，TTFD 则是应用达到可全面互动的状态所用的时间

冷启动 app 进程过程：
1.bindApplication 是整个启动过程交给应用程序的起点。
2.attachBaseContext():最早的预加载时机
3.installProvider()
4.Application.onCreate()

IO Wait 指发生了 IO 操作需要等待 IO 返回结果

1. 视觉优化：windowBackground设置一张图片（成为StartingWindow的Decorview的背景）当应用启动时，空白启动窗口将保留在屏幕上，直到系统首次完成应用绘制。此时，系统进程会切换应用的启动窗口，让用户与应用互动。
2. 初始化任务优化：可以异步初始化的，放异步线程初始化，必须在主线程但可以延迟初始化的，放在IdleHandler中，
3. ContentProvider 优化：去掉没有必要的contentProvider，或者将多个ContentProvider通过startup进行串联成一个，这样可以减少contentprovider对象创建的耗时，ContentProvider 即使在没有被调用到，也会在启动阶段被自动实例化并执行相关的生命周期。在进程的初始化阶段调用完 Application 的 attachBaseContext 方法后，会再去执行 installContentProviders 方法，对当前进程的所有 ContentProvider 进行 install
4. 缩小main dex：MultidexTransform 解析所有manifest中声明的组件生成manifest_keep.txt，再查找manifest_keep.txt中所有类的直接引用类，将其保存在maindexlist.txt中，最后将maindexlist.txt中的所有class编译进main.dex。multiDex优化，自行解析AndroidManifest，自定义main.dex生成逻辑，将和启动页相关的代码分在主dex中，减小主dex大小，加快加载速度。
5. multiDex.install 优化：
	- 4.x使用Dalvik 虚拟机，所以只能执行经过优化后的 odex 文件，为了提升应用安装速度，其在安装阶段仅会对应用的首个 dex 优化成odex并加载到PathClassLoader的dexPathListz中。对于非首个 dex 其会在首次运行调用 MultiDex.install 进行优化并加载到classloader中，而这个优化是非常耗时的，这就造成了 4.x 设备上首次启动慢的问题，MultiDex.install()耗时。它会先解压apk，遍历其中的dex文件，然后压缩成对应的zip文件（这是第一次的逻辑，第二次启动时已经有zip文件则直接读取。），然后对zip文件进行odex优化，最终返回一个DexFile，再将dexfile追加到PathClassLoader的pathList尾部。将install异步化，多线程话，改变加载流程，使得首次不需要加载odex而是直接加载dex，先正常启动，然后后台起一个单独进程慢慢做odex优化
	
# 页面启动优化
1. 布局异步加载
2. 接口预请求--在路由跳转之前添加拦截进行预请求
3. 图片预请求，分为url已知，比如feeds流到详情页，需要加载图片的url是一样的，只是尺寸不同

