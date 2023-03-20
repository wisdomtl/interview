# 手写一个图片库
1. 获取资源：异步并发下载图片，最大化利用cpu资源
2. 资源解码：按实际需求异步解码，多线程并发是否能加快解码速度
3. 资源变换：使用资源池，复用变换的资源，避免内存抖动
4. 缓存：磁盘缓存原始图片，或变换的资源。内存缓存刚使用过的资源，使用lru策略控制大小
5. 感知生命周期：避免内存泄漏
6. 感知内存吃紧：清理缓存

# 数据加载流程
- RequestBuilder 构建 Request和 Target，将请求委托给RequestManager，RequestManager触发Request.begin(),然后调用Engine.load()加载资源，若有内存缓存则返回，否则启动异步任务加载磁盘缓存，若无则从网络加载
- DecodeJob 负责加载数据（可能从磁盘，或网络，onDataFetcherReady），再进行数据解码（onDataFetcherReady），再进行数据变换（Transformation），写ActiveResource，（将变换后的数据回调给Target），将变换后的资源写文件（ResourceEncoder）

# 预加载
preload，加载到一个PreloadTarget，等资源加载好了，就调用clear，将资源从ActiveResource移除存到Lrucache中
## 感知内存吃紧
- 注册ComponentCallbacks2，实现细粒度内存管理onLowMemory(){清除内存},onTrimMemory(){修剪内存}
```
    memoryCache.trimMemory(level);
    bitmapPool.trimMemory(level);
    arrayPool.trimMemory(level);
```
可以设置在onTrimMemory时，取消所有正在进行的请求。

## BitmapPool
- BitmatPool 则是 Glide 维护了一个图片复用池，LruBitmapPool 使用 Lru 算法保留最近使用的尺寸的 Bitmap
- api19 后使用bitmap的字节数和config作为key，而之前使用宽高和congif，所以19以后复用度更高
- 用类似LinkedHashMap存储，键值对中的值是一组Bitmap，相同字节数的Bitmap 存在一个List中（这样设计的目的是，将Lru策略运用在Bitmap大小上，而不是当个Bitmap上），控制BitmapPool大小通过删除数据组中最后一个Bitmap。
- 大部分用于Bitmap变换和gif加载时

## ArrayPool
- 是一个采用Lru策略的数组池，用于解码时候的字节数组的复用。
- 清理内存意味着清理MemoryCache，BitmapPool,ArrayPool

# 缓存
默认情况下，Glide 会在开始一个新的图片请求之前检查以下多级的缓存：
活动资源 (Active Resources) - 现在是否有另一个 View 正在展示这张图片？
内存缓存 (Memory cache) - 该图片是否最近被加载过并仍存在于内存中？
资源类型（Resource） - 该图片是否之前曾被解码、转换并写入过磁盘缓存？
数据来源 (Data) - 构建这个图片的资源是否之前曾被写入过文件缓存？

在 Glide v4 里，所有缓存键都包含至少两个元素
活动资源，内存缓存，资源磁盘缓存的缓存键还包含一些其他数据，包括：
必选：Model
可选：签名
宽度和高度
可选的变换（Transformation）
额外添加的任何 选项(Options)
请求的数据类型 (Bitmap, GIF, 或其他)
## 磁盘缓存策略
- 如果缓存策略是AUTOMATIC（默认），对于网络图片只缓存原始数据，加载本地资源是存储变换过的数据，如果加载不同尺寸的图片，则会获取原始缓存并在此基础上做变换。
- 如果缓存策略是ALL，会缓存原始图片以及每个尺寸的副本，
- 如果缓存策略是SOURCE,只会缓存变换过的资源，如果另一个界面换一个尺寸显示图片，则会重新拉取网络
可通过自定义Key实现操控缓存命中策略（混入自己的值，比如修改时间）

