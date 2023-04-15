# window创建过程
- Activity.attach()时创建了PhoneWindow，每个activity有一个PhoneWindow

# DecorView
- DecorView创建：调用setContentView()中委托给PhoneWindow，并在其中创建DecorView实例并且会根据主题解析预定义的布局文件，并将其作为decorview 的孩子，布局文件中必然有一个id 为 content的容器控件赋值给 contentParent，他就是setContentview的父亲
- DecorView和ViewRootImpl绑定：在handleResumeActivity()中会先将decorview invisible将DecorView添加到窗口，接着创建ViewRootImpl并将DecorView和其绑定，触发requestLayout遍历View树。添加成功后再调用makeVisible 才显示出来
- Activity层级关系：Activity（PhoneWindow（DecorView（statusBar+titleBar+ContentView+navigationBar））），setContentView就是替换ContentView的内容

# ViewRootImpl
- 是 ViewParent，即DecorView的父亲，它实现了View和WindowManagerService间通信协议（IWindow），即ViewRootImpl响应 wms 发送过来的指令，响应方式是发送消息到ViewRootHandler（主线程）
- 是在DecorView被添加到窗口的时候（wm.addView）被 WindowManagerGlobal 新建的（onresume之后）
- ViewRootImpl 在构造的时候会构建和wms的双向通信通道，IWindowSession+IWindow。
- 一个 ViewRootImpl 持有一个 surface对象，对应SurfaceFlinger中的layer对象，有buffer queue的生产者指针，和消费者指针，Surface通过向生产者指针写，然后SurfaceFlinger通过消费者指针读，将内燃渲染到屏幕
- View树遍历：requestlayout()即是触发View树遍历，先向主线程消息队列抛同步屏障，然后将View树遍历包装成一个 Runnable抛给编舞者，编舞者注册接收下一个vsync，然后将View树遍历任务缓存在一个链式数组中，待下一个vsync到来之后向主线程抛异步消息，当消息被执行时会将到期的所有任务都从链上摘下并执行。
- 子线程更新ui：可以实现，需要绕过viewrootImpl的检查，比如在onCreate()中启动异步线程更新ui，onresume之后viewrootimpl才被构建。或者使用surfaceview

# 将View添加到窗口过程
- 是一个app和wms，以及wms和sf双向通信的过程：
	1. app将view添加到window(委托给WindowManagerGlobal，它是一个单例，每个进程只有一个，WindowManagerGlobal构建了ViewRootImpl并调用了setView(){在mWindowSession.addToDisplay()之前requestLayout()})
	2. wms登记窗口（形成windowState），wms接到请求后去sf申请Layer，sf将结果回传给wms，wms再传给app，app就能访问surface指向的内存，是一个buffer队列，app是生产者，sf是消费者，app可以像其中填充内容，然后通知sf调用图形接口渲染到屏幕。（其中app和wms双向通信通过ViewRootImpl实现）
- 只有申请了依附窗口，View才会有可以绘制的目标内存（surface）

## dialog和popupWindow和toast
- 他们都是窗口，只是类型不同，toast是系统级的窗口 而popupwindow是子窗口，dialog是应用窗口，Activity也是应用窗口
- popupwindow如果设置了背景，则会在外面套一层FrameLayout（它会拦截KeyEvent和TouchEvent，所以点击返回键或者点击window以外的区域可以让window消失 ），然后将其add到window
- 普通Dialog必须依附于Activity显示（系统的除外），Dialog构造的时候会新建一个TYPE_APPLICATION的PhoneWindow对象，这种类型的窗口必须使用一个activity token来指明窗口属于谁。Dialog和Activity使用同一个WindowManager（实例是WindowManagerImpl），然后将自定义视图添加到DecorView的mContentParent,显示时把PhoneWindow的DecorView添加到WindowManager中。
- Dialog会拦截发送给Activity的事件（TouchEvent KeyEvent）,因为他新建的PhoneWindow并setCallback()进行事件拦截
- PopupWindow没有新建窗口，所以也没有进行事件拦截。
- PopupWindow算是子窗口，必须依附到其他窗口，依附的窗口可以是应用窗口也可以是系统窗口，但是不能是子窗口
- PopupWindow自窗口中的Token是ViewRootImpl的IWindow对象
- toast和输入法都是系统窗口
- WindowManager.LayoutParams中有两个重要的参数 1. type:用于表示窗口类型 2. token：用于表示窗口分组依据，他是一个IBinder对象

