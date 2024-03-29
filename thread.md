# 进程线程
- 系统按进程分配除CPU以外的系统资源(主存 外设 文件) 系统按线程分配CPU资源
- Android系统进程叫system_server，默认情况下一个Android应用运行在一个进程中，进程名是应用包名，进程的主线程叫ActivityThread
- jvm会等待普通线程执行完毕，但不会等守护线程
- 若线程执行发生异常会释放锁
- 线程上下文切换：cpu控制权由一个运行态的线程转交给另一个就绪态线程的过程（需要从用户态到核心态转换）
- 一对一线程模型：java语言层面的线程会对应一个内核线程
- 抢占式的线程调度，即由系统决定每个线程可以被分配到多少执行时间
- 阻塞线程的方法
  1. sleep()：到阻塞态，但不释放锁，会触发线程调度。
  2. wait():到阻塞态，释放锁（必须先获取锁）。
  3. yield():到就绪态，主动让出cpu，不会释放锁，发生一次线程调度，同优先级或者更高优先级的线程有机会执行。

## 线程安全
原子性：不会被线程调度器中断的操作。
可见性：一个线程中对共享变量的修改，在其他线程立即可见。
有序性：程序执行的顺序按照代码的顺序执行。

### 原子操作包括
1. 除long和double之外的基本类型（int, byte, boolean, short, char, float）的赋值操作。
2. 所有引用reference的赋值操作，不管是32位的机器还是64位的机器。
3. java.concurrent.Atomic.* 包中所有类的原子操作。

## 死锁
- 四个必要条件
1. 互斥访问资源
2. 资源只能主动释放，不会被剥夺
3. 持有资源并且还请求资源
4. 循环等待
解决方案是：加锁顺序+超时放弃

## 线程池
- 如果创建对象代价大，且对象可被重复利用。则用容器保存已创建对象，以减少重复创建开销，这个容器叫做池

## 多进程
- 分散内存，每个进程内存空间有限
- 子进程崩溃 主进程可以继续运行，主进程退出，子进程可以继续工作
- 进程相互守护
- 缺点
  1. 静态成员和单例失效：静态其实是通过内存来共享数据，但不同进程都有自己独立的内存空间
  2. sp存在同步为：sp通过读写xml文件实现共享数据，但多进程并发读会出问题
  3. application会创建多次：新建进程需要独立分配虚拟机和创建应用的过程类似

## 等待通知机制
- ~是一种线程间的通信机制，可以调整多个进程的执行顺序
- 线程生命周期：线程从新建状态到就绪状态，就绪态的线程如果获得了cpu执行权就变成了运行态，运行完变成死亡态，如果运行中产生等待锁的情况（sleep，wait），则会进入阻塞态，当阻塞态的进程被唤醒后进入就绪态，参与cpu时间片的竞争，执行完毕死亡态
2. notify()：随机使一个线程进入就绪态，它需要和调用wait()是同一个对象（获得锁的线程才能调用）
3. notifyAll()：唤醒所有等待线程，让他们到就绪队列中

## `interrupt()`
- 不会真正中断正在执行的线程，只是告诉通知它你应该被中断了，自己看着办吧。
- 若线程正运行，则中断标志会被置为true，并不影响正常运行
- 如果线程正处于阻塞态，则会收到InterruptedException，就可以在 catch中执行响应逻辑
- 若线程想响应中断，则需要经常检查中断标志位，并主动停止，或者是正确处理 InterruptedException

## 内存屏障
- 用于静止重排序
- LoadLoad  Load1; LoadLoad; Load2  确保Load1数据的装载，之前于Load2及所有后续装载指令的装载。
StoreStore  Store1; StoreStore; Store2  确保Store1数据对其他处理器可见（刷新到内存），之前于Store2及所有后续存储指令的存储。
LoadStore Load1; LoadStore; Store2  确保Load1数据装载，之前于Store2及所有后续的存储指令刷新到内存。
StoreLoad Store1; StoreLoad; Load2  确保Store1数据对其他处理器变得可见（指刷新到内存），之前于Load2及所有后续装载指令的装载。StoreLoad Barriers会使该屏障之前的所有内存访问指令（存储和装载指令）完成之后，才执行该屏障之后的内存访问指令。

