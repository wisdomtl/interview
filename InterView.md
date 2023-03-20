## 传统进程通信
发送方将数据存在自己内存空间的缓存区，内核将缓存区数据拷贝到内核，接收方开辟一块缓冲区用于接收从内核拷贝的数据（存储转发）
## Binder
- Linux将内存空间 = 内核空间（操作系统+驱动）+用户空间（应用程序）为了保证内核安全，它们是隔离的。内核空间可访问所有内存空间，而用户空间不能访问内核空间。
- 用户程序只能通过系统调用陷入内核态，从而访问内核空间。系统调用主要通过 copy_to_user() 和 copy_from_user() 实现，copy_to_user() 用于将数据从内核空间拷贝到用户空间，copy_from_user() 用于将数据从用户空间拷贝到内核空间。
- 安全性好：为发送方添加UID/PID身份信息
- 性能更佳：传输过程只要一次数据拷贝，而Socket、管道等传统IPC手段都至少需要两次数据拷贝
，Android将Binder driver挂载为动态内存（LKM：Loadable Kernel Module），通过它以mmap方式将内核空间与接收方用户空间进行内存映射（用户空间一块虚拟内存地址和内核空间虚拟内存地址指向同一块物理地址），这样就只需发送方将数据拷贝到内核就好了。
- server向binder驱动注册服务，将别名传递给binder驱动，驱动发现是新binder则新建binder结点，再把binder别名和引用传递给 ServiceManager 保存在一张映射表进行注册，client通过别名查映射表，并获取binder引用，然后就能像调普通方法一样调用binder 方法。
- android 系统启动时，SystemServer向binder驱动注册ServiceManager，所有用户进程的0号引用就是 ServiceManager
- Binder 通信是不同进程中线程之间的通信，客户端线程会将线程优先级传递给服务端，服务端会其专门的服务线程。
- 一个服务会生成一个binder_node结点，有一个binder_node_lock，所以多个客户端请求同一个oneway服务时不能发生并行，会进队列排队
- 通过Binder实现的跨进程通信是c/s模式的，客户端通过远程服务的本地代理像服务端发请求，服务端处理请求后将结果返回给客户端
- Binder进程通信流程：
	1. 通过IInterface定义服务后会自动生成stub和proxy
	2. 服务器通过实现stub来实现服务逻辑并将其以IBinder形式返回给客户端
	3. 客户端拿到IBinder后生成一个本地代理对象(通过asInterface())，通过代理对象请求服务，代理会构建两个Parcel对象一个data用来传递参数，另一个reply用来存储服务端处理的结果，并调用BinderProxy的transact()发起RPC(Remote Procedure Call)，同时将当前进程挂起，直到服务端onTract()执行完毕返回（所以如果远程调用很费时，不能在UI线程中请求服务）
	4. 这个请求通过Binder驱动传递到远程的Stub.onTransact()调用了服务真正的实现
	5. 返回结果：将返回值写入reply parcel并返回
- binder采用代理模式让远程调用看上去像本地调用一样
- binder通信的大小限制是1mb-8kb（Binder transaction buffer）,这是mmap内存映射的大小限制（单个进程），其中异步传输oneway传输限制是同步的一半（1mb-8kb）/2，内核允许Binder传输的最大限制是4M（mmap的最大空间4mb）
- 在生成的stub中有一个asInterface():它用于将服务端的IBinder对象转换成服务接口，这种转化是区分进程的，如果客户端和服务端处于同一个进程中，此方法返回的是服务端Stub对象本身，否则新建一个远程服务的本地代理
- 生命周期回调都是 AMS 通过 Binder 通知应用进程调用的
- oneway表示异步调用，发起RPC之前不会构建parcel reply
- in 表示客户端向服务端传送值并不关心值的变化，out表示服务端向客户端返回值

## mmap
Linux io= 标准io + 直接io + mmap
1. 标准io
- 用户空间内容写到内核空间页缓存,缺页则创建页，并标记页为脏页，脏页会在合适的时候写入磁盘(延迟写入+ 两次数据拷贝)
2. 直接IO
- java 没有提供直接io 的 api
3. mmap
mmap是操作系统中一种内存映射的方法，内存映射：就是将用户空间的一块内存区域映射到内核空间。映射关系建立后，用户对这块内存区域的修改可以直接反应到内核空间；反之内核空间对这段区域的修改也能直接反应到用户空间
- 减少系统调用。只需要一次mmap()的系统调用，建立映射关系后就可以像操作内存一样。
- 减少数据拷贝次数。mmap()只需要一次数据拷贝。

## ipc
1. Messenger：不支持RPC（Remote Procedure Call） ，低并发串行通信，并发高可能等待施加长
- 它是除了aidl之外的创建Remote bound service的方法
- 它可以实现客户端和服务器的双向串行通信（来信和回信）
- 服务器和客户端本质上是通过拿到对方的Handler像对方发送消息。但Handler不能跨进程传递，所以在外面包了一层Messenger，它继承与IBinder。服务端和客户端分别将定义在自己进程中的Messenger传递给对方，通过Messenger相互发送消息，其实是Messenger中的Handler发送的，服务端将自己的Messenger通过onServiceConnected()返回，客户端通过将Messenger放在Message.replyTo字段发送给服务器
2. AIDL：支持RPC 一对多并发通信
	- 存在多线程问题：跨进程通信是不同进程之间线程的通信，如果有多个客户端发起请求，则服务端binder线程池就会有多个线程响应。即服务端接口存在多线程并发安全问题。
	- RemoteCallbackList用于管理跨进程回调：其内部有一个map结果保存回调，键是IBinder对象，值是回调，服务端持有的接口必须是RemoteCallbackList<T>类型的
	- 远程服务运行在binder线程池，客户端发出请求后被挂起，如果服务耗时长，客户端可能会产生anr，所以需要新启线程请求服务
3. 文件：通过读写同一个文件实现数据传递。（复杂对象需要序列化），并发读可能发生数据不是最新的情况，并发写可以破坏数据，适用于对同步要求低的场景
4. ContentProvider：一对多进程的数据共享，支持增删改查
5. Bundle：仅限于跨进程的四大组件间传递数据，且只能传递Bundle支持的数据类型

# `Bundle`
- 使用ArrayMap存储结构，省内存，查询速度稍慢，因为是二分查找，适用于小数据量
- 使用Parcelable接口实现序列化，而hashmap使用serializable
## arrayMap和HashMap区别
1. 存储结构：arrayMap用一个数组存储key的哈希值，用一个数组存储key和value（挨着i和i+1），而HashMap用一个Entry结构包裹key，value，所以HashMap更加占用空间。
2. 访问方式：arrayMap通过二分查找key数组，时间复杂度是o（log2n），HashMap通过散列定位方式，时间复杂度是o（n），
3. ArrayMap删除键值对时候会进行数组平移以压缩数组
4. 插入键值对时可能发生数组整体平移以腾出插入位置

# `Parcel`
- 将各种类型的数据或对象的引用进行序列化和反序列化，经过mmap直接写入内核空间
- parcel还可以writeBinder，这个引用导致另一端接收到一个指向这个IBinder的代理IBinder
- 使用复用池（是一个Parcel数组），获取Parcel对象
- parcel 存取数据顺序需要保持一致，因为parcel在一块连续的内存地址，通过首地址+偏移量实现存取
- parcel写入内核的共享内存空间，另一个进程可以直接读取这个内核空间（因为做了mmap，不需要另一次copy_to_user()）
- Active object,我们一般存入到Parcel的是数据的本身，而Active object则是写入一个特殊的标志token，这个token引用了原数据对象。当从Parcel中恢复这个对象时，我们可以不用重新实例化这个对象，而是通过得到一个handle值。通过handle值则可以直接操作原来需要写入的对象

## 画圆角
1. 通过drawRoundRect(),使用的画笔设置一个shader，TileMode.CLAMP
2. 通过drawBitmap(),通过画一个圆角的path 通过clipPath()

## service
- 分类
	1. started service
		- 生命周期和启动他的组件无关，必须显示调用stopservice（）才能停止
		- onCreate() onStartCommand(START_NOT_STICKY START_STICKY START_REDELIVER_INTENT) onDestroy()
	2. bound service
		- 生命周期和启动他的组件绑定，组件都销毁了 他也销毁
		- onCreate() onBind(多个组件绑定) onStartCommand() onUnbind() onRebind() onDestroy()
		- Local Bound Service ：为自己应用程序的组件提供服务
		- Remote Bound Service：为其他应用的组件提供服务
			- 1.aidl
			- 2.Messenger
		- bound service如何和绑定组件生命周期联动：在绑定的时候会将serviceConnection保存在LoadedApk的ArrayMap结构中，当Activity finish的时候，会遍历这个结构逐个解绑