## surfaceview
- 一个嵌入View树的独立绘制表面，他位于宿主Window的下方，通过在宿主canvas上绘制透明区域来显示自己
- 虽然它在界面上隶属于view hierarchy，但在WMS及SurfaceFlinger中都是和宿主窗口分离的，它拥有独立的绘制表面，绘制表面在app进程中表现为Surface对象，在系统进程中表现为在WMS中拥有独立的WindowState，在SurfaceFlinger中拥有独立的Layer，而普通view和其窗口拥有同一个绘制表面
- 因为它拥有独立与宿主窗口的绘制表面，所以它独立于主线程的刷新机制，可以在另一个线程中自定义刷新频率
- 它的背景是属于宿主窗口的绘制表面，所以如果背景不透明则会盖住它的绘制内容
- 获取到的 Canvas 对象还是继续上次的 Canvas 对象，而不是一个新的 Canvas 对象

SurfaceView做帧大图帧动画，优化内存和流畅度：
1. 异步绘制：使用 HandlerThread 作为异步渲染线程，可以借用消息机制实现帧的串行绘制，在一帧绘制的结尾postDelay
2. 逐帧解析：边解析边绘制
3. 图片复用：下一帧的解析复用上一帧已经绘制的bitmap
4. 异步解析+预解析：独立的解析线程，提前解析3帧图片
5. 滑动窗口：使用阻塞队列来适配慢速解析和快速绘制。

# TextureView
1. SurfaceView 拥有独立的绘制表面，而TextureView和View树共享绘制表面
2. TextureView通过观察BufferQueue中新的SurfaceTexture到来，然后调用invalidate触发View树重绘，如果有View叠加在TextureView上面，它们的脏区有交集，则会触发不必要的重绘，所以他的刷新操作比SurfaceView更重
3. TextureView 持有 SurfaceTexture，它是一个GPU纹理
4. SurfaceView 有双缓冲机制，绘制更加流畅
5. TextureView 在5.0之前在主线程绘制，5.0之后在RenderThread绘制。

## Surface
- 它是原始图像缓冲区的一个句柄。即raw buffer的内存地址，raw buffer是保存像素数据的内存区域，通过Surface的canvas 可以将图像数据写入这个缓冲区
- Surface类是使用一种称为双缓冲的技术来渲染
- 这种双缓冲技术需要两个图形缓冲区GraphicBuffer，其中一个称为前端缓冲区frontBuffer，另外一个称为后端缓冲区backBuffer。前端缓冲区是正在渲染的图形缓冲区，而后端缓冲区是接下来要渲染的图形缓冲区，当vsync到来时，交换前后缓冲区的指针
- 部分刷新是通过前端缓冲区拷贝像素到后端缓冲区，并且合并脏区以缩小它。
- 每个ViewRootImpl都持有一个Surface。
- 持有BufferQueue的生产者指针。

## SurfaceFlinger
- SurfaceFlinger 是由 init 进程启动的运行在底层的一个系统进程，它的主要职责是合成和渲染多个Surface，并向目标进程发送垂直同步信号 VSync,并在 vsync 产生时合成帧到frame buffer
- SurfaceFlinger持有BufferQueue消费者指针，用于从BufferQueue中取出图形数据进行合成后送到显示器
- View.draw()绘制的数据是如何流入SurfaceFlinger进行合成的？
1. Surface.lockCanvas()从BufferQueue中取出图形缓冲区并锁定
2. View.draw()将内容绘制到Canvas中的Bitmap，就是往图形缓冲区填充数据
3. Surface.unlockCanvasAndPost()解锁缓冲区并将其入队BufferQueue，然后通知SurfaceFlinger进行合成，在下一个vsync到来时进行合成（app直接喝surfaceFlinger通信）
- 应用进程通过Anonymous Shared Memory将数据传输给SurfaceFlinger，因为Binder通信数据量太小
- 手写匿名共享内存：在服务端自定义Binder实例，并在onTransact()方法中构建MemoryFile，然后反射获取FileDescriptor，并将其写入reply parcel。客户端绑定服务时，通过reply获取FileDescriptor。

