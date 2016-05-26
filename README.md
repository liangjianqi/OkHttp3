# OkHttp3源码解析 #



# 特点 #
- 支持[SPDY][link_spdy]，共享同一个Socket来处理同一个服务器的所有请求
- 如果SPDY不可用，则通过连接池来减少请求延时
- 拥有Interceptors轻松处理请求与响应，无缝的支持GZIP来减少数据流量
- 缓存响应数据来减少重复的网络请求

[link_spdy]:http://www.geekpark.net/topics/158198  

# 概念 #
- SPDY
旨在通过压缩、多路复用和优先级来缩短网页的加载时间和提高安全性主要的类

- 多路复用
SPDY允许多个并发HTTP请求共用一个TCP会话，通过复用在单个TCP连接上的多次请求，而非为每个请求单独开放连接，这样只需建立一个TCP连接就可以传送网页上所有资源，不仅可以减少消息交互往返的时间还可以避免创建新连接造成的延迟，使得 TCP 的效率更高。

# 总体设计 #
## 总体设计图 ##
![](http://i.imgur.com/3AQhOcL.png)

主要是通过Diapatcher不断从RequestQueue中取出请求（Call），根据是否已缓存调用Cache或Network这两类数据获取接口之一，从内存缓存或是服务器取得请求的数据。该引擎有同步和异步请求，同步请求通过Call.execute()直接返回当前的Response，而异步请求会把当前的请求Call.enqueue添加（AsyncCall）到请求队列中，并通过回调（Callback）的方式来获取最后结果。
## 重要的类 ##

1. Route
对地址的一个封装类

2. Platform
这个类主要是做平台适应性，针对Android2.3到5.0后的网络请求的适配支持。同时，在这个类中能看到针对不同平台，通过Platform的Class.forName()反射获得当前Runtime使用的socket库

3. Connnection
对JDK中的socket进行了引用计数封装，用来控制socket连接

4. ConnnectionPool
管理减少网络延迟的HTTP和SPDY连接复用

5. Call
请求执行后的返回，可以被取消，但不能被执行两次

6. Dispatcher
异步请求时的策略,用于控制并发的请求

7. Cache
缓存响应文件
8. OkHttpClient  
配置和创建HTTP连接的



# 请求流程图 #
![](http://i.imgur.com/jjN5oJZ.png)

# 请求详解 #
一个简单的异步请求  

        OkHttpClient client = new OkHttpClient.Builder().build();
        Request request = new Request.Builder()
                .url("https://raw.githubusercontent.com/square/okhttp/master/README.md")
                .get()
                .build();

        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                Log.e(TAG,e.getMessage());
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                if(response.isSuccessful()){
                    Log.e(TAG,response.body().string());
                }
            }
        });

当HttpClient的请求入队时，根据代码，实际上是Dispatcher进行了入队操作  

	synchronized void enqueue(AsyncCall call) {
	    if (runningAsyncCalls.size() < maxRequests && runningCallsForHost(call) < maxRequestsPerHost) {
		  //添加正在运行的请求
	      runningAsyncCalls.add(call);
		  //线程池执行请求
	      executorService().execute(call);
	    } else {
		  //添加到缓存队列
	      readyAsyncCalls.add(call);
	    }
 	 }  

当runningRequests<64 && runningRequestsPerHost<5时，直接把AsyncCall添加到runningCalls队列中，并在线程池中执行，反之就放入readyAsyncCalls进行缓存等待  

AsyncCall本质是实现了Runnable接口，内部实现了execute方法

	@Override protected void execute() {
      boolean signalledCallback = false;
      try {
		//执行耗时的IO任务
        Response response = getResponseWithInterceptorChain(forWebSocket);
        if (canceled) {
          signalledCallback = true;
		  //回调
          responseCallback.onFailure(RealCall.this, new IOException("Canceled"));
        } else {
          signalledCallback = true;
		  //回调
          responseCallback.onResponse(RealCall.this, response);
        }
      } catch (IOException e) {
        if (signalledCallback) {
          // Do not signal the callback twice!
          logger.log(Level.INFO, "Callback failure for " + toLoggableString(), e);
        } else {
          responseCallback.onFailure(RealCall.this, e);
        }
      } finally {
        client.dispatcher().finished(this);
      }
    }  

注意finally中的代码，当任务执行完成后，总是会调用Dispatcher的finished方法  

    private void promoteCalls() {
		//如果请求数为最大值是，等待
	    if (runningAsyncCalls.size() >= maxRequests) return; // Already running max capacity.
		//如果缓存队列是空的，等待
	    if (readyAsyncCalls.isEmpty()) return; // No ready calls to promote.
	
	    for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
	      AsyncCall call = i.next();
	      if (runningCallsForHost(call) < maxRequestsPerHost) {
			//将缓存队列中的最后一个任务移动到运行队列中，并开始执行
	        i.remove();
	        runningAsyncCalls.add(call);
	        executorService().execute(call);
	      }
	
	      if (runningAsyncCalls.size() >= maxRequests) return; // Reached max capacity.
	    }
 	 }  

这样就将缓存队列向前走了一步  



# 详细设计 #
## 类关系图 ##


## 核心功能介绍 ##

# 拓展 #



























Http请求头  

![](http://i.imgur.com/6jj2cWs.png)
