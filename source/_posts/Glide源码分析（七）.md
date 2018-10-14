---
title: Glide源码分析（七）
date: 2017/7/19 
---
　　前提：啰哩啰嗦了六篇文章才刚刚把Glide的一次加载网络图片的流程说完。但其实，前面我们还是忽视了很多细节，比如Glide加载了网络图片后是如何缓存到内存中的呢，当存在缓存Glide又是如何去加载的呢？本文将会走进Glide的内存缓存操作。由于我们前文是跟着url走过一遍所以我们其实已经读到过读取缓存的代码，只是没有专门撸一遍罢了。

## Engine读取缓存
　　由于我们前文是跟着url走过一遍所以我们读到过读取缓存的代码，因此我们先去寻找如何读取缓存。这里又不得不提到Engine类，我们还记得它是一个任务创建，发起，回调，管理存活和缓存的资源的重要类。在Engine.load()方法里，我们曾经说过我们回头再说，那么现在说。
	  
	public <R> LoadStatus load(...很多参数...) {
	   		...

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
			...
	}

　　对于keyFactory.buildKey()方法生成缓存资源的key值。EngineKey类并不是很麻烦重写了equal()和hashcode()方法，根据参数对其url，还有宽高等等条件生成唯一的key值，保证只有传入EngineKey的所有参数都相同的情况下才认为是同一个EngineKey对象。在这里我没有去验证，一般情况下我们使用Glide加载网络图片较多，既然model为生成key值得参数的一种，那么我们第一次从网络加载的url就构成了key值的一个参数。

　　再看获取缓存资源的loadFromCache()方法

	  private EngineResource<?> loadFromCache(Key key, boolean isMemoryCacheable) {
    	if (!isMemoryCacheable) {
      		return null;
    	}
    	EngineResource<?> cached = getEngineResourceFromCache(key);
    	if (cached != null) {
      		cached.acquire();
      		activeResources.put(key, new ResourceWeakReference(key, cached, getReferenceQueue()));
    	}
    	return cached;
  	}
　　我们可以看到内部loadFromCache()方法主要调用了getEngineResourceFromCache()方法。注意第一句判断if (!isMemoryCacheable)。在Glide的用法中有这么一个方法skipMemoryCache()方法是用来设置是否使用缓存，传递参数true/false，这里的isMemoryCacheable便是通过skipMemoryCache()来设置的，可见如果我们传递false，在这里直接return null禁止使用缓存。对于获取的EngineResource后的代码，正是load()方法中loadFromActiveResources()获取当前活跃资源的方法资源的由来。当我们获取缓存后，存储活跃资源的map便将刚刚从缓存中取出的资源存储为活跃资源，这个map就是一个使用了弱引用hashmap。我们接着看getEngineResourceFromCache()方法。

	private EngineResource<?> getEngineResourceFromCache(Key key) {
    	Resource<?> cached = cache.remove(key);
    	final EngineResource<?> result;
    	if (cached == null) {
      		result = null;
    	} else if (cached instanceof EngineResource) {
      		result = (EngineResource<?>) cached;
    	} else {
      		result = new EngineResource<>(cached, true /*isMemoryCacheable*/);
    	}
    	return result;
  	}
　　在getEngineResourceFromCache()方法中，第一句就是cache.remove(key)。那么这个cache是什么呢？一个MemoryCache接口，它的实现类是LruResourceCache类。我们这里就不说Lru算法了，当得到这个Resource后，在上面代码里通过activeResources将其保存为活跃资源使其不被Lru算法回收。上述的这些LruResourceCache等等，都是在GlideBuilder类的buidl()方法里初始化的，这些我们前文曾经提到过。

## Engine写入缓存

　　上一篇文章里我们说到DecodeJob解析图片网络资源后通过notifyComplete()方法回调EngineJob的onResourceReady方法后向主线程发了一个Handler。在handleResultOnMainThread()方法中，

	  void handleResultOnMainThread() {
    	....
    	engineResource = engineResourceFactory.build(resource, isCacheable);
    	hasResource = true;
    	engineResource.acquire();
    	listener.onEngineJobComplete(key, engineResource);
		
	    for (ResourceCallback cb : cbs) {
     		 if (!isInIgnoredCallbacks(cb)) {
       			 engineResource.acquire();
       			 cb.onResourceReady(engineResource, dataSource);
      		}
    	}
    
    	engineResource.release();

    	release(false /*isRemovedFromQueue*/);
  	}
　　engineResourceFactory.build()方法中，生成了一个图片资源EngineResource。listener.onEngineJobComplete()方法调用的是Engine的onEngineJobComplete()方法。

	  public void onEngineJobComplete(Key key, EngineResource<?> resource) {
   		 Util.assertMainThread();
    	// A null resource indicates that the load failed, usually due to an exception.
    	if (resource != null) {
      		resource.setResourceListener(key, this);
		
      	if (resource.isCacheable()) {
        	activeResources.put(key, new ResourceWeakReference(key, resource, getReferenceQueue()));
      		}
    	}
    	// TODO: should this check that the engine job is still current?
    	jobs.remove(key);
  	  }
　　我们可以看到，当资源被设置为可缓存加载的时候，activeResources为把它存储起来，这个activeResources就是前文提到的弱引用缓存中。

　　那么，我们前文提到的LruResourceCache如何存储的呢？一开始我也找了好久没找到哪里写入，不过后来想到前文说到当从cache中取出缓存后，activeResources会将其存入。那么这个cache的写入也应该和activeResources有关系，而且Lru算法本身对资源的使用有一定关系。所以注意了handleResultOnMainThread()方法中的engineResource.acquire()方法。

	void acquire() {
    	if (isRecycled) {
      		throw new IllegalStateException("Cannot acquire a recycled resource");
    	}
    	if (!Looper.getMainLooper().equals(Looper.myLooper())) {
      		throw new IllegalThreadStateException("Must call acquire on the main thread");
    	}
    	++acquired;
  	}
　　这里主要就是++acquired，acquired为int值。那--acquired呢？在engineResource.release()方法中

	  void release() {
    	if (acquired <= 0) {
      		throw new IllegalStateException("Cannot release a recycled or not yet acquired resource");
    	}
    	if (!Looper.getMainLooper().equals(Looper.myLooper())) {
      		throw new IllegalThreadStateException("Must call release on the main thread");
   		}
    	if (--acquired == 0) {
      		listener.onResourceReleased(key, this);
    	}
  	  }
　　我们可以看到当acquired==0的时候，回到了Engine的onResourceReleased()方法。
	
	  public void onResourceReleased(Key cacheKey, EngineResource resource) {
    	Util.assertMainThread();
    	activeResources.remove(cacheKey);
    	if (resource.isCacheable()) {
      		cache.put(cacheKey, resource);
    	} else {
      		resourceRecycler.recycle(resource);
    	}
  	  }
　　当acquired==0的时候，LruResourceCache将其put写入了缓存。这是什么意思呢？为什么当图片资源从activeResources中移除，然后再将它put到LruResourceCache当中？我们想一下什么是活跃资源，那不就是正在使用中的图片吗？正在使用的图片使用弱引用来进行缓存，不在使用中的图片使用LruCache来进行缓存的功能。那么，这就是Glide对缓存的处理。