- service默认是后台进程
- Service 和线程区别
	+ thread 是并发运算单元，独立于android 系统，service 是android 系统的一个组件，表示无交互的在背后默默执行的一段逻辑，它会影响当前进程的优先级（后台服务）
	+ service 可以和其他组件生命周期绑定。
- Service和Activity通信
	1. 通过 Intent 传入startService
	2. 发送广播，或者本地广播
	3. bound service，通过方法调用通信
## kotlin性能开销
- 伴生对象开销：会生成静态内部类，对策如下：
	- 对于基本类型和字符串，可以使用const关键字将常量声明为编译时常量。
	- 对于公共字段，可以使用@JvmField注解。
- lazy线程开销：lazy()默认情况下会指定LazyThreadSafetyMode.SYNCHRONIZED，这可能会造成不必要线程安全的开销，应该根据实际情况，指定合适的model来避免不需要的同步锁。
- 基本数据类型数组避免自动装箱：所以当需要声明非空的基本类型数组时，应该使用xxxArray，避免自动装箱。
## sequence
- ~是惰性的：中间操作不会被执行，只有终端操作才会（toList（））
- ~的计算顺序和迭代器不同：~是对一个元素应用全部的操作，然后第二个元素应用全部操作，而迭代器是对列表所有元素应用第一个操作，然后对列表所有元素应用第二个操作
## StringBuffer是线程安全的，StringBuilder是不安全

## 集合
- collection和Map接口
- collection表示一组对象，有些允许重复list（LinkedList，ArrayList），有些不允许重复（HashSet）
- Map表示键值对
- HashSet底层是利用HashMap实现的，键就是add进来的元素，值就是一个Object对象
- ArrayList底层是一个Object数组，他不是线程安全的，Vector通过在方法前添加synchronized实现同步
- LinkedList底层实现是一个双向链表
## string
string是final类型的char数组，表示引用不会改变
## final
表示引用指向不能变，但其指向的变量是可变的
## 泛型
- ~的目的是类型参数化，即用变量表示类型
- 提高代码复用性：编写的代码可复用于多种数据类型
- 提升安全性：在编译期间保证类型安全而不是运行时
- 消除强转：没有泛型的时候，都用Object代替，因为Object可以强转成任何类型，但这样是危险的，麻烦的
- PECS是在使用泛型时为了遵守里氏替换原则必须准守的原则，使用泛型增加代码适用性时保证了类型安全。
	1. PE：Producer extends 实现协变效果：泛型类和类型参数的抽象程度具有相同的变化方向。泛型类只生产泛型，即泛型只会出现在类方法的返回值位置，kotlin中用out表示，java用extend表示
	2. CS:consumer super 实现逆变效果：泛型类只消费泛型，即泛型只出现在类方法的参数位，kotlin中用in表示，java用<? super T>表示
- 类型参数的父子关系是否会延续到外部类上，若延续的叫协变，否则父子关系转向了，这叫逆变，若没有父子关系则叫不变型 ，泛型是不变型
- 类型擦除：为了兼容1.5以前的代码，即编译后的泛型都变成了 Object或者上界，然后在使用的地方进行类型强转，所以泛型只存在于编译前，所以是伪泛型。
- Signature：被擦除的类型信息会记录到~，反射时从~中获取类型
- 当子类覆盖或者实现父类方法时，方法的形参要比父类方法的更为宽松；
- 当子类覆盖或者实现父类方法时，方法的返回值要比父类的更严格。
- 如果在编译的时候就保存了泛型类型到字节码中，那么在运行时我们就可以通过反射获取到，如果在运行时传入实际的泛型类型，这个时候就会被擦除，反射获取不到当前传入的泛型实际类型
- Kotlin reified 可避免类型擦除，它会将方法内联到执行的地方，并且对泛型对象进行instanceof的分类讨论。

## java对象生命周期
1. 创建：为对象分配内存，构造对象
2. 应用：至少一个强引用指向它
3. 不可见：不再持有强引用（程序执行超出了对象的作用域）
4. 不可达：没有强引用指向
5. 收集：准备gc，会执行 finalize()
6. 终结：等待垃圾回收
6. Deallocated：回收完成

## [HashMap]
- 存储结构是开散列表：地址向量+同义词子表=数组+单链表。
- 解决哈希冲突的办法是拉链法：将相同散列地址的键值存放在同义词子表中。
- capacity为啥要为2的幂次，是为了用位与运算代替取模运算，提高性能。
- 为啥loadFactor 是0.75，因为是一个
- 构造~时，并没有初始化地址向量，而是要等到put操作是才构造
- 遍历HashMap的顺序是从地址向量的第一个开始，先从前到后遍历同义词子表，然后下一个同义词子表
- ~通过hash算法先定位到地址向量中对应的位置，然后遍历同义词子表
- ~不是线程安全的，当~扩容的时候要进行迁移，多线程并发put会导致迁移出环。建议使用Hashtable或者ConcurrentHashMap。Hashtable将put和get方法都加上了synchronized，性能较差 

## 引用
### 强引用
- 通过=显式的将对象A赋值给变量a，则A就存在一个强引用a
- 强引用需要显式的置null 以告诉gc该对象可以被回收
- 在一个方法的内部有一个强引用，这个引用保存在栈中，而真正的引用内容（Object）保存在堆中。当这个方法运行完成后就会退出方法栈，则引用内容的引用不存在，这个Object会被回收。但是如果这个o是全局的变量时，就需要在不用这个对象时赋值为null，因为强引用不会被垃圾回收
- 清空list时需要遍历所有元素将其置null
### 软引用
- 如果一个对象只具有软引用，则内存空间足够，垃圾回收器就不会回收它；如果内存空间不足了，就会回收这些对象的内存。只要垃圾回收器没有回收它，该对象就可以被程序使用。软引用可用来实现内存敏感的高速缓存。
### 弱引用
- 弱引用与软引用的区别在于：只具有弱引用的对象拥有更短暂的生命周期。在垃圾回收器线程扫描它所管辖的内存区域的过程中，一旦发现了只具有弱引用的对象，不管当前内存空间足够与否，都会回收它的内存。不过，由于垃圾回收器是一个优先级很低的线程，因此不一定会很快发现那些只具有弱引用的对象。
### 虚引用
- 虚引用主要用来跟踪对象被垃圾回收器回收的活动，当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之 关联的引用队列中。用于在对象被回收时做一些事情
- 

### 软引用、弱引用、虚引用的构造方法均可以传入一个ReferenceQueue与之关联。在引用所指的对象被回收后，引用（reference)本身将会被加入到ReferenceQueue之中，此时引用所引用的对象reference.get()已被回收 (reference此时不为null，reference.get()此时为null)。在一个非强引用所引用的对象回收时，如果引用reference没有被加入到被关联的ReferenceQueue中，则表示还有引用所引用的对象还没有被回收。如果判断一个对象的非强引用本该出现在ReferenceQueue中，实际上却没有出现，则表示该对象发送内存泄漏。


## 接口和抽象类
- 类可以实现很多个接口，但是只能继承一个抽象类
- 类如果要实现一个接口，它必须要实现接口声明的所有方法。但是，类可以不实现抽象类声明的所有方法，当然，在这种情况下，类也必须得声明成是抽象的。
## 字符串常量池
- JVM为了减少字符串对象的重复创建，其维护了一个特殊的内存，这段内存被成为字符串常量池
- 字符串常量池实现的前提条件是java中的String对象是不可变的，否则多个引用指向同一个变量的时候并改变了String对象，就会发生错乱
- 字符串常量池是用时间换空间，cpu需要在常量池中寻找是否有相同字符串
- 字符串构造方式
	1. 字面量形式：String str = "droid"
		 - 使用这种形式创建字符串时，JVM首先会对这个字面量进行检查，如果字符串常量池中存在相同内容的字符串对象的引用，则将这个引用返回，否则新的字符串对象被创建，然后将这个引用放入字符串常量池，并返回该引用
	2. 新建对象形式：String str = new String("droid"); 
		- 使用这种形式创建字符串时，不管字符串常量池中是否有相同内容，新的字符串总是会被创建
		- 对于上面使用new创建的字符串对象，如果想将这个对象的引用加入到字符串常量池，可以使用intern方法。调用intern后，首先检查字符串常量池中是否有该对象的引用，如果存在，则将这个引用返回给变量，否则将引用加入并返回给变量。`String str4 = str3.intern();`