## 多线程访问文件
1. 使用文件锁：文件锁是一种机制，允许文件在同时被多个进程或线程访问时保持同步。在Android中，可以使用Java NIO中的FileChannel来获取文件锁，或者使用Java IO中的FileLock类。
2. 使用互斥锁：在多个进程或线程访问同一文件时，可以使用互斥锁来控制对文件的访问。在Android中，可以使用Java中的synchronized关键字或者ReentrantLock类来实现互斥锁。
3. 使用文件描述符：每个文件在Linux内核中都有一个文件描述符（file descriptor），可以通过这个描述符来控制对文件的访问。在Android中，可以使用Java中的FileDescriptor类来获取文件描述符，从而实现对文件的控制。

## volatile
- 保证变量操作的有序性和可见性
在每一个volatile写操作前面插入一个StoreStore屏障，可以保证在volatile写之前，其前面的所有普通写操作都已经刷新到主内存中。
在每一个volatile写操作后面插入一个StoreLoad屏障，避免volatile写与后面可能有的volatile读/写操作重排序。
在每一个volatile读操作后面插入一个LoadLoad屏障，禁止处理器把上面的volatile读与下面的普通读重排序。
在每一个volatile读操作后面插入一个LoadStore屏障，禁止处理器把上面的volatile读与下面的普通写重排序。
- volatile就是将共享变量在高速缓存中的副本无效化，这导致线程修改变量的值后需立刻同步到主存，读取共享变量都必须从主存读取 
- 当volatile修饰数组时，表示数组首地址是volatile的而不是数组元素
- 通过阻止编译器重排序保证有序性
- 双重锁实例必须带有volatile，因为`INSTANCE = new instance()`不是原子操作，由三个步骤实现1.分配内存2.初始化对象3.将INSTANCE指向新内存，当重排序为1,3,2时，可能让另一个线程在第一个判空处返回未经实例化的单例。volatile保证了有序性
- volatile vs AtomicBoolean：当只有一个写线程多个读线程的时候，volatile能保证线程安全，当有多个读线程和多个写线程时必须使用AtomicBoolean

## JMM
- jmm = Java Memory Modle
- JMM 有一些规则保证了线程安全
  1. as-if-serial：编译器和处理器不会对存在数据依赖关系的操作做重排序，数据依赖分为三种情况1。读后写 2。写后写 3。写后读，这三种情况不允许重排序
  2. 

## CAS
- Compare and Swap
- 当前值，旧值，新值，只有当旧值和当前值相同的时候，才会将当前值更新为新值
- Unsafe将cas编译成一条cpu指令，没有函数调用
- aba问题：当前值可能是变为b后再变为a，此a非彼a，通过加版本号能解决
- 非阻塞同步：没有挂起唤醒操作，多个线程同时修改一个共享变量时，只有一个线程会成功，其余失败，他们可以选择轮询


## 锁
- 锁机制保证了只有获得锁的线程能够操作锁定的内存区域。
- 除了偏向锁，JVM实现锁的方式都用到的循环CAS，当一个线程想进入同步块的时候使用循环CAS的方式来获取锁，当它退出同步块的时候使用循环CAS释放锁。
- 公平锁：等待线程拍成队列，先等待先获得锁，不会产生饥饿。

### `synchronized`
- 隐式加锁释放锁
- 可修饰静态方法，实例方法，代码块
- 当修饰静态方法的时，锁定的是当前类的 Class 对象（就算该类有多个实例，使用的还是同一把锁）。
- 当修饰非静态方法的时，锁定的是当前实例对象 this。当 饰代码块时需要指定锁定的对象。
- 使用时的常见问题是用不同的锁保护同一个对象
- 通过将对变量的修改强制刷新到内存，且下一个获取锁的线程必须从内存拿。保证了可见性
- 同一时间只有一个线程可以执行临界区，即所有线程是串行执行临界区的
- happen-before 就是释放锁总是在获取锁之前发生。
- synchronized特点
  1. 可重入：可重入锁指的是可重复可递归调用的锁，在外层使用锁之后，在内层仍然可以使用，并且不发生死锁。线程可以再次进入已经获得锁的代码段，表现为monitor计数+1
  2. 不公平：synchronized 代码块不能够保证进入访问等待的线程的先后顺序,因为非公平的性能略好，可能可以避免一次上下文切换（因为进等待队列必须会切换到阻塞态）
  3. 不可中断：线程获取不到锁一直等待
  4. 无法判断锁状态：不知道当前线程是否获取了锁
  5. 不灵活：synchronized 块必须被完整地包含在单个方法里。而一个 Lock 对象可以把它的 lock() 和 unlock() 方法的调用放在不同的方法里
  6. 自旋锁（spinlock）：是指当一个线程在获取锁的时候，如果锁已经被其它线程获取，那么该线程将循环等待，然后不断的判断锁是否能够被成功获取，直到获取到锁才会退出循环，synchronized是自旋锁。如果某个线程持有锁的时间过长，就会导致其它等待获取锁的线程进入循环等待，消耗CPU。使用不当会造成CPU使用率极高，jdk1.4之前使用自旋实现synchronized，1.4之后根据不同情况选择是自旋还是等待-通知机制，如果同步块执行时间短，则选择自旋，