## 界面卡顿
- 刷新率是屏幕每秒钟刷新次数,即每秒钟去buffer中拿帧数据的次数, 帧率是GPU每秒准备帧的速度，即gpu每秒向buffer写数据的速度,
- 卡顿是因为掉帧，掉帧是因为写buffer的速度慢于取buffer的速度（每秒60帧），每隔16.6ms显示设备就会去buffer中取下一帧的内容，没有取到就掉帧了
- 当扫描完一个屏幕后，设备需要重新回到第一行以进入下一次的循环，此时有一段时间空隙，称为VerticalBlanking Interval(VBI), Vsync就是 VBI 发生时产生的垂直脉冲,这是最好的交换双缓冲的时间点,交换双缓冲是交换内存地址，瞬间就完成了
- 一帧的显示要经历，cpu计算画多大，画在哪里，画什么，然后gpu渲染计算结果存到buffer，显示器每隔一段时间从buffer取帧。若没有取到帧，只能继续显示上一帧。
- VSYNC=Vertical Synchronization垂直同步，它就是为了保证CPU、GPU生成帧的速度和display刷新的速度保持一致
- VSYNC信号到来意味着,交换双缓冲的内存地址,font buffer 和 back buffer 互换,这个瞬间 back font就供下一帧使用, project butter(4.1)以后, Vsync一到就 GPU和cpu就开始渲染下一帧的数据
- 双缓冲机制：用两个buffer存储帧内容，屏幕始终去font buffer取显示内容，GPU始终向back buffer存放准备好的下一帧，每个Vsync发生吃就是他们交换之时,若下一帧没有准备好,则就错过一次交换,发生掉帧
- 三缓冲机制：为了防止耗时的绘制任务浪费一个vsync周期，用三个buffer存储帧。当前显示buffer a中内容，正在渲染的下一帧存放在buffer b中，当VSYNC触发时，buffer b中内容还没有渲染好，此时buffer a不能清除，因为下一帧需要继续显示buffer a，如果没有第三个buffer，cpu和gpu要白白等待到下一个VSYNC才有可能可以继续渲染后序帧
- View的重绘请求最少要等待两个vsync 才能显示到屏幕上：会包装成一个runnable,存放在Choreographer中,下一个Vsync到来之时,就是它执行之时,执行完毕,形成的帧数据会放在back buffer中,等到下一个Vsync到来时才会显示到屏幕上
1.是否存在overdraw：某个像素在一帧的渲染时间内被绘制了多次---减少层级，去掉不必要的背景
2.布局层次太深：增加measure和layout时间---使用constraintlayout viewstub（高度为0的不可见的View）
3.耗时函数：函数执行时间，除了渲染代码之外是否有别的太消耗cpu的逻辑，占用了cpu渲染的时间

# 同步屏障
- ViewRootImpl 将遍历view树包装成一个Runnable并抛到编舞者， 在抛之前会向主线程消息队列中抛同步屏障
- 同步屏障也是一个Message，只不过 target 等于null
- 取下一条message的算法中，若遇到同步屏障，则会越过同步消息，向后遍历找第一条异步消息找到则返回（编舞者抛的异步消息），若没有找到则会执行epoll挂起
- 当执行到遍历View树的 runnable时，ViewRootImpl会移除同步屏障

