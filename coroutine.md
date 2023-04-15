

## Channel
是一个挂起队列，和java 中 blocking queue 类似

生产者叫 SendChannel，消费者叫 ReceiveChannel 生产者和消费者之间有一条缓冲通道

构造函数中可以传入三个参数
1. 约会RENDEZVOUS：生产者和消费者一对一的关系，若对方没有出现则会阻塞自己。
2. 合并类型CONFLATED：生产者不会被挂起，但会覆盖旧值，接收者接收到的最新的值
3. 缓冲类型BUFFERED：缓冲区满时（缓冲区用数组实现，表示有限缓冲），生产者被挂起，缓冲区没有数据时消费者被挂起
4. 无限型UNLIMITED：缓冲区无限大的BUFFERED型（用链表实现）
执行多线程同时生产，多线程同时消费

被挂起的生产者或消费者会存放在链式队列中等待恢复。

# 协程
- 是建立在线程之上的，更轻量级的（在用户态完成非抢占的并发调度），更容易控制生命周期（结构化并发）的计算单元。
- 借助于suspend方法实现非抢占式的并发调度，协程可以主动让出执行权
- 用同步的方式实现异步效果，普通的线程映射为内核线程，内核线程的调度是抢占cpu时间片，而协程的调度在用户态就能完成，协作式的调度程序可以主动让出执行权（挂起）
- 比使用线程池更容易取消异步操作，享受结构化并发，异常处理
- suspend + resume
- 挂起方法并不会挂起线程，因为就像调用一个带回调的方法一样，它挂起的是协程剩下的代码。

## CPS
- Continuation Passing Style，传递剩余的计算，将剩余的计算作为一个回调传递给方法。

## suspend
- suspend 方法将剩下的计算作为continuation参数，continuation在方法内部又会被包装成另一个continuation，里面有个 invokeSuspend方法调用了挂起方法，它持有label。
- 状态机: 每个suspend 方法都构建一个continuation不经济，一个协程块中的suspend方法会共用一个 continuation。将原先不同的continuation写在了不同的 switch case 分支内，以挂起点为分割点。每执行一个分支点，label 就+1,表示进入下一个分支。挂起方法会被多次调用，因为label值不同每次都会走不同的分支（invokeSuspend）
- suspend 的返回值标志着挂起方法有没有被挂起

## Dispatcher
调度器CoroutineDispatcher是一个ContinuationInterceptor。通过interceptContinuation()将continuation包装成DispatchedContinuation

不同的调度器通过重写 dispatch方法实现不同的线程调度。有些是通过handler抛一个runnable，有些是向线程池抛一个

default 属于cpu运算密集型：线程被阻塞时，cpu是在忙着运算

io 属于io型：线程被阻塞时，cpu是闲着。

## job
可以被取消的任务

他有六种状态：new-active-completing-completed-cancelling-canceled

只有延迟启动的协程的job才会处于new 状态，其他都处于active状态，completing意味着自己的活干完了，在等子协程。 cancelling 是在取消协程之前最后的清理资源的机会。

新建的协程会继承父亲的CoroutineContext，除了其中的job，新协程会新建job并成为父job的子job

## 结构化并发
线程间的并发是没有级联关系的，所以是非结构的
1. 结束一个线程时，怎么同时结束这个线程中创建的子线程？
2. 当某个子线程在执行时需要结束兄弟线程要做怎么做？
3. 如何等待所有子线程都执行完了再结束父线程？
这些问题都可以通过共享标记位、CountDownLatch 等方式实现。但这两个例子让我们意识到，线程间没有级联关系；所有线程执行的上下文都是整个进程，多个线程的并发是相对整个进程的，而不是相对某一个父线程。


## 异常处理
在 coroutineScope中，异常是向上传播的，只要任意一个子协程发生异常，它会把这个异常逐级向上传播，直到根协程，所以在子协程中使用ExceptionHandler是无效的，整个scope都会执行失败，并且其余的所有子协程都会被取消掉；
在 supervisorScope中，异常是向下传播的，一个子协程的异常不会影响整个 scope的执行，也不会影响其余子协程的执行；（重写了childCancelled并返回false）
CancellationException 异常总是被忽略


## 取消协程
每个启动的协程都会返回一个job，调用 job.cancel()会让job处于canceling状态，然后在下一个挂起点抛出CancellationException ,如果协程中没有挂起点，则协程不能被取消。因为每个suspend 方法都会检查job是否活跃，若不活跃则抛出CancellationException ，这个异常只是给上层一次关闭资源的机会，可以通过try-catch 捕获。

特别的，suspendCoroutine 创建的suspend 方法不会检测job是否活跃，所以他无法被cancel

另外如果把suspend 方法 try-catch 了，它抛出的CancellationException也会被捕获，导致协程无法被取消，解决方案是在catch中重新抛出CancellationException

对于没有挂起的协程，需要通过while(isActive)来检查job是否被取消 或者 yield()

当协程抛出CancellationException 后，在启动协程将会被忽略
```
suspend fun main(): Unit = coroutineScope {
   val job = Job()
   launch(job) {
       try {
           delay(2000)
           println("Job is done")
       } finally {
           println("Finally")
           launch { // will be ignored
               println("Will not be printed")
           }
           delay(1000) // here exception is thrown
           println("Will not be printed")
       }
   }
   delay(1000)
   job.cancelAndJoin()
   println("Cancel done")
}
// (1 sec)
// Finally
// Cancel done
```
解决方案是 NonCancellable 
```
suspend fun main(): Unit = coroutineScope {
   val job = Job()
   launch(job) {
       try {
           delay(200)
           println("Coroutine finished")
       } finally {
           println("Finally")
           withContext(NonCancellable) {
               delay(1000L)
               println("Cleanup done")
           }
       }
   }
   delay(100)
   job.cancelAndJoin()
   println("Done")
}
// Finally
// Cleanup done
// Done
```