---
title: Glide源码分析（四）
date: 2017/7/8  
---

## Engine
　　Engine类的load()方法有一大段英文注释总结了load方法对请求的处理帮助我们理解，大概翻译后我们再根据注释去阅读代码：
	
- 检查缓存，如果缓存存在就使用缓存
- 检查当前有没有存在正在使用的活跃资源优先，如果有就使用活跃资源
- 查当前是否有存在的load任务，如果有的话将回调ResourceCallback添加进去

　　对于活跃资源是指那些被请求不止一次而且还没有被释放的资源。一旦资源被释放，资源会被存储到缓存中。如果资源被从缓存中返回给一个新的调用者使用，资源将会被重新成为活跃资源。如果资源被从缓存中清除的话，就不能回收和重复利用这个资源了。

　　啰里啰嗦，一大段我们看看代码是不是如上所说呢？

	  public <R> LoadStatus load(
      GlideContext glideContext,Object model,Key signature,
      	int width,int height,Class<?> resourceClass,Class<R> transcodeClass,
      		Priority priority,DiskCacheStrategy diskCacheStrategy,Map<Class<?>, Transformation<?>> transformations,
      			boolean isTransformationRequired,Options options,boolean isMemoryCacheable,
      				boolean useUnlimitedSourceExecutorPool,boolean onlyRetrieveFromCache,ResourceCallback cb) {
   		 //判断是否在主线程
    	Util.assertMainThread();
    	long startTime = LogTime.getLogTime();
    	//生成缓存资源的key值，便于查询存储缓存资源
    	EngineKey key = keyFactory.buildKey(model, signature, width, height, transformations,
        resourceClass, transcodeClass, options);
    	//获取缓存资源
    	EngineResource<?> cached = loadFromCache(key, isMemoryCacheable);
    	//缓存资源不为空回调获取资源
    	if (cached != null) {
      		cb.onResourceReady(cached, DataSource.MEMORY_CACHE);
      	if (Log.isLoggable(TAG, Log.VERBOSE)) {
        	logWithTimeAndKey("Loaded resource from cache", startTime, key);
      	}
      	return null;
    	}
    	//获取当前活跃的资源
    	EngineResource<?> active = loadFromActiveResources(key, isMemoryCacheable);
    	if (active != null) {
      	cb.onResourceReady(active, DataSource.MEMORY_CACHE);
      	if (Log.isLoggable(TAG, Log.VERBOSE)) {
        	logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      	}
      	return null;
    	}
    	//上面都没有，获取正在运行的load任务
    	EngineJob<?> current = jobs.get(key);
    	if (current != null) {
      		current.addCallback(cb);
      	if (Log.isLoggable(TAG, Log.VERBOSE)) {
        	logWithTimeAndKey("Added to existing load", startTime, key);
      	}
     	 return new LoadStatus(cb, current);
    	}
    	//啥都没有，那我们只能自己创建任务了
		//EngineJob是一个调度DecodeJob的任务，添加，移除资源回调，并notify回调的类
    	EngineJob<R> engineJob = engineJobFactory.build(key, isMemoryCacheable,useUnlimitedSourceExecutorPool);
		//DecodeJob是处理来自缓存或者原始的资源，应用转换动画以及转码。
    	DecodeJob<R> decodeJob = decodeJobFactory.build(
        glideContext,
        model,
        key,
        signature,
        width,
        height,
        resourceClass,
        transcodeClass,
        priority,
        diskCacheStrategy,
        transformations,
        isTransformationRequired,
        onlyRetrieveFromCache,
        options,
        engineJob);
		//存储创建的任务
    	jobs.put(key, engineJob);
		//添加回调
    	engineJob.addCallback(cb);
		//线程池执行decodeJob的run()方法
    	engineJob.start(decodeJob);

    	if (Log.isLoggable(TAG, Log.VERBOSE)) {
      	logWithTimeAndKey("Started new load", startTime, key);
    	}
    	return new LoadStatus(cb, engineJob);
  		}
　　注释诚不欺我，果然就是按照那个步骤来处理。我们要看下engineJob.start(decodeJob)方法。

	  public void start(DecodeJob<R> decodeJob) {
    		this.decodeJob = decodeJob;
    		GlideExecutor executor = decodeJob.willDecodeFromCache()
        		? diskCacheExecutor
        			: getActiveSourceExecutor();
    		executor.execute(decodeJob);
  	  }