## 内存
- 内存缓存分为两级
	1. 活跃图片 ActiveResource
		- 使用HashMap存储正在使用资源的弱引用
		- 资源被包装成带引用计数的EngineResource，标记引用资源的次数（当引用数不为0时阻止被回收或降级，降级即是存储到LruCache中）
		- 这一级缓存没有大小限制，所以使用了资源的弱引用
		- 存：每当下载资源后会在onEngineJobComplete()中存入ActiveResource，或者LruCache命中后，将资源从中LruCache移除并存入ActiveResource。
		- 取：每当资源释放时，会降级到LruCache中（请求对应的context onDestroy了或者被gc了）
		- 开一个后台线程，监听ReferenceQueue，不停地从中获取被gc的资源，将其从ActiveResource中移除，并重新构建一个新资源将其降级为LruCache
		- ActiveResource是为了缓解LruCache中缓存造成压力，因为LruCache中没有命中的缓存只有等到容量超限时才会被清除，强引用即使内存吃紧也不会被gc，现在当LruCache命中后移到ActiveResource，弱引用持有，当内存吃紧时能被回收。
	2. LruCache
		- 使用 LinkedHashMap 存储从活跃图片降级的资源，使用Lru算法淘汰最近最少使用的
		- 存：从活跃图片降级的资源（退出当前界面，或者ActiveResource资源被回收）
		- 取：网络请求资源之前，从缓存中取，若命中则直接从LruCache中移除了。
- 内存缓存只会缓存经过转换后的图片
- 内存缓存键根据10多个参数生成，url，宽高

## 磁盘
- 会将源数据或经过变换的数据存储在磁盘，在内存中用LinkedHashMap记录一组Entry，Entry内部包含一组文件，文件名即是key，并且有开启后台线程执行删除文件操作以控制磁盘缓存大小。
- 写磁盘缓存即是触发Writer将数据写入磁盘，并在内存构建对应的File缓存在LinkedHashMap中
- 根据缓存策略的不同，可能存储源数据和经过变换的数据。

# 感知生命周期
- 构造RequestManager时传入context，可以是app的，activity的，或者是view的
- 向界面添加无界面Fragment（SupportRequestManagerFragment），Fragment把生命周期传递给Lifecycle，Fragment持有RequestManager，RequestManager监听Lifecycle，RequestManager向RequestTracker传递生命周期以暂停加载，RequestTracker遍历所有正在进行的请求，并暂停他们（移除回调resourceReady回调）
- 当绑定context destroy时，RequestManager会将该事件传递给RequestTracker，然后触发该请求Resource的clear，再调用Engine.release，将resource降级到LruCache
- 通过HashMap结构保存无界面Fragment以避免重复创建

# 取消请求
通过移除回调，设置取消标志位实现：无法取消已经发出的请求，会在DecodeJob的异步任务的run()方法中判断，如果cancel，则返回。移除各种回调，会传递到DataFetcher，httpUrlConnection 读取数据后会判断是否cancel，如果是则返回null。并没有断开链接

# 感知网络变化
- 通过 ConnectivityManager 监听网络变化，当网络恢复时，遍历请求列表，将没有完成的任务继续开始

# 线程切换
有一个和主线程绑定的Handler post 到主线程

# Transformation
- 所有的BitmapTransformation 都是从BitmapPool 拿到一个bitmap，然后将在原有bitmap基础上应用一个matrix再画到新bitmap上。
- 变换也是一个key，用以在缓存的时候做区别

# RecycleView图片错乱
- 异步任务+视图复用
- 解决方案：设置占位图+回收表项时取消图片加载（或者新得加载开始时取消旧的加载）+imageview加tag判断是否是自己的图片如果不是则先调用clear
# Glide 缓存失效
- 是因为 Key 发生变化，Url是生成key的依据，Url可能发生变化比如把token追加在后面
- 自定义生成key的方式，继承GlideUrl重写getCacheKey()
# 减少等待时间
- 使用thumbnail并行地加载小资源（只是缩小尺寸的图），大图下载好会替换

# 自定义加载
定义一个Model类用于包装需要加载的数据
定义一个Key的实现类，用于实现第一步的Model中的数据的签名用于区分缓存
定义一个DataFetcher的实现类，用于告诉Glide音频封面如何加载，并把加载结果回调出去
定义一个ModelLoader的实现类用于包装DataFetcher
定义一个ModelLoaderFactory的实现类用于生成ModelLoader实例
将自定义加载配置到AppGlideModule中