- 1.8 之后synchronized性能提升：
    * 偏向锁：目的是消除无竞争状态下性能消耗，假定在无竞争，且只有一个线程使用锁的情况下，在 mark word中使用cas 记录线程id（Mark Word存储对象自身的运行数据，在对象存储结构的对象头中）此后只需简单判断下markword中记录的线程是否和当前线程一致，若发生竞争则膨胀为轻量级锁，只有第一个申请偏向锁的会成功，其他都会失败
    * 轻量级锁：使用轻量级锁，不要申请互斥量，只需要用 CAS 方式修改 Mark word，若成功则防止了线程切换
    * 自旋（一种轻量级锁）：竞争失败的线程不再直接到阻塞态（一次上下文切换，耗时），而是保持运行，通过轮询不断尝试获取锁（有一个轮询次数限制），规定次数后还是未获取则阻塞。进化版本是自适应自旋，自旋时间次数限制是动态调整的。
    * 重量级锁：使用monitorEnter和monitorExit指令实现（底层是mutex lock），每个对象有一个monitor
    * 锁膨胀是单向的，只能从偏向->轻量级->重量级

### `ReentrantLock`
- 手动加锁手动释放：JVM会自动释放synchronized锁，但可重入锁需要手动加锁手动释放，要保证锁定一定会被释放，就必须将unLock()放到finally{}中。手动加锁并释放灵活性更高
- 1. 可中断锁：lockInterruptibly(),未获取则阻塞，但可响应当前线程的interrupt()被调用
- 2. 超时锁：tryLock(long timeout, TimeUnit unit)  ,未获取则阻塞，但阻塞超时。
- 3. 非阻塞获取锁：tryLock() ，未获取则直接返回
- 4. 可重入：若已获取锁，无需再次竞争即可重新进入临界区执行，state +1，出来的时候需要释放两次锁 state -1
- 5. 独占锁：同一时间只能被一个线程获取，其他竞争者得等待（AQS独占模式）
- 性能：竞争不激烈，Synchronized的性能优于ReetrantLock，激烈时，Synchronized的性能会下降几十倍，但是ReetrantLock的性能能维持常态
- 是AQS的实现类，AQS中有一个Node节点组成双向链表，存放等待的线程对象（被包装成Node）
- 获取锁流程：
  1. 尝试获取锁
    公平锁排队逻辑：判断锁是否空闲，若空闲还要判断队列中是否有排在更前面的等待线程，若无则尝试获取锁。若当前独占线程是自己，表示重入，则增加state值
    非公平锁抢占逻辑:直接进行CAS state操作（从0到1），若成功则设置当前线程为锁独占线程。若失败会判断当前线程是否就是独占线程若是表示重入，state+1
  2. 获取失败则入AQS队列，然后在挂起线程:将线程对象包装成EXCLUSIVE模式的Node节点插入到AQS双向链表的尾部（cas值链尾的next结点+自旋保证一定成功），并不断尝试获取锁，或中断Thread.interrupted()
- 释放锁流程：
  1. 释放锁表现为将state减到0
  2. 调用unparkSuccessor()唤醒线程（非公平时如何唤醒）

## `ReentrantReadWriteLock`
- 并发度比ReentrantLock高，因为有两个锁，使用AQS，读锁是共享模式，写锁是独占模式。读多写少的情况下，提供更大的并发度
- 可实现读读并发，读写互斥，写写互斥
- 使用一个int state记录了两个锁的数量，高16位是读锁，低16位是写锁
- 获取写锁过程：除了考虑写锁是否被占用，还要考虑读锁是否被占用，若是则获取锁失败，否则使用cas置state值，成功则置当前线程为独占线程。
- 读并发也有并发数限制，获取读锁时需验证，并使用ThreadLocal记录当前线程持有锁的数量
- 可能发生写饥饿，因为太多读
- 锁降级：当一个线程获取写锁后再获取读锁，然后释放写锁
- 不支持锁升级是为了保证可见性：多个线程获取读锁，其中任意线程获取写锁并更新数据，这个更新对其他读线程是不可见的

