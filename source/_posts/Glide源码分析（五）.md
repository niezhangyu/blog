---
title: Glide源码分析（五）
date: 2017/7/11 
---
　　上文说到三大CurrentGenerator的startNext()调用了网络请求获取了图片资源，分别为ResourceCacheGenerator未经处理的资源数据缓存文件，DataCacheGenerator经过处理的资源数据缓存文件(采样转换等处理)和SourceGenerator原始数据的生成器，包含了根据来源创建的ModelLoader和Model(文件路径，URL...)。这三个类差距不大，都是用来处理数据。我们一直假设的是第一次加载图片资源，所以会执行SourceGenerator的startNext()方法。

	  @Override
  	public boolean startNext() {
    // 判断是否有缓存
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      //如果有缓存就会创建DataCacheGenerator去执行
      cacheData(data);
    }
    //如果上面已经创建了DataCacheGenerator不是空，就改为DataCacheGenerator执行
    if (sourceCacheGenerator != null && sourceCacheGenerator.startNext()) {
      return true;
    }
    sourceCacheGenerator = null;

    loadData = null;
    boolean started = false;
    //是否有更多的ModelLoader,从loaders中找到一个能够加载该文件类型的loader
    //我们这里说的第一次网络加载，对于网络加载的ModelLoader有HttpUriLoader,HttpGlideUrlLoader
    //还支持第三方Okhttp,volley的OkHttpUrlLoader，VolleyUrlLoader
    while (!started && hasNextModelLoader()) {
      loadData = helper.getLoadData().get(loadDataListIndex++);
      if (loadData != null
          && (helper.getDiskCacheStrategy().isDataCacheable(loadData.fetcher.getDataSource())
          || helper.hasLoadPath(loadData.fetcher.getDataClass()))) {
        started = true;
        // 开始进行数据加载,
        loadData.fetcher.loadData(helper.getPriority(), this);
       }
     }
     return started;
   	}
　　下面就是我们熟悉的网络请求，如果请求成功会回调。假设这里我们使用的是最直接的HttpUriLoader，如果请求网络成功后会通过callback.onDataReady(result)回调SourceGenerator的onDataReady()方法。这里继续通过cb.reschedule()回调DecodeJob的reschedule()方法，继续通过cb.reschedule()回调EngineJob的EngineJob()方法

	  @Override
  	public void reschedule(DecodeJob<?> job) {
    	if (isCancelled) {
      		MAIN_THREAD_HANDLER.obtainMessage(MSG_CANCELLED, this).sendToTarget();
    	} else {
		//继续执行SourceGenerator.startNext()方法
      		getActiveSourceExecutor().execute(job);
    	}
  	}
　　这里继续下去还是会执行SourceGenerator.startNext()方法，还记得本文刚开始的代码刚刚进入startNext()方法会判断是否有缓存
，本次就会执行这一步。因为上一次执行的时候是下载资源，因此再次执行的时候内存缓存已经存在，因此直接读取缓存数据cacheData(data)。

	public boolean startNext() {
    // 判断是否有缓存
    if (dataToCache != null) {
      Object data = dataToCache;
      dataToCache = null;
      //如果有缓存就会调用DataCacheGenerator去执行
      cacheData(data);
	  ...
    }

	private void cacheData(Object dataToCache) {
    long startTime = LogTime.getLogTime();
    try {
	  //根据不同的数据资源获得不同的Encoder
      Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
      DataCacheWriter<Object> writer =
          new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
      originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
      helper.getDiskCache().put(originalKey, writer);
      if (Log.isLoggable(TAG, Log.VERBOSE)) {
        Log.v(TAG, "Finished encoding source to cache"
            + ", key: " + originalKey
            + ", data: " + dataToCache
            + ", encoder: " + encoder
            + ", duration: " + LogTime.getElapsedMillis(startTime));
      }
    } finally {
      loadData.fetcher.cleanup();
    }
	//创建针对缓存的DataCacheGenerator
    sourceCacheGenerator =
        new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
  }
	
 