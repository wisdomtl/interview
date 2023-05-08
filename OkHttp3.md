# 调度器 Dispatcher
- 在 OkHttpClient.build 中构建
-  维护三个队列和一个线程池来并发处理网络请求，分别是同步运行队列，正在运行的异步队列，等待请求异步队列
-  持有 ExecutorService ，核心线程数为0，表示不保留空闲线程。最大线程数为 Int.max，表示随时会新建线程，使用同步队列，使得请求的生产不会被阻塞

# 五大拦截器
## 0. 应用拦截器一定会被执行一次
## 1. RetryAndFollowUpInterceptor
重试重定向拦截器
这个拦截器是一个while(true)的循环，只有请求成功或者重试超过最大次数，没有路由供重试时才会退出

请求抛出异常并满足重试条件时才重试，收到3xx，需要重定向时会重新构建请求

## 2. BridgeInterceptor
- 将http request加工，添加header 头字段（Connection:keep-alive,Accept-Encoding：Gzip），再将http response 加工 去掉 header

## 3. CacheInterceptor
- 缓存拦截器
- 从DiskLruCache根据请求url获取缓存Response，然后根据一些http头约定的缓存策略决定是使用缓存响应还是发起新的请求。
- 只缓存 get 请求
- 缓存是空间换时间的方法，缓存需要页面置换算法（LRU,FIFO）
- 缓存减小服务器压力，客户端更快地显示数据，无网下显示数据
- 缓存分为强制缓存和对比缓存
    1. 客户端直接拿数据，若缓存未命中则请求网络并更新数据
    2. 客户端拿数据标识，总是查询网络判断数据是否有效
- 1. 响应头中包含Cache-Control：max-age，表示缓存过期时间
- 2. 响应头中有Last-Modified字段标识资源最后被修改时间，客户端下次发起请求时在请求头中会带上If-Modified-Since，服务器比对如果最后修改时间大于该值则返回200，否则304标识缓存有效。
- 3. 除了用最后修改时间做判断，还可以用资源唯一标识来判断ETag/If-None-Match，响应头包含ETag，再次请求时带上If-None-Match，服务器比对标识是否相同，相同则304，否则200
- 缓存策略：先判断缓存是否过期，若未过期则直接使用，若过期则发起请求，请求头带唯一标识，服务器回200或304，如果没有唯一标识则请求头带上次修改时间，服务器200或304
- 在无网环境下即是缓存过期，依然使用缓存，要添加 应用拦截器，重构request修改cache-control字段为FORCE_CACHE:
```
- public class ForceCacheInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request.Builder builder = chain.request().newBuilder();
        if (!NetworkUtils.internetAvailable()) {
            builder.cacheControl(CacheControl.FORCE_CACHE);
        }
        
        return chain.proceed(builder.build());
    }
}
okHttpClient.addInterceptor(new ForceCacheInterceptor());
```
- 若服务器不支持header头缓存字段，则可以添加网络拦截器，在CacheInterceptor收到响应之前修改response的header
```
public class CacheInterceptor implements Interceptor {
    @Override
    public Response intercept(Chain chain) throws IOException {
        Request request = chain.request();
        Response response = chain.proceed(request);
        Response response1 = response.newBuilder()
                .removeHeader("Pragma")
                .removeHeader("Cache-Control")
                //cache for 30 days
                .header("Cache-Control", "max-age=" + 3600 * 24 * 30)
                .build();
        return response1;
    }
}
```
通过retrofit @HEADER
### DiskLruCache
- 使用Okio做输入输出
- 使用LinkedHashMap存储实体，实现LRU缓存置换算法
- 每个缓存保存clean和 dirty 两个副本，实现读写分离
- 使用journal文件做日志记录，便于恢复

## 4. ConnectInterceptor
- 连接拦截器
- 建立连接及连接上的流
- 维护连接池，以复用连接
- 一个物理链接上有多个流（逻辑上的请求响应对），一个物理链接上的多个流是并发的，但有数量限制，一个流上有多个分配，分配是并发的

### 获取连接
1. 复用已分配的连接（重定向再次请求）
2. 无已分配链接，则从链接池那一个新得链接，通过主机名和端口（并不是池中的链接就能复用，除了host之外的字段要都相等，比如dns，协议，代理）
3. 尝试其他路由再从连接池中获取连接，若找到则进行dns查询
4. 如果缓存池中没有链接，则新建链接（tcp+tls握手，sockect.connect+connectTls），这是耗时的，过程中可能有连接池可能有新的可用连接 所以再次尝试从连接池获取连接，如果成功则释放刚建立的链接，否则把新建连接入池

### dns
通过实现 Dns.lookup()定义自己获取ip地址的算法

#### Tls链接建立
1. 在原先socket基础上生成 SSLSocket，然后tls握手，然后验证证书，通过之后用sslSocket生成source和sink

### 连接复用
- tcp连接建立需要三次握手和四次挥手
- 连接池实现链接缓存，实现同一地址的链接复用
- 连接池以队列方式存储链接ArrayDeque，链接池中同一个地址最多维护5个空闲链接，空闲链接最多存活5分钟

### 连接清理
- 五分钟定时任务，每五分钟遍历所有链接，并找到其中空闲时间最长的，如果空闲时间超过keep-alive（5分钟），或者空闲链接超过了阈值（5个）则清除这个链接 

## 4.x NetworkInterceptor
- 网络拦截器
- 在连接建立完成和发送请求之间
- 可能不被调用，比如缓存命中，或者多次调用重定向

## 5. CallServerInterceptor
- 请求拦截器
- 将请求和响应分装成 http2 的帧，通过Http2ExchangeCodec（内部通过okio实现io）
1 写入请求头
2 写入请求体
3 读取响应头
4 读取响应体
如果响应头中 Connection:close，则在当前链接上设置标志位，表示该链接不能再被复用

# Call
采用工厂方式构建的请求
```kotlin
  fun interface Factory {
    fun newCall(request: Request): Call
  }
```
OkHttpClient实现了这个接口，并构建了 RealCall 对象

## RealCall
### 如何检测重复请求
使用一个 AtomicBoolean 作为请求过的标志位，每次执行 execute之前就会检查
## 如何发起请求
- 请求被封装成 RealCall 对象，异步请求会进一步会封装成一个 Runnable
- 同步请求直接将请求在拦截器责任链上传递（并加到同步请求队列汇总）
- 异步请求会缓存到一个准备请求队列中，并检查当前并发请求数（同一个域最多5个并发，不同域最多64个），若未超阈值，则将请求出队入线程池执行（将请求在责任链上传递）
同一链接上的最大并发数据流是Int.max

## 请求如何在责任链上传递
责任链持有一组拦截器和当前拦截器索引，通过每次复制一条新责任链且索引+1，实现传递
发起请求并获取响应就是在请求和响应在责任链上u型传递的过程
## cookie
http 是无状态的，但可以通过cookie字段进行会话状态管理
两种方式设置cookie1.应用拦截器 2.cookieJar

# 总结
1. 链接池，HTTP/2复用连接
2. 默认支持GZIP，告诉服务器支持gzip压缩格式，请求添加Accept-Encoding: gzip，响应中返回Content-Encoding: gzip（使用哈夫曼算法，重复度越高压缩效果越好）
3. 响应缓存
4. 方便添加拦截器