#### `Condition`
- 是多线程通信的机制，挂起一个线程，释放锁，直到另一个线程通知唤醒，提供了一种自动放弃锁的机制。
- await()挂起线程的同时释放锁（所以必须先获取锁，否则抛异常），signal 唤醒一个等待的线程
- 每个Condition对象只能唤醒调用自己的await()方法的那个线程
- 如果条件不用 Condition 实现，则线程可能不断地获取锁并释放锁，但因继续执行的条件不满足，cpu 负载打满。使用Condition 让等待线程不消耗cpu
- await() 通常配合 while(){await()} 因为被唤醒是从上次挂起的地方执行，还需要再次判断是否满足条件
- await()必须在拥有锁的情况下调用，以防止lost wake-up，即在await条件判断和await调用之间notify被调用了,当await条件满足后，还没来得及执行await时发生线程调度，另一个线程调用了notify(),然后刚才的await()又被执行了，它将错过刚才的notify，因为notify在await之前执行。

## StampedLock
- 实现读读并发，读写并发。
- 在读的时候如果发生了写，应该通过重试的方式来获取新的值，而不应该阻塞写操作
- 用二进制位记录每一次获取写锁的记录

### CountdownLatch
- 用作等待若干并发任务的结束
- 内部有一个计数器，当值为0时，在countdownLatch上await的线程就被唤醒
- 通过AQS实现，初始化是置AQS.state为n，countdow()通过自旋+cas将执行state--效果

### CyclicBarrier
- 用于同步并发任务的执行进度
- 使用 ReentranntLock 保证count线程安全，每次调用await（） count--，然后在condition上阻塞，当count为0时，会signalAll()
-  和CountdownLatch区别，可重用性:
  +  CountdownLatch是一个一次性的对象,计数器无法被重置。一旦计数器变为0,对象不能再被使用。
  + CyclicBarrier对象可以被重复使用,因为其计数器可以循环重置。

## Semaphore
- 用于限制并发访问资源线程的个数
- 基于aqs，初始化是为state赋值最大并发数，调用acquire()时即是cas将state-1，当state<0时，令牌不足，将线程包装成node结点会入队列，然后挂起
- 有公平和非公平两个构造方法

### `AbstractQueuedSynchronizer`
- 实现了cas方式竞争共享资源时的线程阻塞唤醒机制
- aqs提供了两种资源共享方式1.独占：只有一个线程能获取资源（公平，不公平）2.共享：多个进程可获取资源
- aqs使用了模板方法模式，子类只需要实现tryAcquire()和tryRelease()，等待队列的维护不需要关心
- aqs使用了CLH 队列:包括sync queue和condition queue，后者只有使用condition的时候才会产生
- 持有一个volatile int state代表共享资源，state =1 表示被占用（提供了CAS 更新 state 的方法），其他线程来加锁会失败，加锁失败的线程会放入等待队列（被Unsafe.park()挂起）
- 等待队列的队头是独占资源的线程。队列是双向链表

### AtomicInteger
- 线程安全的int值，为了避免在一个变量上使用锁实现同步，这样太重了
- 在高并发的情况下，每次值成功更新了都需要将值刷到主存
- 自增使用cas+自旋+volatile
```
    public final int incrementAndGet() {
        for (;;) {
            int current = get();
            int next = current + 1;
            if (compareAndSet(current, next))
                return next;
        }
    }
```

### `AtomicIntegerArray`
- 线程安全整形数组
- 内部持有 final int[] array，保证了使用时已经初始化，并且引用不能改变
- 对数组的操作退化为对单个元素的操作，因为数组的内存模型是连续的，并且每个元素所占空间一样大，
- 使用Unsafe带有volatile的方法进行对单个元素赋值。

### `AtomicReference`
- 提供对象引用非阻塞原子性并发读写
- 对象引用是一个4字节的数字，代表着堆内存中的一个地址