## view生命周期
构造View --> onFinishInflate --> onAttachedToWindow --> onMeasure --> onSizeChanged --> onLayout --> onDraw --> onDetachedFromWindow
## 设计原则
- 单一职责原则：关于内聚的原则。高内聚、低耦合的指导方针，类或者方法单纯，只做一件事情
- 接口隔离原则：关于内聚的原则。要求设计小而单纯的接口（将过大的接口拆分），或者说只暴露必要的接口，这样可以避免强迫开发者实现不想实现的接口
- 最少知识法则
	- 关于耦合的原则。要求类不要和其他类发生太多关联，达到解耦的效果
	- 从依赖者的角度来说，只依赖应该依赖的对象。
	+ 从被依赖者的角度说，只暴露应该暴露的方法。
	- 对象方法的访问范围应该受到约束：
		1. 对象本身的方法
		2. 对象成员变量的方法
		3. 被当做参数传入对象的方法
		4. 在方法体内被创建对象的方法
		5. 不能调用从另一个调用返回的对象的方法
- 开闭原则：关于扩展的原则。对扩展开放对修改关闭，做合理的抽象就能达到增加新功能的时候不修改老代码（能用父类的地方都用父类，在运行时才确定用什么样的子类来替换父类），开闭原则是目标，里氏代换原则是基础，依赖倒转原则是手段
- 里氏替换原则
	- 为了避免继承的副作用，若继承是为了复用，则子类不该改变父类行为，这样子类就可以无副作用地替换父类实例，若继承是为了多态，则因为将父类的实现抽象化，
- 依赖倒置原则：即是面向接口编程，面向抽象编程，高层模块不该依赖底层模块，而是依赖抽象，

# 设计模式
## 单例模式
1. 静态内部类
	- 虚拟机保证一个类的初始化操作是线程安全的，而且只有使用到的时候才会去初始化，缺点是没版本传递参数
```
public class Lock2Singleton {
  	private volatile static Lock2Singleton INSTANCE;    // 加 volatile
  
  	private Lock2Singleton() {}
  
  	public static Lock2Singleton getSingleton() {
      	if (INSTANCE == null) {                         // 双重校验：第一次校验:为了性能
          	synchronized(Lock2Singleton.class) {        // 加 synchronized
              	if (INSTANCE == null) {                 // 双重校验：第二次校验，为了保证实例唯一。
                  	INSTANCE = new Lock2Singleton();
                }
            }
        }
      	return INSTANCE;
    }
}

```
## 工厂模式
- 目的：解耦。~将对象的使用和对象的构建分割开，使得和对象使用相关的代码不依赖于构建对象的细节
- 增加了一层“抽象”将“变化”封装起来，然后对“抽象”编程，并利用”多态“应对“变化”，对工厂模式来说，“变化”就是创建对象。
- 实现方式
	1. 简单工厂模式
		- 将创建具体对象的代码移到工厂类中
		- 实现了隐藏细节和封装变化，对变化没有弹性，当需要新增对象时需要修改工厂类
	2. 工厂方法模式
		- 定义一个创建对象的抽象方法，让子类决定实例化哪一个具体对象。
		- 角色：抽象对象，具体对象，抽象使用者，具体使用者
		- 特点
			- 只适用于构建一个对象 构建多个对象请使用抽象工厂模式
			- 使用继承实现多态
	3. 抽象工厂模式
		- 定义一个创建对象的接口，把多个对象的创建细节集中在一起
		- 角色：多个抽象对象，多个具体对象，抽象工厂接口，具体工厂，使用者
		- 特点：使用组合实现多态
## 建造者模式
- 它是一种构造复杂对象的方式，复杂对象有很多可选参数，如果将所有可选参数都作为构造函数的参数，则构造函数太长，~实现了分批设置可选参数。Builder模式增加了构造过程代码的可读性
- Dialog用到了这个模式
## 观察者模式
是一种一对多的通知方式（被观察者通知观察者），被观察者持有观察者的引用
ListView的BaseAdapter中有DataSetObservable，在设置适配器的时候会创建观察者并注册，调用notifydataSetChange时会通知观察者，观察者会requestLayout
## 策略模式
- 策略模式目的：将使用算法的客户和算法的实现解耦
- 手段：增加了一层“抽象”将“变化”封装起来，然后对“抽象”编程，并利用”多态“应对“变化”，对策略模式来说，“变化”就是一组算法。
- 实现方式：将算法抽象成接口，用组合的方式持有接口，通过依赖注入动态的修改算法
- setXXListener都是这种模式
## 装饰者模式
- 装饰者模式意图：为现有对象扩展功能
- 手段：具体对象持有超类型对象
- ~是继承的一种替代方案，避免了泛滥子类。
- ~增加了一层抽象，这层抽象在原有功能的基础上扩展新功能，为了复用原有功能，它持有原有对象。这层抽象本身是一个原有类型
- ~实现了开闭原则
## 外观模式
- 外观模式意图：隐藏细节，降低复杂度
- 手段：增加了一层抽象，这层抽象屏蔽了不需要关心的子系统调用细节
- 降低了子系统与客户端之间的耦合度，使得子系统的变化不会影响调用它的客户类。
- 对客户屏蔽了子系统组件，减少了客户处理的对象数目，并使得子系统使用起来更加容易。
- 实现方式：外观模式会通过组合的方式持有多个子系统的类，~提供更简单易用的接口（和适配器类似，不过这里是新建接口，而适配器是已有接口）
- 通过~，可以让类更加符合最少知识原则
- ContextImpl是外观模式
## 适配器模式
- 适配器模式意图： 将现有对象包装成另一个对象
- 手段：增加了一层抽象，这层抽象完成了对象的转换。（具体对象持有另一个而具体对象）
- 是一种将两个不兼容接口（源接口和目标接口）适配使他们能一起工作的方式，通过增加一个适配层来实现，最终通过使用适配层而不是直接使用源接口来达到目的。
## 代理模式
- 代理模式意图：限制对象的访问。或者隐藏访问的细节
- 手段：增加了一层抽象，这层抽象拦截了对对象的直接访问
- 实现方式：代理类通过组合持有委托对象（装饰者是直接传入对象，而代理通常是偷偷构建对象）
- 分类 ：代理模式分为静态代理和动态代理
静态代理：在编译时已经生成代理类，代理类和委托类一一对应
动态代理：编译时还未生成代理类，只是定义了一种抽象行为（接口），只有当运行后才生成代理类，使用Proxy.newProxyInstance(),并传入invocationHandler
- Binder是代理模式
- ContextWrapper
## 模板方法模式
- 模版方法模式目的：复用代码
- 手段：新增了一层抽象（父类的抽象方法），这层抽象将算法的某些步骤泛化，让子类有不同的实现
- 实现方式：在方法（通常是父类方法）中定义算法的骨架，将其中的一些步骤延迟到子类实现，这样可以在不改变算法结构的情况下，重新定义某些步骤。这些步骤可以是抽象的（表示子类必须实现），也可以不是抽象的（表示子类可选实现，这种方式叫钩子）
- android触摸事件中的拦截事件是钩子
- android绘制中的onDraw()是钩子
## 命令模式
- 命令模式意图：将执行请求和请求细节解耦
- 手段：增加了一层“抽象”将“变化”封装起来，然后对“抽象”编程，并利用”多态“应对“变化”，对命令模式来说，“变化”就是请求细节。新增了一层抽象（命令）
- 这层抽象将请求细节封装起来，执行者和这层抽象打交道，就不需要了解执行的细节。因为请求都被统一成了一种样子，所以可以统一管理请求，实现撤销请求，请求队列
- 实现方式：将请求定义成命令接口，执行者持有命令接口
- java中的Runnable就是命令模式的一种实现
## 桥接模式
- 意图：提高系统扩展性
- 手段：抽象持有另一个抽象
- 是适配器模式的泛化模式
## 访问者模式
- 意图：动态地为一类对象提供消费它们的方法。
- 重载是静态绑定（方法名相同，参数不同），即在编译时已经绑定，方法的参数无法实现运行时多态
- 重写是动态绑定（继承），方法的调用者可实现运行时多态
- 双分派：实现a.fun(b)在a和b上都实现运行时多态，实现方法调用者和参数的运行时多态。
- 编译时注解使用了访问者模式，一类对象是Element，表示构成代码的元素（类，接口，字段，方法），他有一个accept方法传入一个Visitor对象
## 异常
- Exception和Error都继承于Throwable
- Exception是程序错误
- Exception又分为checked Exception（编译时异常）和unchecked Exception（运行时）。checked Exception在代码里必须显式的进行捕获，这是编译器检查的一部分。unchecked Exception也就是运行时异常，类似空指针异常、数组越界等，通常是可以避免的逻辑错误
- Error是比程序更加低层的错误， 包括虚拟机错误OutOfMemoryError，StackOverFlowError