# Choreographer
- 将和ui相关的任务与vsync同步的一个类。
- 每个任务被抽象成CallbackRecord，同类任务按时间先后顺序组成一条任务链CallbackQueue。四条任务链存放在mCallbackQueues[]数组结构中
- 触摸事件，动画，View树遍历都会被抛到编舞者，并被包装成CallbackRecord并存入链式数组结构，当Choreographer收到一个Vsync就会依次从输入，动画，绘制这些链中取出任务执行
- 当vsync到来时，会向主线程抛异步消息（执行doFrame）并带上消息生成时间，当异步消息被执行时，从任务链上摘取所有以前的任务，并按时间先后顺序逐个执行。
- 当绘制任务耗时，会发生跳帧，即本该绘制的任务被跳过（通过异步消息生成时间和上一帧绘制时间比较，小于上一帧时间则跳过重新订阅vsync），commit任务会在绘制任务执行完后进行帧时间对齐，如果绘制任务时间超过2个帧，则将上一帧时间往后延，目的是跳过旧帧。
- 订阅下一个信号，也被一个被抛到主线程runnable，前序任务执行时间过长，可能会错过订阅时间点

## 系统启动过程
1. linux启动
	1. 按开机键，执行ROM中预设代码
	2. 启动Bootloader，它会检测硬件并启动操作系统（把操作系统映像文件拷贝到RAM中）
	3. 初始化Kernel：初始化内核，最终启动用户空间的init进程
2. Android启动
	4. init进程：init进程是用户空间的第一个进程，pid为1，init进程负责解析init.rc配置文件，两个最重要的守护进程是zygote进程和servicemanager进程，zygote是Android启动的第一个Dalvik虚拟机，servicemanager是Binder通讯的基础。
	5. Zygote进程：这是第一个java进程，他继续fork出system_server进程
	6. system_server进程：在system_server中开启了核心系统服务（AMS,PMS），并将系统服务添加到ServiceManager中，然后系统进入SystemReady状态
	7. 启动Launcher：系统进入SystemReady状态，AMS向Zygote请求启动Launcher，Zygote fork新进程启动Launcher

## PackageManagerService
- 权限处理（增加查询删除）+包处理（安装卸载）+查询四大组件信息
- 启动顺序~是第三个被启动的系统服务：第一启动Installer服务 第二启动ActivityManagerService服务 
- 所在进程：~运行在system_server进程，ApplicationPackageManager是~本地代理
- 构建时机：~在SystemServer启动时被创建（~属于引导服务），PackageManagerService.main(){构建pms实例并将其注册到ServiceManager}
- 扫描apk：~在构造的时候会扫描特定的目录下的apk文件（/system/frameworks，/system/app，/vendor/app，/data/app，/data/app-private），磁盘上的apk会被PackageParse解析成Package对象保存在内存中（解析AndroidManifest.xml），这些信息也会持久化到磁盘形成packages.xml
- 安装apk：~通过Installer调用installd服务进行安装，Installer连接installd的socket，从socket对象获得输入流和输出流，请求installed服务时就向输出流中写字节，installd在init进程中启动的。安装应用就是在data/data目录下创建应用包名目录
- 已安装应用的信息被放在/data/system/packages.xml里，该文件是在解析apk时生成的，记录了 permission apk名字，uid，version。下次开机是从里面读取信息加载到内存中PMS.mSettings
- PMS维护了四大组件的Resolver，将Intent解析成对应的组件类
- PMS在启动后会扫描所有已安装的app，加载他们的manifest文件，生成packages.list 和 packages.xml文件。
- 在安装应用是会判断是否要进行dexopt，即dex优化，通过binder通信调用installd进行优化

## 应用安装流程
- 应用安装过程就是解析apk并拷贝其中内容的过程：
	1. 将apk拷贝到/data/app/包名目录下，将其解压，用PackageParse解析AndroidManifest.xml成Package对象
	2. 将so文件拷贝到/data/app/包名/lib目录
	2. 然后对dex文件进行优化，并保存在data/dalvik-cache目录下,启动ART的可执行文件是oat，系统会将dex转换为oat
	3. 创建data/data/包名 目录，用于存放用户数据       
- APK 的安装是拷贝和注册的过程，不仅仅需要将 APK 的实体放到系统的特定目录（/data/app/），而且需要向 PackageManagerService 注册包名、注册四大组件，该 APK 才能算是成功安装到了 Android系统

