---
title: Glide源码分析（六）
date: 2017/7/17 
---

　　调用DataCacheGenerator的startNext()后执行了FileLoader的loadData()方法
		  
	@Override
    public void loadData(Priority priority, DataCallback<? super Data> callback) {
      try {
		//从缓存中获取资源
        data = opener.open(file);
      } catch (FileNotFoundException e) {
        if (Log.isLoggable(TAG, Log.DEBUG)) {
          Log.d(TAG, "Failed to open file", e);
        }
		//失败回调
        callback.onLoadFailed(e);
        return;
      }
	  //成功回调
      callback.onDataReady(data);
    }
　　加载资源成功回调DataCacheGenerator的onDataReady()方法，继续回调DecodeJob类的onDataFetcherReady()方法

	  @Override
  	public void onDataFetcherReady(Key sourceKey, Object data, DataFetcher<?> fetcher,
      DataSource dataSource, Key attemptedKey) {
    	this.currentSourceKey = sourceKey;
    	this.currentData = data;
    	this.currentFetcher = fetcher;
    	this.currentDataSource = dataSource;
    	this.currentAttemptingKey = attemptedKey;
		//如果不是当前的线程重新来过
    	if (Thread.currentThread() != currentThread) {
      		runReason = RunReason.DECODE_DATA;
      		callback.reschedule(this);
    	} else {
		//对加载的数据进行处理
      		decodeFromRetrievedData();
    	}
  	}
　　decodeFromRetrievedData()方法内部，调用了decodeFromData()方法获得Resource。当Resource不为空的时候，会调用notifyEncodeAndRelease()方法。这个方法负责缓存完成后通过回调告诉外面加载完成。所以这个方法里面，最重要的是notifyComplete()方法，通过callback回调EngineJob的onResourceReady()方法。然后通过发送Handler消息到主线程。

	  @Synthetic
  	void handleResultOnMainThread() {
    	...

    	for (ResourceCallback cb : cbs) {
      		if (!isInIgnoredCallbacks(cb)) {
        		engineResource.acquire();
        		cb.onResourceReady(engineResource, dataSource);
      		}
    	}
    // 到这里加载结束，释放所有资源
	    ...
  	}
　　cb.onResourceReady(engineResource, dataSource)调用的是SingleRequest的onResourceReady()方法。

	public void onResourceReady(Resource<?> resource, DataSource dataSource) {
    	...判断resource是否为空...

    	Object received = resource.get();
    	...判断received是否为空...

    	onResourceReady((Resource<R>) resource, (R) received, dataSource);
 	 }

	private void onResourceReady(Resource<R> resource, R result, DataSource dataSource) {
    	// We must call isFirstReadyResource before setting status.
   	 	boolean isFirstResource = isFirstReadyResource();
    	status = Status.COMPLETE;
    	this.resource = resource;
    	...
    	if (requestListener == null
        	|| !requestListener.onResourceReady(result, model, target, dataSource, isFirstResource)) {
      		Transition<? super R> animation =
          		animationFactory.build(dataSource, isFirstResource);
			//这个target我们在说load方法里说过，此处就是ImageViewTarget
      		target.onResourceReady(result, animation);
    	}

    	notifyLoadSuccess();
  	}
　　我们在看ImageViewTarget的onResourceReady()方法。

		@Override
  	public void onResourceReady(Z resource, @Nullable Transition<? super Z> transition) {
    	if (transition == null || !transition.transition(resource, this)) {
      		setResourceInternal(resource);
    	} else {
      		maybeUpdateAnimatable(resource);
    	}
  	}

	private void setResourceInternal(@Nullable Z resource) {
    	maybeUpdateAnimatable(resource);
    	setResource(resource);
  	}
　　setResource为一个抽象方法，实现该方法的是DrawableImageViewTarget，它的setResource()方法很简单。至此，图像成功加载。

	  @Override
  	protected void setResource(@Nullable Drawable resource) {
    	view.setImageDrawable(resource);
  	}