## 内存模型
dalvik虚拟机内存空间被划分成多个区域 = 虚拟机栈+ 程序计数器+ 方法区+ 堆+ 本地方法栈
- 方法区：方法区主要是存储已经被 JVM 加载的类信息（版本、字段、方法、接口）、常量、静态变量、即时编译器编译后的代码和数据。该区域被各个线程共享的内存区域。
- 堆区：又称动态内存分配，存放所有用通过new创建的类对象（包括该对象其中的所有成员变量），也就是对象的实例。这部分内存在不使用时将会由 Java 垃圾回收器来负责回收
	- 堆内存没有可用的空间存储生成的对象，JVM会抛出java.lang.OutOfMemoryError
	- 堆内存分为新生代和老年代和永生代，新生代又分为Eden、From Survivor、To Survivor三个区域
		- 永生代用于存放class信息
- 虚拟机栈 ：虚拟机栈是线程私有的数据结构，它用来描述 Java 方法执行的内存模型，一个java方法的开始和结束对应一个栈帧的入出
	- 栈帧（Stack Frame）
		- 一个线程包含多个栈帧，而每个栈帧内部包含局部变量表、操作数栈、动态连接、返回地址等
		- 局部变量表是变量值的存储空间，我们调用方法时传递的参数，以及在方法内部创建的局部变量都保存在局部变量表中。在 Java 编译成 class 文件的时候，就会在方法的 Code 属性表中的 max_locals 数据项中，确定该方法需要分配的最大局部变量表的容量。
		- 在方法退出后都需要返回到方法被调用的位置，程序才能继续执行。而虚拟机栈中的“返回地址”就是用来帮助当前方法恢复它的上层方法执行状态。
	- 如果栈内存没有可用的空间存储方法调用和局部变量，JVM会抛出java.lang.StackOverFlowError
- 本地方法栈和虚拟机栈类似，只不过用于执行native方法
- 程序计数器：每个线程都需要一个程序计数器，用于记录正在执行指令的地址

## gc
- 垃圾定义：有两种定义垃圾的方法
	1. 引用计数：给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；计数器为0的对象就是垃圾
	2. 可到达性:从GC Roots作为起点，向下搜索它们引用的对象，可以生成一棵引用树，树的节点视为可达对象，反之视为不可达。不可到达的对象是垃圾，被定义为垃圾的对象不代表马上会被回收，还会去检查是否要执行finalize方法
- GC种类：Minor GC、Full GC ( 或称为 Major GC )
	- 垃圾回收回收不可到达的对象，即没有引用的对象，可到达的对象一定被根引用
	- Minor GC 是发生在新生代中的垃圾收集动作，所采用的是copy and sweep(经过6次gc还存活的对象会被放到老年代)
	- Full GC 是发生在老年代的垃圾收集动作，所采用的是mark and sweep
	- 分代回收(generational collection)：每个对象记录有它的世代(generation)信息。所谓的世代，是指该对象所经历的垃圾回收的次数。世代越久远的对象，在内存中存活的时间越久
- GC回收算法	 
	- copy and sweep：内存被分为两个区域。对象总存活于两个区域中的一个。当垃圾回收启动时，Java程序暂停运行。JVM从根出发，找到可到达对象，将可到达对象复制到空白区域中并紧密排列，修改由于对象移动所造成的引用地址的变化。最后，直接清空对象原先存活的整个区域，使其成为新的空白区域。适用于存活对象少，垃圾对象多的场景
	- mark and sweep：每个对象将有标记信息，用于表示该对象是否可到达。当垃圾回收启动时，Java程序暂停运行。JVM从根出发，找到所有的可到达对象，并标记(mark)。随后，JVM需要扫描整个堆，找到剩余的对象，并清空这些对象所占据的内存堆，缺点是容易产生内存碎片。适用于存活对象多，垃圾对象少的场景
	- 分代回收算法：老年代每次gc只有少量对象被回收，而新生代有大量对象被回收，对于新生代采用copy and sweep，对老年代采用mark and sweep。

### gcRoot
1、虚拟机栈（javaStack）（栈帧中的局部变量区，也叫做局部变量表）中引用的对象。

2、方法区中的类静态属性引用的对象。

3、方法区中常量引用的对象。

4、本地方法栈中JNI(Native方法)引用的对象。

eg
// 分配了一个又一个对象
放到Eden区
// 不好，Eden区满了，只能GC(新生代GC：Minor GC)了
把Eden区的存活对象copy到Survivor A区，然后清空Eden区（本来Survivor B区也需要清空的，不过本来就是空的）
// 又分配了一个又一个对象
放到Eden区
// 不好，Eden区又满了，只能GC(新生代GC：Minor GC)了
把Eden区和Survivor A区的存活对象copy到Survivor B区，然后清空Eden区和Survivor A区
// 又分配了一个又一个对象
放到Eden区
// 不好，Eden区又满了，只能GC(新生代GC：Minor GC)了
把Eden区和Survivor B区的存活对象copy到Survivor A区，然后清空Eden区和Survivor B区
## dalvik gc
- 有四种类型
Dalvik的GC类型共有四种：
GC_CONCURRENT: 表示是在已分配内存达到一定量之后触发的GC。
GC_FOR_MALLOC: 表示是在堆上分配对象时内存不足触发的GC。
GC_BEFORE_OOM: 表示是在准备抛OOM异常之前进行的最后努力而触发的GC。
GC_EXPLICIT: 表示是应用程序调用System.gc、VMRuntime.gc接口或者收到SIGUSR1信号时触发的GC。
其中前三种是在分配对象时触发的
- 分为串行gc和并行gc
- 串行gc开始时会将非gc线程都暂停，直到gc完成
## art虚拟机
- Non Moving Space：存放只读属性，永久数据
- Zygote Space：zygote和应用共享
- Allocation Space：进程独占
- Image Space：连续地址空间，不进行垃圾回收，存放系统预加载类，每次开机启动只需把系统类映射到Image Space，zygote和应用进程共享
- Large Obj Space：离散地址空间，存放大于12K的大对象，不属于堆
2. 三种力度不同的回收策略Mark Sweep（3）、Partial Mark Sweep（2）和Sticky Mark Sweep（1），他们分为并行和串行
- 为大对象单独分配内存区域，避免造成堆的gc
- 后台默默地做内存整理moving collector，减少碎片，dvm使用mark and sweep 导致内存碎片。
- 并行gc被分割成多个子任务，由线程池执行，充分利用多核性能，让gc过程更高效
## JIT ART
- dalvik每次应用启动都会发生jit，将dex字节码转换成机器码
- ART和Dalvik都算是一种Android运行时环境，或者叫做虚拟机，用来解释dex类型文件。但是ART是安装时解释（AOT），会占用额外的存储空间，安装很慢，Dalvik是运行时解释（JIT），即每次启动都会编译，启动慢，耗电
- JIT对于使用频率高的代码编译成机器码
## dalvik虚拟机，jvm
- 在Android 5.0以下，使用的是Dalvik虚拟机，5.0及以上，则使用的是ART虚拟机
- dex适用于低内存及速度有限的设备，dex文件比class文件体积小（消除了冗余信息，将多个class文件相同的部分统一存一份）
- dalvik虚拟机内存空间
	- Linear Alloc：存放只读属性，永久数据
	- Zygote Space：zygote和应用共享
	- Allocation Space：进程独占
- jvm基于内存栈执行，dvm基于寄存器执行，速度更快
- Dalvik将堆分成了Active堆和Zygote堆
- jvm中每一个类对应一个.class文件 而dvm将所有.class抱在一起并压缩

# 编译打包流程
1. 打包资源，res下文件转换成二进制，asset目录
2. 编译java文件为class文件
3. 将class文件转换为dex文件
4. 将资源和dex打包到apk
5. 签名

## 广播
- 是一种用观察者模式实现的异步通信
- 有静态注册和动态注册两种方式，静态注册生命周期比动态注册长（在应用安装后就处于监听状态）
	- 静态注册广播接收器时只要定义了intent-filter 则android:exported属性为true 表示该广播接收器是跨进程的
	- 静态注册广播容易被攻击:其他App可能会针对性的发出与当前App intent-filter相匹配的广播，由此导致当前App不断接收到广播并处理；
	- 静态注册广播容易被劫持：其他App可以注册与当前App一致的intent-filter用于接收广播，获取广播具体信息。
	- 通过manifest注册的广播是静态广播
	- 静态广播是常驻广播，常驻广播在应用退出后依然可以收到
	- 动态注册的不是常驻广播，它的生命周期痛注册组件一致
	- 动态注册要等到启动他的组件启动时时才注册
	- 动态广播优先级比静态高