## app启动过程
- App启动过程就好比，在Launcher进程中启动另一个进程的Activity
- 冷启动：Launcher 通知 AMS，AMS 通知zygote创建应用进程-创建应用主线程ActivityThread，及消息循环，ActivityThread和AMS不断交互创建Application对象，Activity对象
- 热启动：应用进程还存活，只是不活跃，
- 温启动，进程还存活，但Activity需要重头创建
- 在log中过滤displayed关键词，

## SystemServiceManager
- 它负责创建系统服务和管理系统服务生命周期
- 系统服务是通过反射构造器创建的，然后被添加到SystemServiceManager.mServices中保存，最后调用了onStart()启动服务

## 应用启动过程
1. launcher进程向SystemServer进程请求启动新应用的主activity
2. SystemServer进程创建出对应activityRecord对象
3. 如果界面隶属于现有task，则压栈，否则新建task栈
4. SystemServer要求launcher界面onPause
5. 启动一个starting window
6. 检测新起activity对应的应用进程是否已创建，若没有则让zygote fork一个新进程
7. 应用进程中创建ActivityThread.main(),开启主线程消息循环
8. SystemServer给应用进程主线程发消息要求创建并启动主activity
9. 通过跨进程通信触发activity的各种生命周期回调
10. 到onResume的时候会将顶层视图添加到窗口，并构建viewrootimpl，以此触发一次view树遍历

