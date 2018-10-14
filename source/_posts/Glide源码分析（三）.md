---
title: Glide源码分析（三）
date: 2017/7/5 
---

	Glide
    	.with(this)
    	.load(url)
    	.placeholder(R.drawable.loading_spinner)
    	.into(myImageView);
　　上文我们跟随代码走过了Glide.with(this)方法得到了RequestManager对象，并了解了RequestManager对象具备智能管理请求的作用。那么本文，就开始走进如何请求图片。

## load()
　　Load()方法代码只有一句话：

      public RequestBuilder<Drawable> load(@Nullable Object model) {
    	   return asDrawable().load(model);
  	   }
　　asDrawable()方法获得了RequestBuilder对象，并将其泛型设置为Drawable。我们接着看RequestBuilder.load(model)方法

      public RequestBuilder<TranscodeType> load(@Nullable Object model) {
    	   return loadGeneric(model);
  	   }

  	  private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
    	   this.model = model;
           isModelSet = true;
           return this;
       }
　　可以看出，load()方法更多的是对RequestBuilder对象参数的设置，这里我们先只说参数为url的情况。这个Object model，就是我们传进的url。
## into()

      public Target<TranscodeType> into(ImageView view) {
    		Util.assertMainThread();
    		Preconditions.checkNotNull(view);
    		...
    		return into(context.buildImageViewTarget(view, transcodeClass));
  			}

	　  public <Y extends Target<TranscodeType>> Y into(@NonNull Y target) {
		    //检测是否为UI线程
		    Util.assertMainThread();
    		//判断target是否为空
    		Preconditions.checkNotNull(target);
    		//判断上文提到的model是否设置
    		if (!isModelSet) {
      			throw new IllegalArgumentException("You must call #load() before calling #into()");
    		}
    		//获取存在还未完成的Request
    		Request previous = target.getRequest();
    		//如果存在，清空
    		if (previous != null) {
      			requestManager.clear(target);
    		}

    		requestOptions.lock();
    		//创建新的请求
    		Request request = buildRequest(target);
    		target.setRequest(request);
    		requestManager.track(target, request);

    		return target;
  			}
　　如果此时没有Request，会调用buildRequest()创建新的Request。

### 创建Request
      private Request buildRequest(Target<TranscodeType> target) {
    		return buildRequestRecursive(target, null, transitionOptions, requestOptions.getPriority(),requestOptions.getOverrideWidth(),requestOptions.getOverrideHeight());
  	  }

  	  private Request buildRequestRecursive(Target<TranscodeType> target,
      		@Nullable ThumbnailRequestCoordinator parentCoordinator,TransitionOptions<?, ? super TranscodeType> transitionOptions,Priority priority, int overrideWidth, int overrideHeight) {
    		
      		....

      		return obtainRequest(target, requestOptions, parentCoordinator, transitionOptions,priority,overrideWidth, overrideHeight);
        }
  	 }
　　这两段代码涉及到很多对Request的设置，Glide有一个专门的RequestOptions类。我们这里目前先只研究最简单的的第一次请求情况。所以直奔obtainRequest()

### SingleRequest
　　obtainRequest()只有一句代码就是调用了SingleRequest的静态obtain()方法。通过init()初始化了SingleRequest。所以我们知道了在第一次加载网络图片的时候，RequestBuilder的into()方法创建的Request正是SingleRequest。我们回到RequestBuilder的into()方法。

	requestManager.track(target, request);

	void track(Target<?> target, Request request) {
    	targetTracker.track(target);
    	requestTracker.runRequest(request);
  	}

　　targetTracker的作用是持有当前所有存活的Target，并触发Target相应的生命周期方法，方便开发者在整个请求过程的不同状态中进行回调，做相应的处理。targetTracker.track(target)方法就是将当前的target保存到集合中。
	
　　requestTracker类是RequestManager管理请求的工具，直接操作正是requestTracker。requestTracker.runRequest(request)启动了SimleRequest,调用了SingleRequest的begin方法。

	  @Override
  	public void begin() {
    	stateVerifier.throwIfRecycled();
    	startTime = LogTime.getLogTime();
    	if (model == null) {
      		if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
        		width = overrideWidth;
        		height = overrideHeight;
      		}
      
      	int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
      	onLoadFailed(new GlideException("Received null model"), logLevel);
      	return;
    	}

    	status = Status.WAITING_FOR_SIZE;//设置等待图片size的宽高状态
    	//必须要确定图片的宽高，确定了则调用onSizeReady
    	if (Util.isValidDimensions(overrideWidth, overrideHeight)) {

      		onSizeReady(overrideWidth, overrideHeight);
    	} else {
      	//设置回调，监听界面的绘制，当检测到宽高有效时，回调onSizeReady方法
      		target.getSize(this);
    	}

    	if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
        && canNotifyStatusChanged()) {
			//在加载完成前加载默认图
      		target.onLoadStarted(getPlaceholderDrawable());
    	}
    	if (Log.isLoggable(TAG, Log.VERBOSE)) {
      		logV("finished run method in " + LogTime.getElapsedMillis(startTime));
    	}
  	}

　　这里简单说下加载完成前加载默认图，在源码里辗转反侧之后，调用的其实是下面这个很明显的方法设置图片。这就是Glide.placeholder()传入的占位图

	  public void onLoadFailed(@Nullable Drawable errorDrawable) {
    		super.onLoadFailed(errorDrawable);
    		setResourceInternal(null);
    		setDrawable(errorDrawable);
  		}

	  public void setDrawable(Drawable drawable) {
    		view.setImageDrawable(drawable);
  	    }
　　我们接着看onSizeReady()方法。

	 @Override
  	public void onSizeReady(int width, int height) {
    	stateVerifier.throwIfRecycled();
    	...
    	if (status != Status.WAITING_FOR_SIZE) {
      		return;
    	}
    	//将状态改为运行状态
    	status = Status.RUNNING;
	   ...
    
	loadStatus = engine.load(
        glideContext,
        model,
        requestOptions.getSignature(),
        this.width,
        this.height,
        requestOptions.getResourceClass(),
        transcodeClass,
        priority,
        requestOptions.getDiskCacheStrategy(),
        requestOptions.getTransformations(),
        requestOptions.isTransformationRequired(),
        requestOptions.getOptions(),
        requestOptions.isMemoryCacheable(),
        requestOptions.getUseUnlimitedSourceGeneratorsPool(),
        requestOptions.getOnlyRetrieveFromCache(),
        this);
    	...
  	}
　　我们可以看到最后调用的是Engine.load()方法。我们前文说过Engine类是一个任务创建，发起，回调，管理存活和缓存的资源的重要类。如果没有忘记上文，Engine类正是在Glide的build()方法里创建出来。我们下文将专门介绍这一点。
	


	