- 广播接收者BroadcastReceiver通过Binder机制向AMS(Activity Manager Service)进行注册，广播发送者通过binder机制向AMS发送广播，广播的Intent和Receiver会被包装在BroadcastRecord中，多个BroadcastRecord组成队列
- onReceive()回调执行时间超过10s 会发生anr，如果有耗时操作需要使用IntentService处理，不建议新建线程
- 分为有序广播和无序广播
	- 无序广播：所有广播接收器接受广播的先后顺序不确定，广播接收者无法阻止广播发送给其他接收者。
	- 有序广播：广播接收器收到广播的顺序按预先定义的优先级从高到低排列 接收完了如果没有丢弃，就下传给下一个次高优先级别的广播接收器进行处理，依次类推，直到最后
		- 自定义Intent并通过sendOrderBroadcast()发送
		- 可以通过在intent-filter中设置android:priority属性来设置receiver的优先级，优先级相同的receiver其执行顺序不确定，如果BroadcastReceiver是代码中注册的话，且其intent-filter拥有相同android:priority属性的话，先注册的将先收到广播
		- 使用setResult系列函数来结果传给下一个BroadcastReceiver
		- getResult系列函数来取得上个BroadcastReceiver返回的结果
		- abort系列函数来让系统丢弃该广播，使用该广播不再传送到别的BroadcastReceiver
- 还可以分为本地广播和跨进程广播
	- 本地广播仅限于在应用内发送广播，向LocalBroadCastManager注册的接收器都存放在本地内存中，跨进程广播都注册到system_server进程 
## IntentService
- 他是Service和消息机制的结合，它适用于后台串行处理一连串任务，任务执行完毕后会自销毁。
- 它启动时会创建HandlerThread和Handler，并将HandlerThread的Looper和Handler绑定，每次调用startService()时，通过handler发送消息到新线程执行
- 但是它没有将处理结果返回到主线程，需要自己实现（可以通过本地广播）

## 触摸事件
- 触摸事件的传递是从根视图自顶向下“递”的过程，触摸事件的消费是自下而上“归”的过程。
- 触摸事件由ViewRootImpl通过ViewRootHandler接收到，然后存取一个链式队列，再逐个分发给`Activity`接收到触摸事件后，会传递给`PhoneWindow`，再传递给`DecorView`，由`DecorView`调用`ViewGroup.dispatchTouchEvent()`自顶向下分发`ACTION_DOWN`触摸事件。
- `ACTION_DOWN`事件通过`ViewGroup.dispatchTouchEvent()`从`DecorView`经过若干个`ViewGroup`层层传递下去，最终到达`View`。`View.dispatchTouchEvent()`被调用。
- `View.dispatchTouchEvent()`是传递事件的终点，消费事件的起点。它会调用`onTouchEvent()`或`OnTouchListener.onTouch()`来消费事件。
- 每个层次都可以通过在`onTouchEvent()`或`OnTouchListener.onTouch()`返回`true`，来告诉自己的父控件触摸事件被消费。只有当下层控件不消费触摸事件时，其父控件才有机会自己消费。
- `ACTION_MOVE`和`ACTION_UP`会沿着刚才`ACTION_DOWN`的传递路径，传递给消费了`ACTION_DOWN`的控件，如果该控件没有声明消费这些后序事件，则它们也像`ACTION_DOWN`一样会向上回溯让其父控件消费。
- 父控件可以通过在`onInterceptTouchEvent()`返回`true`来拦截事件向其孩子传递。如果在孩子已经消费了`ACTION_DOWN`事情后才进行拦截，父控件会发送`ACTION_CANCEL`给孩子。
- 事件分发流程：system server 进程中有一个InputReader和InputDispatcher，后者会和应用进程socket 通信，应用进程的native 层是 socket 客户端，客户端收到消息后会在应用进程的 ViewRootImpl中构建InputEventReceiver ，然后就是将事件分发到Activity，然后是window，再是 DecorView，然后在 View 树上传递

## 滑动冲突
父子都可以消费滑动事件时会发生滑动冲突
	1. 父控件主动：父控件只拦截自己滑动方向上的事件，其余事件不拦截继续传递给子控件
	```
	@Override
	public boolean onInterceptTouchEvent(MotionEvent ev) {
    final float x = ev.getX();
    final float y = ev.getY();
 
    final int action = ev.getAction();
    switch (action) {
        case MotionEvent.ACTION_DOWN:
            mDownPosX = x;
            mDownPosY = y;
 
            break;
        case MotionEvent.ACTION_MOVE:
            final float deltaX = Math.abs(x - mDownPosX);
            final float deltaY = Math.abs(y - mDownPosY);
            // 这里是够拦截的判断依据是左右滑动，读者可根据自己的逻辑进行是否拦截
            if (deltaX > deltaY) {
                return false;
            }
    }
 
    return super.onInterceptTouchEvent(ev);
	}
	```
2. 子控件主动：子控件要求父控件不要拦截down事件和自己想消费的move事件，其余事件还是让父控件拦截

```
@Override
public boolean dispatchTouchEvent(MotionEvent ev) {
    int x = (int) ev.getRawX();
    int y = (int) ev.getRawY();
    int dealtX = 0;
    int dealtY = 0;
 
    switch (ev.getAction()) {
        case MotionEvent.ACTION_DOWN:
            dealtX = 0;
            dealtY = 0;
            // 保证子View能够接收到Action_move事件
            getParent().requestDisallowInterceptTouchEvent(true);
            break;
        case MotionEvent.ACTION_MOVE:
            dealtX += Math.abs(x - lastX);
            dealtY += Math.abs(y - lastY);
            Log.i(TAG, "dealtX:=" + dealtX);
            Log.i(TAG, "dealtY:=" + dealtY);
            // 这里是够拦截的判断依据是左右滑动，读者可根据自己的逻辑进行是否拦截
            if (dealtX >= dealtY) {
                getParent().requestDisallowInterceptTouchEvent(true);
            } else {
                getParent().requestDisallowInterceptTouchEvent(false);
            }
            lastX = x;
            lastY = y;
            break;
        case MotionEvent.ACTION_CANCEL:
            break;
        case MotionEvent.ACTION_UP:
            break;
 
    }
    return super.dispatchTouchEvent(ev);
}

```
	
## `Handler`
- 用于串行通信的工具类。
- 消息池：链式结构，静态，所有消息共用，取消息时从头部取，消息分发完毕后头部插入
- 消息按时间先后顺序插入到链表结构的消息队列中，最旧的消息在队头，最新的消息在队尾
- Looper通过无限循环从消息队列中取消息并分发给其对应的Handler，并回收消息
- Android消息机制共有三种消息处理方式，它们是互斥的，优先级从高到低分别是1. Runnable.run() 2. Handler.callback 3. 重载Handler.handleMessage()
- 若消息队列中消息分发完毕,则调用natviePollOnce()阻塞当前线程并释放cpu资源,当有新消息插入时或者超时时间到时线程被唤醒
- idleHandler 是在消息队列空闲时会被执行的逻辑,每拿取一次消息有且仅有一次机会执行.通过queueIdle()返回true表示每次取消息时都会执行,否则执行一次就会被移出
- 同步消息屏障是一种特殊的同步消息,他的target为null, 在 MessageQueue.next()中遇到该消息,则会遍历消息队列优先执行所有异步消息,若遍历到队列尾部还是没有异步消息,则阻塞会调用epoll,直到异步消息到来或者同步屏障被移出
- 使用epoll实现消息队列的阻塞和唤醒，Message.next()是个无限循环，若当前无消息可处理会阻塞在nativePollOnce（），若有延迟消息，则设置超时，没有消息时主线程休眠不会占用cpu
- epoll是一个IO事件通知机制 监听多个文件描述符上的事件 epoll 通过使用红黑树搜索被监控的文件描述符（内核维护的文件打开表的索引值）
- epoll最终是在epoll_wait上阻塞
- nativeWake() 唤醒是通过往管道中写入数据，epoll监听写入事件，epoll_wait()就返回了


## ContentProvider
- 它将访问应用程序数据的行为抽象成接口，屏蔽了具体的访问细节，若更换数据库也不会影响上层访问数据的代码
- ContentProvider 主要以表格的形式组织数据，同时也支持文件数据
- 统一资源标识符即 URI，用来唯一标识 ContentProvider 其中的数据，外界进程通过 URI 找到对应的 ContentProvider 其中的数据，在进行数据操作。
- URI格式
	`content://authority/path/id`
	authority:授权信息，用以区分不同的 ContentProvider
	path:表名，用以区分 ContentProvider 中不同的数据表
	id: ID号，用以区别表中的不同数据
	示例:content://com.example.omooo.demoproject/User/1
	上述 URI 指向的资源是：名为 com.example.omooo.demoproject 的 ContentProvider 中表名为 User 中 id 为 1 的数据