# 线程池
1. 磁盘缓存线程池，一个核心线程：用于io图片编码
2. 加载资源线程池，最多不超过4个核心线程数，用于处理网络请求，图片解码转码
3. 动画线程池，最多不超过2个线程
4. 磁盘缓存清理线程池
5. ActiveResource 开启一个后台线程监听ReferenceQueue 
所有线程池都默认采用优先级队列

# 请求优先级
通过给加载线程池配置优先级队列，加载任务DecodeJob 实现了 compareTo 方法，将priority相减

# 优化
1. 服务器存多种尺寸的图片
2. 自定义 AppGlideModule，按设备性能好坏设定MemoryCategory.HIGH,LOW,NORMAL，内存缓存和bitmapPool的大小系数，以及图片解码格式，ARGB_8888，RGB_565
2. RecyclerView 在onViewRecycled 中调用clear ，因为recyclerView会默认会缓存5个同类表项，如果类型很多，内存中会持有表项，如果这些表项都包含图片，Glide 的ActiveResource会膨胀。导致gc
3. 如果 RecyclerView 包含一个很长的itemView，超过一屏，其中包含很多照片，最好把长itemView拆成多个itemView
5. 使用thumbnail，加载一个缩略图，最好是一个独立的链接，如果是本地的也不差
6. 使用preload，将资源提前加载到内存中。
6. 大部分情况下 RESOURCE ，即缓存经过变换的图片上是最好选择，节约内存和磁盘。对于gif资源只缓存原始资源DATA，因为gif是多张图每次编码解码反而耗时
7. 使用Glide实现变换，因为有BitmapPool供复用

# 总结
1. 会根据控件大小进行下采样，以解码出符合需求的大小，对内存更友好
2. 内存缓存+磁盘缓存
3. 感知生命周期，取消任务，防止内存泄漏
4. 感知内存吃紧，进行回收
5. BitmapPool，防止内存抖动的进行bitmap变换
6. 定义请求优先级

# webView 优化
1. 使用 Glide 加载 webView 中的图片资源，借用Glide的缓存，重写shouldInterceptRequest(), 用Glide下载并解析成bitmap，把Bitmap包装成ByteArrayInputStream传递给WebResourceResponse并返回


# 如何加载Gif
读取流的前三个字节，若判断是gif，则会命中gif解码器-将资源解码成GifDrawable，它持有GifFrameLoader会将资源解码成一张张Bitmap并且传递给DelayTarget的对象，该对象每次资源加载完毕都会通过handler发送延迟消息回调 onFrameReady() 以触发GifDrawable.invalidataSelf()重绘。加载下一帧时会重新构建DelayTarget























DecodeJob 解码资源，
DiskLruCacheWrapper 是DiskCache的内部实现
DiskLruCacheWrapper.get -- DiskLruCache.get() -- Value -- Value.getFile() -- File
DiskLruCache 用LinkedHashMap缓存Entry，Entry里面存了一组File，Editor 是文件的操纵类
BitmapEncoder 将Bitmap输出到磁盘
BitmapTransformation 是一个预定义的使用BitmapPool进行变换、
所有的变换被存在一个map结构中，键是资源类型
ModelLoader模型加载器，将一个Model转换成可以被解码的资源，并且考虑控件的大小。
LoadPath 遍历 DecodePath ，第一个成功decode的资源被返回
从磁盘缓存（文件）加载的是一个Object对象，需要经过解码（ResourceDecoder），变换（Transformation），转码（ResourceTranscoder）
Engine -- 
	EngineJob -- 
		DecodeJob -- 生成generator -- 
			Generator.startNext() -- 获取文件缓存 -- 根据文件寻找对应ModelLoader(FileLoader) -- 
				LoadData.loadData -- DecodePath -- ResourceDecoder.decode -- (generator.onDataReady) -- 
			Generator -- (FetcherReadyCallback.onDataFetcherReady) -- 
		DecodeJob --(onResourceReady)-- 
	EngineJob --(ResourceCallback.onResourceReady)-- SingleRequest
Resource是一个可以被回收的资源