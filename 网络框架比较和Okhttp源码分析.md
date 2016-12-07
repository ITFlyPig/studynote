### Android网络框架比较和Okhttp源码分析
> 本篇文章是自己学习OkHttp的时候记录下的笔记，目的在于把知识点以自己的方式来学通和记录下来备忘，参考了很多网上的文章，很感谢他们。随着自己的理解，知识可能会变的，逐渐添加。

<small>
**1. HttpClient：**    
   建议废弃,谷歌官网从安卓6.0系统开始默认不再支持httpClient,基于httpClient的框架建议不再使用(Android-async-http框架 基于 httpClient，建议废弃).   
 **2. HttpUrlConnection**   
 建议用框架,从Android4.4开始HttpURLConnection的底层实现采用的是okHttp.    
 **3. Volley**  
 android2.2及以下版本默认使用HttpClient，而android2.3及以上版本默认使用HttpUrlConnection。 官方已经认可okHttp框架，不再更新volley框架，建议废弃
> volley框架的特点：
>  1. 扩展性强 -基于接口设计。2. 提供简单的图片加载工具。 3.非常适合去进行数据量不大，但通信频繁的网络操作   
> 缺点：对于大数据量的网络操作，比如说下载和上传文件等，Volley的表现就会非常糟糕。   
  
**4. Retrofit**   
Retrofit 将请求地址转换为接口，通过注解来指定请求方法，请求参数，请求头，返回值等信息。如果你的应用程序中集成了OKHttp，Retrofit默认会使用OKHttp处理其他网络层请求   
**5. okHttp**  
支持文件上传下载，非常高效，支持SPDY、连接池、GZIP和 HTTP 缓存。默认情况下，OKHttp会自动处理常见的网络问题，像二次连接、SSL的握手问题。从Android4.4开始HttpURLConnection的底层实现采用的是okHttp   
**总结：**因为Volley不再更新，其他的都支持Okhttp，所以主要分析Okhttp得源码
****
### Okhttp的源码分析
#### Okhttp特点
> 1.支持HTTP2/SPDY黑科技  
2.socket自动选择最好路线，并支持自动重连   
3.拥有自动维护的socket连接池，减少握手次数(共享同一个Socket来处理同一个服务器的所有请求)   
4.拥有队列线程池，轻松写并发   
5.拥有Interceptors轻松处理请求与响应（比如透明GZIP压缩,LOGGING）    
6.基于Headers的缓存策略(缓存响应数据来减少重复的网络请求)