- MIME 数据类型:它是用来指定某个扩展名的文件用某种应用程序来打开。可以通过ContentProvider.getType(uri) 来获得。每种 MIME 类型由两部分组成：类型 + 子类型。示例：text/html、application/pdf
- 可以自定义全局的读写权限还可以定义path权限
- Activity、Service、BroadcastReceiver 这三大组件都只有在它们被调用到时，才会进行实例化，并执行它们的生命周期；ContentProvider 即使在没有被调用到，也会在启动阶段被自动实例化并执行相关的生命周期。在进程的初始化阶段调用完 Application 的 attachBaseContext 方法后，会再去执行 installContentProviders 方法，对当前进程的所有 ContentProvider 进行 install
- ContentProvider.onCreate()在Application.onCreate()之前调用
## ContentResolver
统一管理不同的 ContentProvider 间的操作。即通过 URI 即可操作不同的 ContentProvider 中的数据
## ContentObserver
- 观察指定uri变化并作出响应
- 通过将自己注册给Content service
- ContentResolver.notifyChange()将数据变化通知给content service
## Serializable 和 Parcelable 的区别
- Parcelable是将一个普通对象的数据保存在parcel中的接口,Parcelable 使用parcel作为数据读写的载体，调用Parcel.marshall()进行序列化
- Parcelable将数据序列化后存入共享内存（内核空间），其他进程从这块共享内存中读取字节流并反序列化
- Serializable使用反射，并且会产生一些除数据本身以外的额外信息，比如协议，类名长度，字段长度，速度慢，会产生很多中间变量
- Intent 实现了Parcelable，如果put serializable的值会先序列化为字节数组，然后再写入，二次序列化
- 序列化是将结构化对象转换为字节流的过程
- 序列化将对象转换成更加通用的形式，方便传输，存储
## 存储方式
1. SharedPreference
	- 以xml文件形式存储在磁盘
	- 读写速度慢，需要解析xml文件
	- 文明存储，安全性差
	- 是线程安全的，但不是进程安全的
	- 调用写操作后会先写入内存中的map结构，调用commit或者apply才会执行写文件操作
	- anr:
		1. Activity或者service onstop的时候，sp会等待写任务结束，如果任务迟迟没有结束则会造成anr
		2. getSharedPreference会开启线程读文件，调用getString会调用wait()等待读文件结束，如果文件大，则会造成主线程阻塞。
	- sp一共有三个锁：写文件锁，读内存map的锁，写editor中map的锁
2. SQLite
3. 文件
4. DataStore
	- 基于Flow，提供了挂起方法而不是阻塞方法。
	- 基于事物更新数据
	- 支持protocol buffer
	- 不支持部分刷新
## recyclerview
1. **`Recycler`有4个层次用于缓存`ViewHolder`对象，优先级从高到底依次为`ArrayList<ViewHolder> mAttachedScrap`、`ArrayList<ViewHolder> mCachedViews`、`ViewCacheExtension mViewCacheExtension`、`RecycledViewPool mRecyclerPool`。如果四层缓存都未命中，则重新创建并绑定`ViewHolder`对象**
2. **缓存性能：**
    | 缓存 | 重新创建`ViewHolder` | 重新绑定数据 |
    | ------ | ------ | ------ |
    | mAttachedScrap | false | false |
    | mCachedViews | false | false |
    | mRecyclerPool | false | true |

3. **缓存容量：**
    - `mAttachedScrap`：没有大小限制，但最多包含屏幕可见表项。
    - `mCachedViews`：默认大小限制为2，放不下时，按照先进先出原则将最先进入的`ViewHolder`存入回收池以腾出空间。
    - `mRecyclerPool`：对`ViewHolder`按`viewType`分类存储（通过`SparseArray`），同类`ViewHolder`存储在默认大小为5的`ArrayList`中。
4. **缓存用途：**
    - `mAttachedScrap`：用于布局过程中屏幕可见表项的回收和复用。
    - `mCachedViews`：用于移出屏幕表项的回收和复用，且只能用于指定位置的表项，有点像“回收池预备队列”，即总是先回收到`mCachedViews`，当它放不下的时候，按照先进先出原则将最先进入的`ViewHolder`存入回收池。
    - `mRecyclerPool`：用于移出屏幕表项的回收和复用，且只能用于指定`viewType`的表项
5. **缓存结构：**
    - `mAttachedScrap`：`ArrayList<ViewHolder>`
    - `mCachedViews`：`ArrayList<ViewHolder>`
    - `mRecyclerPool`：对`ViewHolder`按`viewType`分类存储在`SparseArray<ScrapData>`中，同类`ViewHolder`存储在`ScrapData`中的`ArrayList`中

## ItemDecoration
用于绘制ItemView以外的内容，有两个回调onDraw和onDrawOver，区别是绘制的顺序有先后，onDraw()会在RecyclerView.onDraw()中调用，表示绘制RecyclerView自身的内容，会在绘制孩子之前，所以出现在孩子下面，而onDrawOver() 是在 RecyclerView.draw()中调用，在绘制孩子之后调用，所以会出现在孩子上方
## RelativeLayout和LinearLayout 性能
- layout和draw的性能差不多
- RelativeLayout的measure性能比LinearLayout差，因为他需要在两个方向上都measure一次才能确定孩子大小
- LinearLayout当设置weight属性后在第一次measure时候会先跳过这些孩子，然后再重新measure这些孩子
- FrameLayout测量过程会遍历所有孩子并记住孩子中的最大宽高，以此来作为自己的宽高

## WeakHashMap
- 用于存放键值对，当发生gc时，其中的键值对可能被回收。适用于对内存敏感的缓存
- 存放键值对的Entry继承自WeakReference。当发生gc时，Entry被回收并加入到ReferenceQueue中
- 访问~时，会将已经gc的键值对从中删除（通过遍历ReferenceQueue）


## `Bitmap`
- raw和drawable和sdcard，这些不带dpi的文件夹中的图片被解析时不会进行缩放（inDensity默认为160）
- 获取bitmap宽高：若inJustDecodeBounds为true，则不会把bitmap图片的像素加载到内存(实际是在Native层解码了图片，但是没有生成Java层的Bitmap)，只是获取该bitmap的原始宽（outWidth）和高（outHeight）
- 在4.4之前如果复用bitmap则不支持在native层缩放，缩放放到java层，将原来bimap基础上新建bitmap，销毁原来的，这样效率低， 4.4以后支持在native层做缩放
- 8.0，bitmap像素数据在native，6.0以前finalize释放native的bitmap，之后通过注册NativeAllocationRegistry（简化了Cleaner的使用），将native资源的大小计入GC触发的策略之中。即使java堆增长缓慢，而native堆增长快速也同样会触发gc
- Cleaner继承自虚引用，虚引用指向对象被回收时，虚引用对象会进入ReferenceQueue，异步线程ReferenceQueueDaemon会在ReferenceQueue.wait()上等待，只有对象被回收，然后遍历引用队列，若存在Cleaner则调用clean方法释放native内存
- Bitmap大小=长*宽*像素大小。像素大小
- BitmapFactory.Options
	+ 复用bitmap：加载时设置inBitmap表示使用之前bitmap对象使用过的内存 而不是重新开辟新内存（如果被复用的Bitmap == 返回的被加载的Bitmap，那么说明复用成功了）。复用条件是，图像是可变的isMutable为true
	+ inPreferredConfig
		- 只有当图片是webp 或者png24的时候，inPreferredConfig才会有效果。
		- ALPHA_8 ： 图片只有alpha值，没有RGB值，一个像素占用一个字节 
		- RGB_565:2字节
		- ARGB_8888 ： 一个像素占用4个字节
	- inSampleSize:为2的幂次，表示压缩宽高为原来的1/2，像素密度不变，按需加载，按ImageView大小加载，计算inSampleSize的算法是，原始宽高不停除2，inSampleSize不停乘2，直到原始宽高小于需求宽高。
- 回收bitmap（Bitmap.recycle()）：只是释放图片native对象的内存，并且去除图片像素数据的引用，让图片像素数据可以被垃圾回收
- Bitmap占用内存大小因素：
	1. 图片的原始宽高（即我们在图片编辑软件中看到的宽高）
	2. 解码图片时的Config配置（即每个像素占用几个字节）
	3. 解码图片时的缩放因子（即inTargetDensity/inDensity）