### `CopyOnWriteArrayList`
- 实现了线程安全的读写并发，读读并发，但不能实现写写并发（上锁了，synchronized），因为他们操纵是不同的副本
- 使用不可变 Object[] 作为容器
- 写时复制数组，写入新数组，引用指向新数组。加锁防止一次写操作导致复制了多个数组副本
- 读操作就是普通的获取数组某元素，
- 适合读多写少，因为写需要复制数组，耗时
- 适合集合不大，因为写时要复制数组，耗时，耗内存
- 实时性要求不高，因为可能会读到旧数据。(对新数组写，对数组读)
- 采用快照遍历，即遍历发起时形成一张当前数组的快照，并且迭代器不允许删除，新增元素。不会发生ConcurrentModificationException，但可能实时性不够。
- 适用于作为观察者的容器

### `ArrayBlockingQueue`
- 大小固定的，线程安全的顺序队列，不能读读，读写，写写并发。
- 使用Object[]作为容器，环形数组。比复制拷贝效率高。
- 存取使用同一把锁 ReentrantLock 保证线程安全+2个condition（写操作可能在notFull条件上阻塞，读操作可能在notEmpty上阻塞）
- 遍历支持remove及并发读写。
- 适用于控制内存大小的生产者消费者模式，队列满，则阻塞生产者/有限等待，队列空则阻塞消费者/有限等待。
- 适用于做缓存，缓存有大小限制，缓存是生产者消费者模型，多线程场景下需考虑线程安全。

### `LinkedBlockingQueue`
- 线程安全的链队列，实现读写并发，不能读读，写写并发
- 存取用两把不同的 ReentrantLock，适用于读写高并发场景。
- 可实现并行存取，速度比 ArrayBlockingQueue 快，但有额外的Node结点对象的创建和销毁，可能引发 gc，若消费速度远低于生产速度，则内存膨胀

### `SynchronousQueue`
- 以非阻塞线程安全的方式将元素从一个生产线程递交给消费线程
- 适用于生产的内容总是可以被瞬间消费的场景，比如容量为 Int.MAX_VALUE 的线程池，即当新请求到来时，总是可以找到新得线程来执行请求，不管是老的空闲线程，还是新建线程。
- 存储结构是一个链，使用 cas + 自旋的方式实现线程安全

### `PriorityBlockingQueue`
- 使用 Object[] 作为容器实现堆
- 使用 ReentrantLock 保证线程安全，读和取同一把锁
- 每次存取会引发排序，使用堆排序提高性能
  1. 每次写，都写到数组末尾，然后堆向上调整
  2. 每次读都读取数组头，并将数组末尾元素放到数组头。然后执行一次向下调整

### `ConcurrentLinkedQueue`
- 是一个链式队列，使用非阻塞方式实现并发读写的线程安全，而是使用轮询+`CAS`保证了修改头尾指针的线程安全
- 存储结构是带头尾指针的单链表。
- 头尾指针和结点next域都使用 volatile 保证可见性。
- 出队时，通过 cas 保证结点 item 域置空的线程安全，更新头指针也使用了 cas。
- 入队时，通过 cas 保证结点 next 域指向新结点的线程安全，更新尾指针也使用了 cas。
- 出于性能考虑，头尾指针的更新都是延迟的。每插入两个结点，更新一下尾指针，每取出两个结点，更新一下头指针。
- 适用于生产者消费者场景
- 入队算法：总是从当前tail指向的尾部向后寻找真正的尾部（因为tail更新滞后，并且可能被另一个入队线程抢占），找到后通过cas值next域

### `ConcurrentHashMap`
- 1.7 使用的是开散列表，数组+链表
- 1.7 Segment 数组，Segment 是一个 ReentrantLock，分段锁，并发数是 Segment 的数量。每个 Segment 持有一个 Entry 数组（桶）。（一个entry就是一条链）
- 1.7 定位一个元素需要两次hash，先定位到 Segment 再定位到元素所在链表头。
- 1.7 put()先尝试非阻塞的获取锁，失败则自旋重试，并计算出对应桶的位置，到达最大尝试次数后阻塞式的获取。
- 1.7 ConcurrentHashMap.get()不需要上锁是因为键值对实体中将value声明成了volatile，能够在线程之间保持可见性
- 1.7 如果ConcurrentHashMap的元素数量增加导致ConrruentHashMap需要扩容，ConcurrentHashMap不会增加Segment的数量，而只会增加Segment中链表数组的容量大小，扩容的时候首先会创建一个两倍于原容量的数组，然后将原数组里的元素进行再 hash 后插入到新的数组里
- 1.7 遍历链表是个相对耗时的操作
- 1.8 将重入锁改成synchronized，因为它被优化过了
- 1.8 也是开散列表，数组+链表（红黑树）
- 1.8 当链表长度大于8时，则将链表转换为红黑树，增加查找效率
- 1.8 使用cas方式保证只有一个线程初始化成功
- 1.8 put操作：对key进行hash得到数组索引，若数组未初始化则初始化，如果索引位置为null 则直接cas写（失败则自旋保持成功），（后面的部分synchronize了）如果索引位置为链头指针，则进行链插入，往后遍历找到相同的key 则覆盖，否则链尾插入，若索引位置是红黑树，则进行红黑树插入。
- 1.8 锁的粒度更细了，一个桶一个锁。
- 1.8 Node.next 用volatile修饰