#### Okhttp的拦截器
应用拦截器和网络拦截器的区别？    
addInterceptor:设置应用拦截器，主要用于设置公共参数，头信息，日志拦截等（主要是处理request和response）    
addNetworkInterceptor：设置网络拦截器，主要用于重试或重写   
一张图说明：
![
](http://upload-images.jianshu.io/upload_images/1458573-76f6b7798a26ec17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

例如：初始请求http://baidu.com 会重定向1次到https://baidu.com 。
则应用拦截器会执行1次，返回的是https的响应
网络拦截器会执行2次，第一次返回http的响应，第二次返回https的响应
> **应用拦截器：**  
  1. 不需要担心中间过程的响应,如重定向和重试.  
 2. 总是只调用一次,即使HTTP响应是从缓存中获取.   
 3. 观察应用程序的初衷. 不关心OkHttp注入的头信息如: If-None-Match.    
 4. 允许短路而不调用 Chain.proceed(),即中止调用.    
 5. 允许重试,使 Chain.proceed()调用多次. 
 ****  
**网络拦截器**   
1. 能够操作中间过程的响应,如重定向和重试.   
2. 当网络短路而返回缓存响应时不被调用.   
3. 只观察在网络上传输的数据.    
4. 携带请求来访问连接.




#### Okhttp的拦截机制
![
](http://upload-images.jianshu.io/upload_images/1458573-d7375161cda31f32.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**RetryAndFollowUpInterceptor做的事情：**    
1. 创建StreamAllocation对象，为后面流程的执行准备条件。   
2.处理重定向的HTTP响应。    
3.错误恢复。   
**BridgeInterceptor:**  
这个Interceptor做的事情比较简单。可以分为发送请求和收到响应两个阶段来看。在发送请求阶段，BridgeInterceptor补全一些http header，这主要包括Content-Type、Content-Length、Transfer-Encoding、Host、Connection、Accept-Encoding、User-Agent，还加载Cookie，随后创建新的Request，并交给后续的Interceptor处理，以获取响应。   
而在从后续的Interceptor获取响应之后，会首先保存Cookie。如果服务器返回的响应的content是以gzip压缩过的，则会先进行解压缩，移除响应中的header Content-Encoding和Content-Length，构造新的响应并返回；否则直接返回响应。
**CacheInterceptor:**    
在请求发送阶段，主要是尝试从cache中获取响应，获取成功的话，且响应可用未过期，则响应会被直接返回；否则通过后续的Interceptor来从网络获取，获取到响应之后，若需要缓存的，则缓存起来。   
**ConnectInterceptor:**   
这个类的定义看上去倒是蛮简洁的。ConnectInterceptor的主要职责是建立与服务器之间的连接，但这个事情它主要是委托给StreamAllocation来完成的。如我们前面看到的，StreamAllocation对象是在RetryAndFollowUpInterceptor中分配的。   
ConnectInterceptor通过StreamAllocation创建了HttpStream对象和RealConnection对象，随后便调用了realChain.proceed()，向连接中写入HTTP请求，并从服务器读回响应。     

**CallServerInterceptor:**                            CallServerInterceptor首先将http请求头部发给服务器，如果http请求有body的话，会再将body发送给服务器，继而通过httpStream.finishRequest()结束http请求的发送。

随后便是从连接中读取服务器返回的http响应，并构造Response。

如果请求的header或服务器响应的header中，Connection值为close，CallServerInterceptor还会关闭连接。

最后便是返回Response。
#### OKhttp的请求流程
![
](http://upload-images.jianshu.io/upload_images/1458573-79aa905da482f0e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图表示的是Okhttp的请求流程
****
![
](http://upload-images.jianshu.io/upload_images/1458573-b48f7b4793267b2d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图表示的是请求的时序图
****
![
](http://upload-images.jianshu.io/upload_images/1458573-fed46f02396b5bfe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)   
上图表示的OKhttp的总体设计

**一次网络请求执行的流程：**   
1.创建request   
2.通过newCall函数将request转换为Call  
3.通过Dispatcher将请求call添加到队列(client.dispatcher().enqueue(...new AsyncCall(..)))，通过线程池执行队列中的任务。  
4.执行的时候在run方法中调用AsyncCall的execute()方法  
5.在execute（）方法中调用getResponseWithInterceptorChain()，在其中调用了ApplicationInterceptorChain.proceed(),如果有其他的应用拦截器的话，就会遍历拦截器集合，执行每一个拦截的intercept()方法，最终还是会执行getResponse方法()。
5.在getResponse中调用HttpEngine.sendRequest()方法发送请求。（getResponse还有失败重试机制）  
6.在HttpEngine.sendRequest()中，先从 Cache 中判断当前请求是否可以从缓存中返回，没有Cache则调用connect()方法连接网络。  
7.connect()方法中又调用streamAllocation.newStream(),在其中又使用
```java
RealConnection resultConnection = findHealthyConnection(...)
```
搜索可用的socket连接，在findHealthyConnection方法中使用findConnection来实际工作，findConnection的作用：查找连接池ConnectionPool中是否有可用的连接，有则返回连接，没有则新建连接。
> 这里获得可用的Socket对象之后，会根据Connection创建HttpStream	
![
](http://img.blog.csdn.net/20161117224123754)

8.发起请求是在Connection.connect()这里，实际执行是在HttpConnection.flush()这里进行一个刷入。这里重点应该关注一下sink和source，他们创建的默认方式都是依托于同一个socket：
this.source = Okio.buffer(Okio.source(socket));
this.sink = Okio.buffer(Okio.sink(socket));
如果再进一步看一下io的源码就能看到：
Source source = source((InputStream)socket.getInputStream(), (Timeout)timeout);  
Sink sink = sink((OutputStream)socket.getOutputStream(), (Timeout)timeout);
9.得到连接后通过HttpStream就可以写入请求和读取返回body，是在HttpEngine.readResponse中读取的。readNetworkResponse方法的代码如下：
> ![
](http://img.blog.csdn.net/20161118115127484)

#### OkHttp重要的知识点
##### OkHttp之网络连接
尽管程序只提供了URL，但是OkHttp在连接web服务器时会使用三种类型：URL（每个URLs表明一个特定的地址和查询，每个web服务器持有许多个URL。）, 地址Addresses（地址指定了一个web服务器，所有的静态配置需要连接到服务器：端口号），路线Route。
当你通过OkHttp请求一个URL时，会发生下面的事情：  
   1.通过URL和配置的OkHttpClient创建一个地址。这个地址指定如何连接到web服务器。  
   2.试图根据这个从连接池中选择一个可用的连接(Socket链接)。  
   3.如果没有在连接池中找到连接，则选择一条线路连接。这就意味着创建一个DNS请求，获取服务器的IP地址。 然后根据需要，选择一个TLS版本，和代理服务器。   
   4.如果这是一条新的线路，则通过创建一个直接的socket连接，一个TLS隧道（对于HTTPS而言是一个HTTP代理），或者创建一个直接的TLS连接。然后进行TLS握手。   
   5.发送请求，获取响应。如果连接出问题，OkHttp会选择另外一条路线重试。这就允许OkHttp在服务器地址的子集不可达时进行恢复。 在连接池中的连接陈旧或者尝试的TLS版本不支持时，这一功能也是非常有用的。  
   HttpConnection是对底层处理Connection的封装,一个HttpConnection主要有如下生命周期：
> 1、发送请求头；  
2、打开一个sink(io中有固定长度的或者块结构chunked方式的)去写入请求body；  
3、写入并且关闭sink；  
4、读取Response头部；  
5、打开一个source（对应到第2步的sink方式）去读取Response的body；  
6、读取并关闭source；  

#### 复用连接池
##### 出现连接池的原因
创建socket需要进行3次握手，而释放socket需要2次握手(或者是4次)。重复的连接与释放tcp连接就像每次仅仅挤1mm的牙膏就合上牙膏盖子接着再打开接着挤一样。而每次连接大概是TTL一次的时间(也就是ping一次)，在TLS环境下消耗的时间就更多了。很明显，当访问复杂网络时，延时（而不是带宽）将成为非常重要的因素。  
当然，上面的问题早已经解决了，在http中有一种叫做keepalive connections的机制，它可以在传输数据后仍然保持连接，当客户端需要再次获取数据时，直接使用刚刚空闲下来的连接而不需要再次握手
> Okhttp支持5个并发KeepAlive，默认链路生命为5分钟(链路空闲后，保持存活的时间)

我们知道在socket连接中，也就是Connection中，本质是封装好的流操作，除非手动close掉连接，基本不会被GC掉，非常容易引发内存泄露。  
**OkHttp中的连接保持和回收**  
**保持：**   
在OkHttp中使用deque（Deque也就是双端队列，双端队列同时具有队列和栈性质）来保存连接，也就是连接池的数据结构。通过put（放进连接池）和get（从连接池中获取）来操作连接。   
**如何找到不活跃的连接：**  
遍历RealConnection连接中的StreamAllocationList，它维护着一个弱应用列表
查看此StreamAllocation是否为空(它是在线程池的put/remove手动控制的)，如果为空，说明已经没有代码引用这个对象了，需要在List中删除
遍历结束，如果List中维护的StreamAllocation删空了，就返回0，表示这个连接已经没有代码引用了，是泄漏的连接;否则返回非0的值，表示这个仍然被引用，是活跃的连接。  
当然，找到不活跃的连接之后，我们清除的时候就可以把它清除了。就像Java中GC的引用计数。  
参考 <a>http://www.jianshu.com/p/92a61357164b</a>

#### OkHttp的缓存
![
](http://upload-images.jianshu.io/upload_images/98641-4bd320d4e34af60a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
上图是浏览器的缓存机制
在OkHttp中：  
1. Okhttp的缓存是自动完成的，完全由服务器Header决定的，自己没有必要进行控制。网上热传的文章在Interceptor中手工添加缓存代码控制，它固然有用，但是属于Hack式的利用，违反了RFC文档标准，不建议使用，OkHttp的官方缓存控制在注释中。如果读者的需求是对象持久化，建议用文件储存或者数据库即可（比如realm）。  
2. 服务器的配置非常重要，如果你需要减小请求次数，建议直接找对接人员对max-age等头文件进行优化；服务器的时钟需要严格NTP同步  

参考连接<a>http://www.jianshu.com/p/9cebbbd0eeab</a>

参考：
<a>http://www.jianshu.com/p/db197279f053</a>  
<a>http://blog.csdn.net/zxm317122667/article/details/53212384</a>  
<a>http://www.jianshu.com/p/230e2e2988e0</a>  
<a>http://frodoking.github.io/2015/06/29/android-okhttp-connectionpool-http1-x-http2-x/</a>
<a>http://www.jianshu.com/p/aad5aacd79bf</a>  
**再次感谢**