### 版本1
1. 创建一个name为“zygote”的Socket服务端，监听socket，用于等待AMS的请求Zygote来创建新的进程；
2. 预加载类和资源，包括drawable、color、OpenGL和文本连接符资源等，保存到Resources一个全局静态变量中，下次读取系统资源的时候优先从静态变量中查找；
3. 启动SystemServer进程；
4. 通过runSelectLoop(）方法，开启一个死循环，等待AMS的请求创建新的应用程序进程。

### 版本2
1. ams 通知zygote fork一个app进程
2. 初始化Binder，以进程pid在设备上创建/proc/pid目录，在该目录上可以读取进程binder线程池，内核缓冲区
3. Binder驱动为该进程在内核空间创建一个binder_proc结构体，并放到队列中，以记录多少个进程在使用binder驱动
4. 为应用进程启动binder线程池，后续当binder驱动发现无可用线程时则会通知应用进程创建binder线程
4. 双向通信：应用进程和系统进程双向通信
*ActivityThread--(IActivityManager)-->ActivityManagerService*
应用进程通过IActivityManger接口向系统进程请求服务，这个接口有两个实现，ActivityManagerNative(Stub)和ActivityManagerProxy(Proxy)
*ActivityManagerService--(IApplicationThread)-->ActivityThread*
系统进程通过IApplicationThread将请求结果返回给应用进程，这个接口有两个实现，其中一个ApplicationThreadProxy是客户端的代理运行在系统进程，ApplicationThreadNative是客户端的stub运行在客户端(ActivityThread.ApplicationThread实现了这个接口)，当应用启动时(ActivityThread.attach())，会把ApplicationThread传递给ActivityManagerService。
*通信过程如下* 当前Activity为Activity0，要启动Activity为Activity1
launcher进程--(通过AMP向SystemServer发送请求)--(我要启动Activity1)-->SystemServer
	系统进程创建出Activity1的ActivityRecord
	如果Activity1属于现有TaskRecord则将该TaskRecord置为栈顶，否则新建TaskRecord
launcher进程<--(你先让当前Activity暂停)--系统进程
launcher进程--(我已回调当前Activity.onPause())-->系统进程
	启动一个Starting window 空白页面
	系统检测Activity1对应应用进程是否已经启动（通过ActivityRecord.processName获取ProcessRecord，如果为null表示进程没有启动过），若没有则通知Zygote让其fork一个新的进程
应用进程<--(正式启动ActivityA)--系统进程
	在应用进程通过反射构建ActivityThread对象，并调用它的main方法，在main方法中在开启主线程消息循环
	应用进城想systemServer发起attachApplication请求
	systemServer向应用将进程发起scheduleLaunchActivity请求
	应用进程收到请求后向主线程发送LAUNCH_ACTIVITY消息，然后通过反射构建Activity对象，调用attach方法
	Activity.attach(){将上下文，Application对象，Token和Activity实例绑定，并新建窗口}
	Activity1.onCreate()
	Activity1.onStart()
	Activity1.onRestoreInstanceState()
	Activity1.onPostCreate()
	Activity1.onResume()
	触发一次view树绘制当绘制完成后，将应用窗口替换Starting window
应用进程--(onResume已经执行完毕,我处于空闲状态)-->系统进程
应用进程<--(回调上个Activity后续生命周期函数)--系统进程
	Activity0.onSaveInstanceState()
	Activity0.onStop()
	
该过程中有多次应用进程和system_server进程的通信，每一次通信都会对应一个Binder线程的创建

2. 通过消息机制回调Activity生命周期函数
通过向主线程发送消息的方式，并最终通过Instrumentation来回调Activity生命周期函数

3. 通过token验证Activit身份
token建立了应用进程Activity和系统进程ActivityRecord的关联，在Activity切换时需要通过应用进程将token传递给系统进程，系统进程以此找到对应的ActivityRecord
- token产生
	构造ActivityRecord时会new一个token
- token传递
	IApplicationThread.scheduleLaunchActivity()//将token传递给应用进程
	ActivityThread.performLaunchActivity(){
		Activity.attch()//将token传递给Activity实例
	}
- token存储
	Activity.mToken存储着Token的弱引用

4. 找宿主任务栈
启动Activity时需要找到宿主TaskRecord，如果没有则要新建一个
通过ActivityStack.createTaskRecord()创建TaskRecord

5. 将待启动Activity设置为焦点Activity
通过IActivityManager.setFocusedActivityLocked()


## Activity调度
- ActivityStackSupervisor管理多个ActivityDisplay，一个ActivityDisplay中包含着多个ActivityStack，当前只有一个获得焦点的ActivityStack被显示
- 每个ActivityStack管理者多个TaskRecord，栈顶的TaskRecord表示当前活跃任务
- 每个TaskRecord管理者多个ActivityRecord，栈顶ActivityRecord表示当前活跃界面
- 每个ActivityRecord表示一个Activity
- 每个被显示的Activity，其ActivityRecord必然位于TaskRecord栈顶，其宿主TaskRecord必然位于ActivityStack栈顶

## `ActivityRecord`
- ActivityRecord内部存储了由AndroidManifest.xml文件解析出来的Activity配置信息，同时持有了AMS的引用、Activity当前状态、所属Task等信息

## `TaskRecord`
- TaskRecord记录了任务栈标识符，持有的ActivityRecord列表，ActivityStack的引用
## `ActivityStackSupervisor`
- ActivityStackSupervisor是Android系统中Activity的终极大管家，它本身不直接持有ActivityStack的引用，而是通过持有RootActivityContainer间接持有ActivityDisplay引用，进而间接管理ActivityStack。

## `ActivityDisplay`
- 是对屏幕的抽象
- 维护了各种 stack，包括launcher，最近任务

## `SystemServer`
- systemserver的main方法是它的入口，在main方法中通过 new SystemServer().run(){
	1. 启动boostStrap服务
	2. 启动核心服务
	3. 启动其他服务
}
- 会启动binder线程池，用于和其他进程进行binder通信
- 创建SystemServiceManager，用于启动、创建和管理服务

## `Zygote`
- init进程通过 service 命令创建
- 是第一个art虚拟机
- 通过socket方式与其他进程通信
- 如果每个应用程序在启动之时都需要单独运行和初始化一个虚拟机，会大大降低系统性能，因此Android首先创建一个zygote虚拟机，然后通过它孵化出其他的虚拟机进程，进而共享虚拟机内存和框架层资源（fork比创建更高效），这样大幅度提高应用程序的启动和运行速度
- zygote会提前加载一个资源，共享库，jni函数，systemServer从zygote孵化出来后可以直接使用这些资源
- zygote.main(){
	1. 启动AndroidRuntime ，就是Dalvik虚拟机
}
- 来到ZygoteInit.java类，找到对应的Java main方法，此方法的主要工作为：
1. 预加载类和资源(HAL,图形驱动，文字资源，webview，sharedLibrary)，预加载的好处是fork的时候不需要再次加载，直接拷贝就好，这些资源被保存到全局变量Resources中，它是一个全局静态变量
2. 创建服务端Socket
3. 启动SystemServer进程
4. 开启循环，利用os.poll()管道机制阻塞等待，systemServer通知它创建app进程
- 为啥使用socket通信，因为zygote先于serviceManager，若使用binder，那么需要等待serviceManager创建完成之后再向SystemServer注册Binder服务，这里需要额外的同步操作

# `WindowManagerService`
- ~是系统服务，用于管理窗口（添加窗口，移除窗口，窗口排序，触摸事件，窗口动画）
- ~和客户端双向通信：
	1. ActivityThread--(IWindowSession)-->Session-->WindowManagerService
		WMS本地代理是在ViewRootImpl构造时通过WindowManagerGlobal.getWindowSession()获得
	2. WindowsManagerService--(IWindow)-->ActivityThread
		- IWindow实例也是在ViewRootImpl构造时新建的（new ViewRootImpl.W()）
- 客户端一个window对应一个ViewRootImpl（IWindow，DecorView），在服务端一个window对应一个WindowState，IWindow和WindowState被存在放HashMap中，服务端根据IWindow来去重，同一个window不能add两次
- 服务端有一个WindowToken结构，用于表达窗口分组，一个WindowToken对应同一分组下的若干WindowState, Activity与Dialog对应的是AppWindowToken，PopupWindow对应的是普通的WindowToken
- windowState.attach()会创建 SurfaceSession 和 SurfaceFlinger建立连接
- view只有被添加到窗口后才会被绘制
- WindowManagerService为每一个应用保留一个Session
- AMS在为Activity创建ActivityRecord的时候，会新建IApplicationToken.Stub appToken对象，在startActivity之前会首先向WMS服务登记当前Activity的Token（存在wms的map结构中，键是IApplicationToken，值是AppWindowToken）
- WindowState内维护了三个int类型的layer，这些layer会传递给SurfaceFlinger，已决定在合成是的z轴顺序

## `ActivityManagerService`
- 在SystemServer中被启动，通过SystemServiceManager.startService(ActivityManagerService.Lifecycle.class) {内部通过反射构建了ActivityManagerService.Lifecycle实例，在Lifecycle的构造方法通过new ActivityManagerService()构造 ams}
- 负责四大组件的启动切换调度
- AMS 构造的时候
	1. 创建了两个线程，一个是工作线程，一个是和进程启动相关的线程
	2. 启动了低内存检测LowMemDetector，通过epoll机制获取低内存通知
	3. 构建ActiveServices管理service
	4. 构建ProviderMap管理contentProvider
	5. 初始化ActivityTaskManager
	6. 开启Watchdog，监听线程阻塞
	7. 创建OomAdjuster用于调整进程的优先级等级
	8. 创建BatteryStatsService和ProcessStatsService用于管理电量状态和进程状态
- 获取AMS对象，是通过 ServiceManager.getService() 获取一个 IBinder 对象，然后通过asInterface()获取本地对象或者远程对象的本地代理（Android 10中所有系统服务都通过AIDL接口来获取，在Android10以前获取服务不是通过直接通过AIDL接口的，而是通过ActivityManagerNative来转发，本质还是通过AIDL生成类Stub来获取）
- 它负责管理Activity，它通过ActivityStackSupervisor管理Activity调度，ActivityStackSupervisor实例在它构造函数中被创建
- 它会和应用进程双向通信以完成启动Activity，
- 它维护所有进程信息,包括系统进程，它通过ProcessRecord来维护进程运行时的状态信息，需要将应用进程绑定到ProcessRecord才能开始一个Application的构建

## `ActivityTaskManager`
- android 10 以后，将和activity生命周期相关的代码移到ActivityTaskManager中

## `ServiceManager`
- 所有的服务都保存在ServiceManager内，通过getService()，获取IBinder对象然后调用stub.asInterface()转换成本地代理对象
- 

## `Instrumentation`
- activity的创建和生命周期回调都通过Instrumentation完成