### Glide的线程池
　　这里看到了线程池GlideExecutor，所以插播一下对于Glide对线程池的使用，它们是在GlideBuilder的build()方法里实现的，我们前文讲过build()方法里初始化了Glide对象和参数。
	
	public Glide build(Context context) {
		//sourceExecutor用于缓存未命中Glide的加载、解码和转换任务，
    	if (sourceExecutor == null) {
      		sourceExecutor = GlideExecutor.newSourceExecutor();
    	}
		//diskCacheExecutor用于缓存命中时的加载、解码和转换任务
    	if (diskCacheExecutor == null) {
      		diskCacheExecutor = GlideExecutor.newDiskCacheExecutor();
    	}
			...初始化Glide等...
	}

	//创建缓存未命中时候的线程池，参数calculateBestThreadCount()返回线程池大小
	public static GlideExecutor newSourceExecutor() {
    	return newSourceExecutor(calculateBestThreadCount(), DEFAULT_SOURCE_EXECUTOR_NAME,UncaughtThrowableStrategy.DEFAULT);
  	}

	//创建缓存命中时候的线程池，线程池大小默认为1，参数DEFAULT_DISK_CACHE_EXECUTOR_THREADS=1
	public static GlideExecutor newDiskCacheExecutor() {
    	return newDiskCacheExecutor(DEFAULT_DISK_CACHE_EXECUTOR_THREADS,DEFAULT_DISK_CACHE_EXECUTOR_NAME, UncaughtThrowableStrategy.DEFAULT);
  	}
	//获取线程池大小  根据CPU的数量和Java虚拟机中可用的处理器数量来选择合适的线程数
	public static int calculateBestThreadCount() {
    	ThreadPolicy originalPolicy = StrictMode.allowThreadDiskReads();
		//获取CPU的信息
    	File[] cpus = null;
    	try {
      		File cpuInfo = new File(CPU_LOCATION);
      		final Pattern cpuNamePattern = Pattern.compile(CPU_NAME_REGEX);
      		cpus = cpuInfo.listFiles(new FilenameFilter() {
        	@Override
        	public boolean accept(File file, String s) {
          		return cpuNamePattern.matcher(s).matches();
        		}
      		});
    	} catch (Throwable t) {
      	if (Log.isLoggable(TAG, Log.ERROR)) {
        	Log.e(TAG, "Failed to calculate accurate cpu count", t);
      	}
    	} finally {
      		StrictMode.setThreadPolicy(originalPolicy);
    	}
		//获取cpu数量
    	int cpuCount = cpus != null ? cpus.length : 0;
		//获取java虚拟机可用的cpu数量
    	int availableProcessors = Math.max(1, Runtime.getRuntime().availableProcessors());
		//比较大小，最大不能超过4   MAXIMUM_AUTOMATIC_THREAD_COUNT=4
    	return Math.min(MAXIMUM_AUTOMATIC_THREAD_COUNT, Math.max(availableProcessors, cpuCount));
  		}
　　好了，Glide的线程池就说到这里，我们说回executor.execute(decodeJob)。
### DecodeJob的run()方法
　　DecodeJob的run()方法很简单，仅仅是对取消任务和任务失败进行了判断。主要是调用runWrapped()方法。

	  @Override
  		public void run() {
      try {
		//如果取消了任务注销全部的初始化的参数
      	if (isCancelled) {
        	notifyFailed();
        	return;
      	}
		//
      	runWrapped();
      } catch (RuntimeException e) {
        ...如果抛异常了也采取取消任务的操作...
      }
    }　　
　　runWrapped()方法代码见下方，主要是根据RunReason的不同而不同。RunReason为一个枚举，表示再次执行的原因。分别为INITIALIZE，指第一次调度任务；SWITCH_TO_SOURCE_SERVICE,指缓存加载失败而重新获取数据；DECODE_DATA，指获取了缓存成功但是操作和回调不在同一个线程上。
  
	private void runWrapped() {
     	switch (runReason) {
      		case INITIALIZE:
				//获取下一步执行的策略
        		stage = getNextStage(Stage.INITIALIZE);
				//根据策略的不同，获得Generator
        		currentGenerator = getNextGenerator();
				//load数据
        		runGenerators();
        		break;
      		case SWITCH_TO_SOURCE_SERVICE:
				//load数据
        		runGenerators();
        		break;
      		case DECODE_DATA:
				//处理已经load到的数据
        		decodeFromRetrievedData();
        		break;
      		default:
        		throw new IllegalStateException("Unrecognized run reason: " + runReason);
    		}
  		}
　　以上代码我们看到，当RunReason不同，调用的方法也不同。我们第一次请求网络自然是走的是RunReason.INITIALIZE情况。我们分析一下策略，它也是一个枚举，根据不同的策略从哪里读取数据。

	  private enum Stage {
    	/**第一次执行的策略*/
    	INITIALIZE,
    	/** 解码从缓存资源 */
    	RESOURCE_CACHE,
    	/** 从源数据缓存的解码 */
    	DATA_CACHE,
    	/** 从检索源解码*/
    	SOURCE,
    	/** 编码转换资源加载成功后。*/
    	ENCODE,
    	/**没有更多的可行的阶段。 */
    	FINISHED,
  		}
　　主要加载的策略为三种：RESOURCE_CACHE，DATA_CACHE，SOURCE。runWrapped()方法中，根据不同的策略，getNextGenerator()获得不同的DataFetcherGenerator。RESOURCE_CACHE对应着ResourceCacheGenerator，DATA_CACHE对应着DataCacheGenerator，SOURCE对应着SourceGenerator。当获得不同的DataFetcherGenerator，调用runGenerators()load数据。对于RunReason==DECODE_DATA的情况，decodeFromRetrievedData()处理已经load的数据。我们先看runGenerators()。
		
 	private void runGenerators() {
    	currentThread = Thread.currentThread();
    	startFetchTime = LogTime.getLogTime();
    	boolean isStarted = false;
		//
    	while (!isCancelled && currentGenerator != null
			//关键方法，调用各自的不同currentGenerator.startNext()
        	&& !(isStarted = currentGenerator.startNext())) {
			//如果没有命中资源，则换策略继续执行
      		stage = getNextStage(stage);
      		currentGenerator = getNextGenerator();
			/换策略不行重新来过
      		if (stage == Stage.SOURCE) {
        		reschedule();
        		return;
      		}
    	}
    	// 重新来过还是不行，朋友放弃吧
    	if ((stage == Stage.FINISHED || isCancelled) && !isStarted) {
      		notifyFailed();
    	}
  	}	
　　接着三大CurrentGenerator的startNext()调用了网络请求获取了图片资源，我们下文再看
## 总结
　　网上有一个流程图总结的很好，就拿过来作为总结吧
![](http://orvoldspw.bkt.clouddn.com/glide_load_flow.jpg)
  