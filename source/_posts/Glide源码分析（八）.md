---
title: Glide源码分析（八）
date: 2017/7/20 
---

## 前提
　　上文说到缓存的读取和写入，Glide采用Lru算法自然还有磁盘缓存。今天在撸磁盘缓存的时候，我发现自己在前面撸图片加载的流程的时候还是忽视了一些东西。如此看来，源码不仅仅是读一遍就可以的，就像小时候读一些经典名著要反复的读，书读百遍其义自见。

## 磁盘缓存的写入流程
　　我们还记得，在Glide第一次加载网络图片资源时使用的是SourceGenerator，从网络加载成功后会调用DataCacheGenerator加载回调最终加载到ImageView等控件上。在图片从网络获取后SourceGenerator的回调方法onDataReady()中
	
	public void onDataReady(Object data) {
    	DiskCacheStrategy diskCacheStrategy = helper.getDiskCacheStrategy();
    	if (data != null && diskCacheStrategy.isDataCacheable(loadData.fetcher.getDataSource())) {
      	dataToCache = data;
      	// We might be being called back on someone else's thread. Before doing anything, we should
      	// reschedule to get back onto Glide's thread.
      	cb.reschedule();
    	} else {
      	cb.onDataFetcherReady(loadData.sourceKey, data, loadData.fetcher,
          	loadData.fetcher.getDataSource(), originalKey);
    	}
 	 }
　　我在前文忽视了dataToCache = data这一句代码。正是因为这句设置，当再次回调SourceGenerator的startNext()方法的时候，才会因为dataToCache不为null，从而去执行SourceGenerator的cacheData()方法。

	public boolean startNext() {
    	if (dataToCache != null) {
      		Object data = dataToCache;
      		dataToCache = null;
      		cacheData(data);
    	}
    		....
  	}	
	
	private void cacheData(Object dataToCache) {
    	long startTime = LogTime.getLogTime();
    	try {
      		Encoder<Object> encoder = helper.getSourceEncoder(dataToCache);
			//注释1
      		DataCacheWriter<Object> writer =
          		new DataCacheWriter<>(encoder, dataToCache, helper.getOptions());
      		originalKey = new DataCacheKey(loadData.sourceKey, helper.getSignature());
      		helper.getDiskCache().put(originalKey, writer);
      		if (Log.isLoggable(TAG, Log.VERBOSE)) {
       		...
      		}
    	} finally {
      		loadData.fetcher.cleanup();
    	}

    	sourceCacheGenerator =
        	new DataCacheGenerator(Collections.singletonList(loadData.sourceKey), helper, this);
  	}
　　我们要注意注释1开始后的代码。这里先创建了一个负责写入磁盘的DataCacheWriter对象。然后根生成DataCacheKey键对象，一般而言如果没有过多的设置键对象就是图片的url和宽高。helper.getDiskCache().put(originalKey, writer)就是写入磁盘缓存的关键。我们等会会专门对这一点进行详细的了解。

## 	磁盘缓存的读取流程
　　磁盘缓存的读取，自然就在DataCacheGenerator的startNext（）方法中，和SourceGenerator的代码几乎对应。
		
	public boolean startNext() {
    	while (modelLoaders == null || !hasNextModelLoader()) {
      		sourceIdIndex++;
      		if (sourceIdIndex >= cacheKeys.size()) {
        		return false;
     		}
      		Key sourceId = cacheKeys.get(sourceIdIndex);
      		Key originalKey = new DataCacheKey(sourceId, helper.getSignature());
      		cacheFile = helper.getDiskCache().get(originalKey);
      		．．．
    	}

## 磁盘缓存
　　Glide的磁盘缓存主要实现是DiskLruCache。先看一下，Glide是如何使用到DiskLruCache。

	helper.getDiskCache(）

	//DecodeHelper类中的
	DiskCache getDiskCache() {
    	return diskCacheProvider.getDiskCache();
  	}
　　实际实现为Engine的内部类LazyDiskCacheProvider的getDiskCache()方法

	public DiskCache getDiskCache() {
      if (diskCache == null) {
        synchronized (this) {
          if (diskCache == null) {
			//注释2
            diskCache = factory.build();
          }
          if (diskCache == null) {
            diskCache = new DiskCacheAdapter();
          }
        }
      }
      return diskCache;
     }
    }
　　注释2利用工厂模式，创建了diskCache对象。整个方法又利用单例模式保证了diskCache对象的唯一性。DiskCache.Factory有两个实现类，InternalCacheDiskCacheFactory和ExternalCacheDiskCacheFactory，区别在于当图片使用外部存储的时候采用ExternalCacheDiskCacheFactory。最终执行了DiskLruCacheWrapper的get()方法获得DiskCache对象也就是DiskLruCacheWrapper对象。这里面最终还是依靠DiskLruCache来实现。