### 红黑树
- 红黑树是平衡二叉排序树的升级版
- 二叉排序树是二分查找，最大查找次数为树高度
- 二叉排序树的缺点是，当插入元素是有序的时，二叉树变成链表，查找需要遍历所有结点，复杂度为O(n)
- 插入结点后通过变色和旋转来保持红黑树的平衡
- 左旋就是右孩子向父亲的方向旋转，并将右孩子的左孩子换父亲
- 二叉树删除算法：
  1. 要删除的节点正好是叶子节点，直接删除就 OK 了；
  2. 只有左孩子或者右孩子，直接把这个孩子上移放到要删除的位置就好了；
  3. 有两个孩子，就需要选一个合适的孩子节点作为新的根节点，该节点称为 继承节点。
- 红黑树的删除红色结点不会破坏平衡，删除黑结点会造成删除子树黑结点减一，将兄弟结点的黑色弄成红色，或者将兄弟结点转过来一个黑色
- 平衡树是为了解决二叉查找树退化为链表的情况，而红黑树是为了解决平衡树在插入、删除等操作需要频繁调整的情况
- 1.根节点是黑色2.不能有连续连个红结点3.任一结点的黑高相同。保证了没有任何一条路径会比其他路径长出两倍

### `ConcurrentModificationException`
- 当遍历数组的同时删除其中元素就会发生这个异常 是fast-fail机制 
- 调用next()时会检查 modCount和expectModCount是否一致，不一致则抛
- 但单线程下如何解决这个问题：使用iterator.remove，他会同步modCount和expectModeCount

### BlockingQueue
- 实现都是基于 ReentrantLock
- 是一个具有特殊入队和出队操作的队列
  1. 入队时，如果队列满，则等待队列有空位，阻塞生产者，避免任务生产速度过快
  2. 出队时，如果队列为空，则等待队列直到非空，阻塞消费者
- BlockingQueue 实现主要用于生产者-使用者队列 
- 线程池为什么要用阻塞队列来组织任务，因为任务生产速度和任务消费速度不匹配

## ThreadPoolExecutor
- 构造线程池对象参数如下：
1. 核心线程数：线程池中一直存活的线程数量，当新的任务请求到来时,如果线程池中线程数<核心线程数,则会新建线程,默认情况下核心线程会一直存活,只有当allowCoreThreadTimeOut设置为true时且发生超时才会被销毁
2. 最大线程数,线程池中线程的上限
3. keepAlive：非核心线程允许空闲的最大时长，超过空闲时间则会被销毁(当池中线程数>=核心线程数时创建出来的线程都是非核心线程)
- ThreadPoolExecutor线程池管理策略
if 线程池中正在运行的线程数 < corePoolSize
  {新建线程来处理任务(即使线程池中有线程处于空闲状态)}
else if 线程池中正在运行的线程数 >= corePoolSize
  {
  if 缓冲队列未满
    任务被放入缓冲队列
  else 缓冲队列满
    if maximumPoolSize > 线程池中正在运行的线程数 > corePoolSize
      新建线程来处理任务 此时的任务会被立即执行
    else if 线程池中正在运行的线程数 = maximunPoolSize
      通过handler所指定的策略来处理此任务
        拒绝策略(丢弃策略)
          ThreadPoolExecutor.AbortPolicy 悄悄地丢弃一个任务
          ThreadPoolExecutor.DiscardOldestPolicy 丢弃最旧的任务，重新提交最新的
          ThreadPoolExecutor.CallerRunsPolicy 在调用者的线程中执行被拒绝的任务
          ThreadPoolExecutor.DiscardPolicy() 丢弃当前任务
  }

### 拒绝策略
1.  AbortPolicy 会抛出RejectedExecutionException 异常
2.  DiscardPolicy 悄悄地丢弃最新的任务（rejectedExecution 空实现）
3.  DiscardOldestPolicy，丢弃最老的任务