- BitmapRegionDecoder用于图片分块加载
- jpg 色彩丰富，没有兼容性问题，不支持透明度，和动画，适用于摄影作品。jpg在高对比度的场景下效果不好，比如黑色文字在白色的背景上
- png包括透明度适用于图标，因为大面积的重复颜色，适用于无损压缩
- webp包括有损和无损两个方式，webp的浏览器支持不佳，webp支持动画和透明度，之前动画只能用gif，透明度只能选png，有损压缩后的webp的解码速度慢，比gif慢2倍，webp支持的颜色比gif多
- 图片缩放比例 scale = 设备分辨率 / 资源目录分辨率  如：1080x1920的图片显示xhdpi中的图片，scale = 480 / 320 = 1.5，图片的宽高会乘以scale
## 缩包
analyze apk---res asset lib dex 大小
1.gradle shrinkResources 删除无用资源
2.压缩图片资源
3.proguard 混淆
4.有些图片可以用shape来代替
5. 将所有图片 webp 化
6. AndResGuard 将长资源名称变短

## overload和override
- overload具有相同名称的不同方法，通过参数列表区分，和返回值无关
- override相同方法的不同实现，参数和返回值必须一样
## res和asset区别
- 两者都会打包入apk
- res中资源会被映射成R文件，除了raw子目录之外所有资源都会被编译成二进制
- asset中资源相当于文件，不会被编译成二进制

## 进程优先级
- 一共有五个进程优先级
1、 前台进程(Foreground process)：该进程中有前台组件正在运行，oom_adj：FOREGROUND_APP_ADJ=0
	 - 正在交互的Activity，Activity.onResume()
	 - 前台服务
	 - Service.onCreate() onStart()正在执行
	 - Receiver.onReceive()正在执行
2、 可见进程(Visible process) VISIBLE_APP_ADJ = 1
	- 正在交互的Activity，Activity.onPause()
3、 服务进程(Service process)
	- 后台服务
4、 后台进程(Background process) BACKUP_APP_ADJ = 3
	- 不可见Activity Activity.onStop()
	- 后台进程优先级等于后台服务，所以长时间后台任务最后其服务
5、 空进程(Empty process)：不包含任何组件的进程，Activity 在退出的时候进程不会销毁, 会保留一个空进程方便以后启动. 但在内存不足时进程会被销毁；

按下返回键退出应用，此时应用进程变成缓存进程，随时可能被杀掉
按下home键退出应用，此时应用是不可见进程

## LruCache
- ~是内存缓存，持有一个 LinkedHashMap 实例
- ~用LinkedHashMap作为存储结构，且LinkedHashMap按访问顺序排序，最新的结点在尾部，最老的结点在头部
## LinkedHashMap
- 是一个有序 map，可以按插入顺序或者访问顺序排列
- 在 hashMap 基础上增加了头尾指针形成双向链表，继承 Node 添加前后结点的指针，每次构建结点时会将他链接到链尾。
- 若是按访问顺序排序，存取键值对的时候会将其拆下插入到链尾，链头是最老的结点，满时会被移出
- 按访问顺序来排序是LRU缓存的一种实现。
## diskLruCache
- 内部有一个线程池（只有一个线程）用于清理缓存
- 内部有一个LinkedHashMap结构代表内存中的缓存，键是key，值是Entry实体（key+file文件）
- ~有一个journal文件缓存操作的日志文件，构造时会读取日志文件并将其转换成LinkedHashMap存储在内存
- 取缓存时，先读取内存中的Entry，然后将其转换成Snapshot对象，可以从Snapshot中拿到输入流
- 写缓存时，新建Entry实体并存在LinkedHashMap中，将其转换成Editor对象，可以从中拿到输出流
- 通过LinkedHashMap实现LRU替换
- 每一个Cache项有四个文件，两个状态（DIRTY,CLEAN）,每个状态对应两个文件：一个文件存储Cache meta数据，一个文件存储Cache内容数据
## 多线程访问数据库
- 实现多线程读写的关键是enableWriteAheadLogging属性，这个方法 API Level 11添加的，也就是所3.0以上的版本就基本不可能实现真正的多线程读写了。简单的说通过调用enableWriteAheadLogging()和disableWriteAheadLogging()可以控制该数据是否被运行多线程读写，如果允许，它将允许一个写线程与多个读线程同时在一个SQLiteDatabase上起作用。实现原理是写操作其实是在一个单独的log文件，读操作读的是原数据文件，是写操作开始之前的内容，从而互不影响。当写操作结束后读操作将察觉到新数据库的状态。当然这样做的弊端是将消耗更多的内存空间。
## Lifecycle
- 让任何组件可以作为观察者观察界面生命周期
- 通过LifecycleRegistry，它持有所有观察者，通过注册ActivityLifecycleCallbacks 实现生命周期的分发，如果是29 以下则将ReportFragment添加到activity中
- 监听应用前后台切换：通过registerActivityLifecycleCallbacks，然后在维护一个活跃activity的数量，ProcessLifecycleOwner为我们做了这件事情 用于监听应用前后台切换，ProcessLifecycleOwner的初始化通过ContentProvider实现



## 恢复Activity数据
1. onSaveInstanceState()+onRestoreInstanceState():会进行序列化到磁盘，耗时，杀进程依然存在
2. Fragment+setRetainInstance()：数据保存在内存，配置发生变化时数据依然存在，但杀进程后数据不存在
3. onRetainNonConfigurationInstance() + getLastNonConfigurationInstance():数据保存在内存，配置发生变化时数据依然存在，但杀进程后数据不存在



## 类加载
- 编译：javac 命令把 .java 文件编译成字节码（.class 文件）
- 运行：jvm执行.class
- 类加载过程：加载---链接---初始化
1. 类加载：jvm把.class作为二进制流读入内存，并实例化一个Class对象，jvm 并不是一次性把所有类都加在到内存，而是执行过程中遇到没有加载的才加载，并只加载一次（Android加载的dex）
3. 验证：二进制合法性校验
4. 准备：为类变量在方法区赋初始值
5. 解析：将类名，方法名，字段名替换为内存地址
6. 初始化：对类的主动引用，包括new 调用静态方法，使用静态字段
7. 使用：
8. 卸载：
统计类加载耗时：反射BaseDexClassLoader的 pathList，写入自定义的 PathClassLoader（装饰者增加耗时统计）
### 类加载器 PathClassLoader
只能加载应用包内的dex
### 类加载器 DexClassLoader
可以加载任意位置的 dex
### 类加载器 BaseDexClassLoader
- 持有 DexPathList
- BaseDexClassLoader（DexPathList（Element数组（DexFile（多个Class））））
### DexPathList
- 将 dex 文件转换成 element 存入数组（dexElements）
- findClass()是遍历Elements并进行类名匹配。
### Android 类加载过程
- Dex 文件在类加载器中被包装成 Element，Element以数组形式被DexPathList 持有，加载类时通过遍历Element数组进行类名匹配查找，只要把新的Dex文件插入到 Element头部即可实现热修（反射）
- 双亲委托：类加载器加载类时，将加载请求逐级向上委托，直到BootStrapClassloader，真正的加载从顶层开始，逐级向下查找。避免了类重复加载，以及安全，因为系统类总是由上层类加载器加载，无法通过自定义篡改
- 先父亲，再孩子
- 先静态后非静态
- 先字段，后构造器（字段先后有定义顺序决定）
- 先代码块 后构造方法

# 注解
注解为代码添加一些额外的信息，以便稍后可以读取这些信息。这些信息可以帮助代码检查，编译时生成代码以减少模板代码
## 元注解
1. @Retention：定义注解生命周期
	- RetentionPoicy.SOURCE注解只保留在源文件，当Java文件编译成class文件的时候，注解被遗弃；用于做一些检查性的操作，比如 @Override
	- RetentionPoicy.CLASS:注解被保留到class文件，但jvm加载class文件时候被遗弃，这是默认的生命周期；用于在编译时进行一些预处理操作，比如生成一些辅助代码（编译时注解即是编写生成代码的代码），ButterKnife 使用编译时注解，即在编译时通过自定义注释解析器AbstractProcessor读取注解并由此生成java文件（在里面调用了 findViewById）
	- RetentionPoicy.RUNTIME:注解不仅被保存到class文件中，jvm加载class文件之后，仍然存在；用于在运行时去动态获取注解信息
2. @Target:定义了Annotation所修饰的对象范围


# LiveData
LiveData 的数据观察者在内部被包装成另一个对象（实现了 LifecycleEventObserver 接口），它同时具备了数据观察能力和生命周期观察能力
LiveData 内部会将数据观察者进行封装，使其具备生命周期感知能力。当生命周期状态为 DESTROYED 时，自动移除观察者
LiveData 的观察者会维护一个“值的版本号”，用于判断上次分发的值是否是最新值，“新观察者”被“老值”通知的现象叫“粘性”。因为新观察者的版本号总是小于最新版号，且添加观察者时会触发一次老值的分发
在高频数据更新的场景下使用 LiveData.postValue() 时，会造成数据丢失。因为“设值”和“分发值”是分开执行的，之间存在延迟。值先被缓存在变量中，再向主线程抛一个分发值的任务。若在这延迟之间再一次调用 postValue()，则变量中缓存的值被更新，之前的值在没有被分发之前就被擦除了。

# 虚拟内存
 - 每个应用访问的地址空间是虚拟地址空间，所以可以无穷大，Linux负责将虚拟内存转换为物理地址。
 - 为了方便将虚拟内存地址和物理地址进行映射，内存空间被分割成若干个页（通常是4kb大小）
 - Memory Management Unit（MMU）这个硬件专门用于将虚拟地址转换为物理地址。它通过查询映射表得到物理地址
 - 虚拟地址分为高4位的页号，和后面的偏移量，每次通过页号查询映射表得到物理地址的页号，然后再将偏移量拼在后面得到物理地址

## equals()
- equals() 定义在JDK的Object.java中。可以定义两个对象是否相等的逻辑
- "=="相等判断符用于比较基本数据类型和引用类型数据。 当比较基本数据类型的时候比较的是数值，当比较引用类型数据时比较的是引用(指针)即指向堆内存的地址
- ==的语义是固定的，而equals()的语义是自定义的
## hashCode()
- hashCode() 的作用是获取哈希码，它实际上是返回一个int整数。仅仅当创建并某个“类的散列表”(关于“散列表”见下面说明)时，该类的hashCode() 才有用，作用是：确定该类的每一个对象在散列表中的位置，Java集合中本质是散列表的类，如HashMap，Hashtable，HashSet
- HashMap 如果使用equals判断key是否重复，需要逐个比较，时间复杂度为O(n),但如果使用hashCode()，因为它是一个int值。所以可以直接作为数组结构的某个索引值,如果该索引位置没有内容则表示key没有重复，复杂度为O(1)

1)、如果两个对象相等，那么它们的hashCode()值一定相同。这里的相等是指，通过equals()比较两个对象时返回true。
2)、如果两个对象hashCode()相等，它们并不一定相等。因为在散列表中，hashCode()相等，即两个键值对的哈希值相等。然而哈希值相等，并不一定能得出键值对相等。补充说一句：“两个不同的键值对，哈希值相等”，这就是哈希冲突。


## Fragment
- fragment保存状态 setRetainInstance(true)
	- 表示activity重建时保持fragment实例
    - fragment 实例被保存在FragmentManagerViewModel中的HashSet结构中
    - 在Activity.onCreate()执行时 若 savedInstanceState 不为空 则委托FragmentController恢复fragment, FragmentController 持有 FragmentManager
 - 使用Fragment保存配置切换的数据：将Fragment设置为setRetainInstance(true)，在Activity.onCreate()中通过FragmentManager获取该Fragment，如果实例不空则表示是因为配置发生变化，可以从Fragment中的业务字段获取保存值。

## ViewModel
- ViewModel 实例被存储在ViewModelStore的map中, 在配置发生变化时onRetainNonConfigurationInstance会被调用（ViewModelStore的map对象会被存储在NonConfigurationInstances中），在恢复ViewModel时再getLastNonConfigurationInstance中再次获取
- 先从 stroe 拿，若 Store 中没有，则通过Factory 构造一个（没有指定factory 就反射出一个ViewModel 实例并存入strore）
- Activity 实现了LifecycleOwner，等onDestroy时会尝试（非配置变化时）调用 store的 clear(遍历了 viewmodel的clear)
- ViewModel 在 Fragment 中不会因配置改变而销毁的原因其实是因为其声明的 ViewModel 是存储在 FragmentManagerViewModel 中的，而 FragmentManagerViewModel 是存储在宿主 Activity 中的 ViewModelStore 中，又因 Activity 中 ViewModelStore不会因配置改变而销毁，故 Fragment 中 ViewModel 也不会因配置改变而销毁。
 - ViewModel 的构建被抽象为 ViewModelProvider.Factory。
- ViewModel 的实例被存储在 ViewModelStore 中的 HashMap 结构中，ViewModelStore 的构建被抽象为 ViewModelStoreOwner，Activity 和 Fragment 都实现了该接口且持有了 ViewModelStore 的实例，为了避免横竖屏切换时 ViewModelStore 被重新构建。它还被一个静态类 NonConfigurationInstances，以保证横竖屏切换时可以从中恢复。 
- ViewModel 的获取通过 ViewModelProvider 实现，它屏蔽了通过 ViewModelProvider.Factory 构建以及通过 ViewModelStore 缓存 ViewModel 实例的细节。
- 这套存储机制使得 ViewModel 生命周期比 Activity 更长。当 Activity 销毁重建时，就不会重新触发业务逻辑。

## sealed class
- 是一个继承结构固定的抽象类，即在编译时已经确定了子类数量，不能在运行时动态新增
- 它的子类只能声明在同一个包名下
- 是一个抽象类，且构造方法是私有的，它的孩子是final类，而且孩子声明必须嵌套在sealed class内部。
- 枚举的局限性
	限制枚举每个类型只允许有一个实例
	限制所有枚举常量使用相同的类型的值

## launch mode
taskAffinity与allowTaskReparenting配合：我们可以在AndroidManifest.xml为Activity配置android:allowTaskReparenting属性，表示允许此Activity更换其从属的任务栈。设置此属性的Activity一但当前Task切换到了后台，就会回到它“倾向”的任务栈中
1. singleTask
全局唯一
总结：先检查是否有和待启动Activity的taskAffinity相同的task（若未显示指定，则默认和app的第一个activity拥有相同的taskAffinity，为包名），若无则新建task并新建Activity压栈，若有则把该task移到前台并在该task中寻找待启动activity,若找到则将该task中该activity之上的所有activity弹出,让其成为栈顶(此时onCreate()不被调,onNewIntent()被调)，若没有找到则新建Activity实例并压栈
2. singleInstance
全局唯一
如果不存在待启动Activity，则新建task来容纳待启动activity,并且该task不能放入其他activity，若存在，则将对应的task移到前台
该类型只能在manifest中指定，不能通过Intent.setFlag()
该类型只能作用于Task栈底的Activity
3. standard
全局不唯一
每次都在当前task中新建一个实例(待启动activity.onCreate()每次都会被调用)
4. singleTop
全局不唯一
只有当Activity位于task的栈顶时,该activity实例才会被重复利用(onNewIntent()被调用而不是onCreate()),否则都会新建实例


## ThreadLocal
- 用于将对象和当前线程绑定（将对象存储在当前线程的ThreadLocalMap结构中）
- ThreadLocalMap是一个类似HashMap的存储结构，键是ThreadLocal对象的弱引用，值是要保存的对象
- set()方法会获取当前线程的ThreadLocalMap对象
- threadLocal内存泄漏：key是弱引用，gc后被回收，value 被entry持有，再被ThreadLocalMap持有，再被线程持有，如果线程没有结束，则value无法访问到，也无法回收，方案是及时remove掉不用的value
- threadlocal 会自动清理key为null 的entry

## crossinline
在具有inline 特性的同时，避免非局部返回，因为直接return掉函数会影响原有功能，crossinline的lambda内部必须使用局部返回，比如return@foo
## noinline
拒绝对内联函数中的 lambda参数进行内联，如果参数是 lambda 被内联之后，就不能当对象使用了

## kotlin 空安全
- 是通过if判空实现空安全的
- kotlin和java交互的时候空安全被破坏
	+ kotlin调用java：java的返回值是可空的，但是kotlin调用是没有使用?，可以通过在java代码添加@NotNull注解进行非空约束
	+ java 调用kotlin：kotlin的参数规定是非空的，但是java调用时可以传入null。


## 持久化
- 内部存储路径：/data/user/0/包名 等价于/data/data/包名，其中0表示第一个用户
- 外部存储路径：/storage/emulated/0/

19. 电量优化
20. fragment的滑动
21. Https请求慢的解决办法

- 新问题
1. NDK JNI 如何加载NDK库？如何在jni中注册native函数，有几种注册方法？
3. 热修复

- 生僻
1. int,long的取值范围以及BigDecimal，数值越界了如何处理
3. 广播传输数据有什么限制
